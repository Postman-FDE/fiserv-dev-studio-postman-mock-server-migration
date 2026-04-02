# Fiserv Dev Studio — Postman Mock Server Migration: Implementation Guide

> **Purpose:** This document is the complete implementation specification for migrating Fiserv Dev Studio's mock server infrastructure from Prism to Postman Mock Servers. It is written to be self-contained — an engineer (or AI assistant) with access to Fiserv's codebase and this document should have everything needed to implement the migration end-to-end.

> **Last updated:** 2026-04-01

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [MongoDB Schema Design](#2-mongodb-schema-design)
3. [Postman API Reference](#3-postman-api-reference)
4. [Ingestion Pipeline — Scenario Handling](#4-ingestion-pipeline--scenario-handling)
5. [Usage Pipeline — Serving Mock Responses](#5-usage-pipeline--serving-mock-responses)
6. [Backfill Script — One-Time Migration](#6-backfill-script--one-time-migration)
7. [Naming Conventions](#7-naming-conventions)
8. [Notes & Considerations](#8-notes--considerations)

---

## 1. Architecture Overview

The architecture has **two pipelines** that share a common Backend Service:

### 1.1 Ingestion Pipeline (Provisioning Postman Resources)

```
GitHub (OpenAPI Specs)
    │
    │  onChange (webhook)
    ▼
Backend Service ──────► Postman (create/update workspace, spec, collection, mock server)
    │
    │  store Postman resource IDs
    ▼
MongoDB (postman_tenants + postman_resources collections)
```

**Flow:**
1. A GitHub webhook fires when an OpenAPI spec file is added, updated, or deleted.
2. The Backend Service receives the webhook and determines what changed (new tenant, new version, spec update, or deletion).
3. The Backend Service calls the Postman API to create or update the appropriate resources (workspace, spec, collection, mock server).
4. The Backend Service stores the resulting Postman resource IDs in MongoDB so they can be looked up at request time.

### 1.2 Usage Pipeline (Serving Mock Responses)

```
Dev Studio UI
    │
    │  GraphQL mutation (same as today)
    ▼
Backend Service
    │
    │  1. Construct comboKey from request params
    │  2. Lookup postmanMockServerBaseUrl from MongoDB
    │  3. Proxy request to Postman mock server
    │
    ▼
Postman Mock Server ──► mock response ──► Backend Service ──► Dev Studio UI
```

**Flow:**
1. The Dev Studio UI sends the same GraphQL mutation (`RunTenantApiSandboxAtlas`) it sends today, with `productName`, `requestType`, `requestPath`, `apiVersion`, `requestBody`, `requestHeaders`, etc.
2. The Backend Service constructs the `comboKey` the same way it does today: `{productName}_{apiFullPath}_{requestType}_{apiVersion}`.
3. Instead of looking up a raw spec from MongoDB and spinning up an ephemeral Prism server, the Backend Service now:
   - Looks up the `postmanMockServerBaseUrl` from MongoDB using the comboKey.
   - Makes an HTTP request to `{postmanMockServerBaseUrl}{requestPath}` with the appropriate method, headers, and body.
4. Returns the Postman mock server's response to the Dev Studio UI.

**The Dev Studio UI requires zero changes.** The swap happens entirely within the Backend Service.

### 1.3 Postman Resource Hierarchy

For each tenant/product, the Postman resources are organized as follows:

```
Workspace (1 per tenant)
  └── Spec in Spec Hub (1 per version)
       └── Generated Collection (1 per spec, linked/synced to the spec)
            └── Mock Server (1 per collection)
```

**Example for CommerceHub:**
```
Workspace: "CommerceHub"
  ├── Spec: "CommerceHub - v1.25.2000"
  │    └── Collection: "CommerceHub - v1.25.2000"
  │         └── Mock Server: "CommerceHub - v1.25.2000 Mock"
  │              → mockUrl: https://<mock-id>.mock.pstmn.io
  ├── Spec: "CommerceHub - v1.26.0202"
  │    └── Collection: "CommerceHub - v1.26.0202"
  │         └── Mock Server: "CommerceHub - v1.26.0202 Mock"
  │              → mockUrl: https://<mock-id>.mock.pstmn.io
  └── ... (one spec → collection → mock per version)
```

---

## 2. MongoDB Schema Design

The migration introduces **two new MongoDB collections** alongside the existing `tenant_mock_api` collection. The existing collection remains largely unchanged — only one new field is added.

### 2.1 New Collection: `postman_tenants`

**Purpose:** Maps each tenant/product to its Postman workspace. This is a simple lookup table used to determine whether a workspace already exists for a given tenant.

**One document per tenant.**

```typescript
interface PostmanTenant {
  tenantName: string;              // e.g., "CommerceHub" — unique index
  postmanWorkspaceId: string;      // e.g., "1f0df51a-8658-4ee8-a2a1-d2567dfa09a9"
  status: "provisioning" | "ready" | "error";
  createdAt: Date;
  updatedAt: Date;
}
```

**Indexes:**
- Unique index on `tenantName`

**Example document:**
```json
{
  "tenantName": "CommerceHub",
  "postmanWorkspaceId": "1f0df51a-8658-4ee8-a2a1-d2567dfa09a9",
  "status": "ready",
  "createdAt": "2026-04-01T00:00:00.000Z",
  "updatedAt": "2026-04-01T00:00:00.000Z"
}
```

### 2.2 New Collection: `postman_resources`

**Purpose:** Stores the full set of Postman resource IDs for each spec-version combination. This is the primary lookup table used by the usage pipeline to resolve a comboKey to a mock server URL.

**One document per tenant + version combination.**

```typescript
interface PostmanResource {
  tenantName: string;              // e.g., "CommerceHub"
  version: string;                 // e.g., "1.26.0202"
  fileName: string;                // e.g., "reference/1.26.0202/openapi.yaml"
  branchName: string;              // e.g., "main"
  postmanSpecId: string;           // Spec Hub spec ID
  postmanSpecFilePath: string;     // File path within the spec, e.g., "openapi.yaml"
  postmanCollectionId: string;     // Plain collection ID (UUID) — used for GET/DELETE operations
  postmanCollectionUid: string;    // Collection UID (ownerId-collectionId) — used for mock creation and sync
  postmanMockServerId: string;     // Mock server ID
  postmanMockServerUrl: string;    // e.g., "https://<id>.mock.pstmn.io"
  status: "provisioning" | "ready" | "error" | "syncing";
  errorMessage?: string;           // Populated when status is "error"
  createdAt: Date;
  updatedAt: Date;
}
```

**Indexes:**
- Compound unique index on `{ tenantName, version }`
- Index on `tenantName` (for listing all versions of a tenant)

**Example document:**
```json
{
  "tenantName": "CommerceHub",
  "version": "1.26.0202",
  "fileName": "reference/1.26.0202/openapi.yaml",
  "branchName": "main",
  "postmanSpecId": "b4fc1bdc-6587-4f9b-95c9-f768146089b4",
  "postmanSpecFilePath": "openapi.yaml",
  "postmanCollectionId": "12ece9e1-2abf-4edc-8e34-de66e74114d2",
  "postmanCollectionUid": "12345678-12ece9e1-2abf-4edc-8e34-de66e74114d2",
  "postmanMockServerId": "e3d951bf-873f-49ac-a658-b2dcb91d3289",
  "postmanMockServerUrl": "https://e3d951bf-873f-49ac-a658-b2dcb91d3289.mock.pstmn.io",
  "status": "ready",
  "createdAt": "2026-04-01T00:00:00.000Z",
  "updatedAt": "2026-04-01T00:00:00.000Z"
}
```

### 2.3 Existing Collection: `tenant_mock_api` (No Change)

The existing `tenant_mock_api` collection stores one document per comboKey (per endpoint per version). The usage pipeline no longer needs the `spec` field to spin up Prism — it now resolves to a Postman mock server URL via the `postman_resources` collection instead.

Since the comboKey already contains the `tenantName` and `version`, the Backend Service can parse these out and look up `postman_resources` directly. **No schema change is needed to `tenant_mock_api`.**

The Backend Service uses the request parameters (`productName`, `apiVersion`) that are already available in the incoming GraphQL mutation variables to look up the corresponding `postman_resources` record. The existing `tenant_mock_api` collection stays completely untouched.

---

## 3. Postman API Reference

All Postman API endpoints use the base URL `https://api.getpostman.com` and require the `X-API-Key` header for authentication.

```
X-API-Key: <your-postman-api-key>
Content-Type: application/json
```

### 3.1 Create a Workspace

Creates a new workspace for a tenant/product.

```
POST /workspaces
```

**Request Body:**
```json
{
  "workspace": {
    "name": "CommerceHub",
    "type": "private",
    "description": "Postman workspace for CommerceHub mock servers"
  }
}
```

> **Note:** The `type` must be `"private"` to ensure workspace contents are not publicly accessible.

**Response (200 OK):**
```json
{
  "workspace": {
    "id": "1f0df51a-8658-4ee8-a2a1-d2567dfa09a9",
    "name": "CommerceHub"
  }
}
```

**Save:** `workspace.id` → `postman_tenants.postmanWorkspaceId`

**(Optional) After creation,** you can set workspace roles so that non-admin users have read-only access. This prevents accidental modification of specs, collections, and mock servers via the Postman UI:

```
PATCH /workspaces/:workspaceId/roles
```

**Request Body:**
```json
{
  "roles": [
    {
      "op": "add",
      "path": "/user",
      "value": [
        {
          "id": "<userId>",
          "role": "1"
        }
      ]
    }
  ]
}
```

> Role IDs: `"1"` = Viewer (read-only), `"2"` = Editor, `"3"` = Admin. This step can be deferred and configured manually in the Postman UI after provisioning if preferred.

### 3.2 Delete a Workspace

```
DELETE /workspaces/:workspaceId
```

**Response (200 OK):**
```json
{
  "workspace": {
    "id": "1f0df51a-8658-4ee8-a2a1-d2567dfa09a9"
  }
}
```

### 3.3 Create a Spec (Upload OpenAPI Spec to Spec Hub)

Uploads an OpenAPI spec file to the Postman Spec Hub within a workspace.

```
POST /specs?workspaceId=<workspaceId>
```

**Request Body:**
```json
{
  "name": "CommerceHub - v1.26.0202",
  "type": "OPENAPI:3.0",
  "files": [
    {
      "path": "openapi.yaml",
      "content": "<entire YAML file content as a string>"
    }
  ]
}
```

**Accepted `type` values:** `OPENAPI:2.0`, `OPENAPI:3.0`, `OPENAPI:3.1`

> **Important:** The `content` field must contain the full YAML file content as a string. The `path` field is the filename within the spec — use `"openapi.yaml"` consistently. Files cannot exceed 10 MB.

**Response (201 Created):**
```json
{
  "name": "CommerceHub - v1.26.0202",
  "type": "OPENAPI:3.0",
  "id": "b4fc1bdc-6587-4f9b-95c9-f768146089b4",
  "createdAt": "2025-03-15T13:48:28.000Z",
  "createdBy": 12345678,
  "updatedAt": "2025-03-15T13:48:28.000Z",
  "updatedBy": 12345678
}
```

**Save:** `id` → `postman_resources.postmanSpecId`

### 3.4 Update a Spec File (Update Existing Spec Content)

When a spec file is modified (e.g., new examples added, descriptions changed), update the file content in the existing spec.

```
PATCH /specs/:specId/files/:filePath
```

- `:specId` — the spec's ID from `postman_resources.postmanSpecId`
- `:filePath` — the file path within the spec (e.g., `openapi.yaml` from `postman_resources.postmanSpecFilePath`)

**Request Body:**
```json
{
  "content": "<updated YAML file content as a string>"
}
```

> **Important:** Only one property (`content`, `name`, or `type`) can be updated per call. For updating spec content, only send the `content` field.

**Response (200 OK):**
```json
{
  "id": "2fdc8ea1-d02e-4e50-989e-6fa28f42b995",
  "path": "openapi.yaml",
  "createdAt": "2025-07-22T19:29:08.000Z",
  "updatedAt": "2025-07-22T19:29:57.000Z",
  "createdBy": 12345678,
  "updatedBy": 12345678,
  "type": "DEFAULT"
}
```

### 3.5 Delete a Spec

```
DELETE /specs/:specId
```

**Response:** `204 No Content`

### 3.6 Get Spec Files

Used to retrieve the file path(s) within a spec. Useful if you need to confirm the `filePath` before updating.

```
GET /specs/:specId/files
```

**Response (200 OK):**
```json
{
  "files": [
    {
      "id": "7224f35c-d4d4-4e6e-8d73-21255ba87c0a",
      "path": "openapi.yaml",
      "name": "openapi.yaml",
      "type": "ROOT",
      "createdAt": "2025-07-22T18:46:25.000Z",
      "updatedAt": "2025-07-22T18:48:39.000Z",
      "createdBy": 12345678,
      "updatedBy": 12345678
    }
  ],
  "meta": {
    "nextCursor": null
  }
}
```

### 3.7 Generate a Collection from a Spec (Async)

Generates a Postman collection from a spec. This is an **asynchronous** operation — it returns a task ID that must be polled for completion.

**Step 1: Kick off generation**

```
POST /specs/:specId/generations/collection
```

**Request Body:**
```json
{
  "name": "CommerceHub - v1.26.0202",
  "options": {
    "folderStrategy": "Tags",
    "includeDeprecated": true,
    "enableOptionalParameters": true
  }
}
```

> The `options` field is optional. See [OpenAPI to Postman Converter Options](https://github.com/postmanlabs/openapi-to-postman/blob/develop/OPTIONS.md) for all available options.

**Response (202 Accepted):**
```json
{
  "taskId": "66ae9950-0869-4e65-96b0-1e0e47e771af",
  "url": "/specs/b4fc1bdc-6587-4f9b-95c9-f768146089b4/tasks/66ae9950-0869-4e65-96b0-1e0e47e771af"
}
```

**Step 2: Poll for task completion**

```
GET /specs/:specId/tasks/:taskId
```

Poll this endpoint every **2–3 seconds** until the `status` field is no longer `"pending"`.

**Response (pending):**
```json
{
  "status": "pending",
  "meta": {
    "model": "collection",
    "action": "generation"
  }
}
```

**Response (completed):**
```json
{
  "status": "completed",
  "meta": {
    "action": "generation",
    "model": "collection"
  },
  "details": {
    "resources": [
      {
        "id": "12345678-12ece9e1-2abf-4edc-8e34-de66e74114d2",
        "url": "/collections/12ece9e1-2abf-4edc-8e34-de66e74114d2"
      }
    ]
  }
}
```

The task response returns the collection UID in `details.resources[0].id`. To get both the plain ID and UID, make a follow-up call:

```
GET /collections/:collectionId
```

Pass the value from `resources[0].id` — this endpoint accepts either format. The response contains both values:
- `collection.info._postman_id` → plain collection ID → **save to** `postman_resources.postmanCollectionId`
- `collection.info.uid` → collection UID → **save to** `postman_resources.postmanCollectionUid`

**Which ID to use where:**

| Operation | Use |
|---|---|
| `GET /collections/:collectionId` | `postmanCollectionId` (plain ID) |
| `DELETE /collections/:collectionId` | `postmanCollectionId` (plain ID) |
| `POST /mocks` — `collection` body field | `postmanCollectionUid` (UID) |
| `PUT /collections/:collectionUid/synchronizations` | `postmanCollectionUid` (UID) |

### 3.8 Sync a Collection with its Spec (Async)

After updating a spec's content (section 3.4), the generated collection will be out of sync. Use this endpoint to re-sync the collection so it reflects the updated spec. The mock server automatically picks up changes from the collection — no mock server update is needed.

**Step 1: Kick off sync**

```
PUT /collections/:collectionUid/synchronizations?specId=<specId>
```

- `:collectionUid` — from `postman_resources.postmanCollectionUid`
- `specId` query parameter — from `postman_resources.postmanSpecId`

**No request body required.**

**Response (202 Accepted):**
```json
{
  "taskId": "66ae9950-0869-4e65-96b0-1e0e47e771af",
  "url": "/specs/b4fc1bdc-6587-4f9b-95c9-f768146089b4/tasks/66ae9950-0869-4e65-96b0-1e0e47e771af"
}
```

**Step 2: Poll for task completion**

Same polling mechanism as section 3.7, step 2. Poll `GET /specs/:specId/tasks/:taskId` until `status` is `"completed"`.

### 3.9 Create a Mock Server

Creates a mock server for a collection within a workspace.

```
POST /mocks?workspace=<workspaceId>
```

**Request Body:**
```json
{
  "mock": {
    "name": "CommerceHub - v1.26.0202 Mock",
    "collection": "12345678-12ece9e1-2abf-4edc-8e34-de66e74114d2",
    "private": false
  }
}
```

- `collection` — **required.** The collection **UID** from `postman_resources.postmanCollectionUid`.
- `private` — set to `false` so the mock server is publicly accessible (no `x-api-key` header required when calling the mock URL).
- `environment` — optional, not needed for this use case.

> **Important:** You cannot create mocks for collections added to an API definition. Since we are using collections generated from Spec Hub specs (not the legacy API Builder), this should not be an issue.

**Response (200 OK):**
```json
{
  "mock": {
    "id": "e3d951bf-873f-49ac-a658-b2dcb91d3289",
    "owner": "12345678",
    "uid": "12345678-e3d951bf-873f-49ac-a658-b2dcb91d3289",
    "collection": "12345678-12ece9e1-2abf-4edc-8e34-de66e74114d2",
    "mockUrl": "https://e3d951bf-873f-49ac-a658-b2dcb91d3289.mock.pstmn.io",
    "name": "CommerceHub - v1.26.0202 Mock",
    "config": {
      "headers": [],
      "matchBody": false,
      "matchQueryParams": true,
      "matchWildcards": true,
      "delay": null,
      "serverResponseId": null
    },
    "createdAt": "2022-06-09T19:00:39.000Z",
    "updatedAt": "2022-06-09T19:00:39.000Z"
  }
}
```

**Save:**
- `mock.id` → `postman_resources.postmanMockServerId`
- `mock.mockUrl` → `postman_resources.postmanMockServerUrl`

### 3.10 Delete a Mock Server

```
DELETE /mocks/:mockId
```

**Response (200 OK):**
```json
{
  "mock": {
    "id": "e3d951bf-873f-49ac-a658-b2dcb91d3289",
    "uid": "12345678-e3d951bf-873f-49ac-a658-b2dcb91d3289"
  }
}
```

### 3.11 Delete a Collection

```
DELETE /collections/:collectionId
```

**Response (200 OK):**
```json
{
  "collection": {
    "id": "12ece9e1-2abf-4edc-8e34-de66e74114d2",
    "uid": "12345678-12ece9e1-2abf-4edc-8e34-de66e74114d2"
  }
}
```

---

## 4. Ingestion Pipeline — Scenario Handling

When a GitHub webhook fires indicating a spec file change, the Backend Service must determine which scenario applies and execute the corresponding workflow.

### 4.0 Decision Tree

```
Webhook received (tenantName, version, action)
    │
    ├── action = "ADDED" or new file detected
    │   │
    │   ├── Does postman_tenants record exist for this tenantName?
    │   │   │
    │   │   ├── NO  → Scenario 1: New tenant + new spec
    │   │   └── YES → Does postman_resources record exist for this tenantName + version?
    │   │       │
    │   │       ├── NO  → Scenario 2: New version for existing tenant
    │   │       └── YES → Scenario 3: Spec update (resource already provisioned)
    │   │
    ├── action = "MODIFIED"
    │   │
    │   └── Does postman_resources record exist for this tenantName + version?
    │       │
    │       ├── YES → Scenario 3: Spec update
    │       └── NO  → Scenario 2: New version (treat as new if no resource exists)
    │   
    └── action = "DELETED"
        │
        └── Does this delete the last spec for this tenant?
            │
            ├── NO  → Scenario 4: Spec/version deleted
            └── YES → Scenario 5: Tenant deleted entirely
```

### 4.1 Scenario 1: New Tenant + First Spec

**Trigger:** A spec file is added for a product/tenant that has never been provisioned before. No `postman_tenants` record exists for this `tenantName`.

**Workflow:**

```
Step 1: Create workspace
    POST /workspaces
    Body: { workspace: { name: tenantName, type: "private", description: "..." } }
    → Save workspaceId

Step 2: (Optional) Set workspace roles to read-only for non-admins
    PATCH /workspaces/:workspaceId/roles
    → Set team members to Viewer (role "1") — can also be done manually later

Step 3: Save tenant record in MongoDB
    Insert into postman_tenants:
    {
      tenantName,
      postmanWorkspaceId: workspaceId,
      status: "ready",
      createdAt: now,
      updatedAt: now
    }

Step 4: Upload spec to Spec Hub
    POST /specs?workspaceId=<workspaceId>
    Body: {
      name: "{tenantName} - v{version}",
      type: "OPENAPI:3.0",
      files: [{ path: "openapi.yaml", content: yamlFileContent }]
    }
    → Save specId

Step 5: Generate collection from spec (async)
    POST /specs/:specId/generations/collection
    Body: { name: "{tenantName} - v{version}", options: { ... } }
    → Get taskId

Step 6: Poll for task completion
    GET /specs/:specId/tasks/:taskId
    → Poll every 2-3 seconds until status = "completed"
    → Get collectionIdFromTask from details.resources[0].id

Step 7: Fetch collection details to get both plain ID and UID
    GET /collections/:collectionIdFromTask
    → Save collectionId from response.collection.info._postman_id
    → Save collectionUid from response.collection.info.uid

Step 8: Create mock server
    POST /mocks?workspace=<workspaceId>
    Body: {
      mock: {
        name: "{tenantName} - v{version} Mock",
        collection: collectionUid,     ← must use UID here
        private: false
      }
    }
    → Save mockServerId and mockUrl

Step 9: Save resource record in MongoDB
    Insert into postman_resources:
    {
      tenantName,
      version,
      fileName,
      branchName,
      postmanSpecId: specId,
      postmanSpecFilePath: "openapi.yaml",
      postmanCollectionId: collectionId,
      postmanCollectionUid: collectionUid,
      postmanMockServerId: mockServerId,
      postmanMockServerUrl: mockUrl,
      status: "ready",
      createdAt: now,
      updatedAt: now
    }
```

### 4.2 Scenario 2: New Version for Existing Tenant

**Trigger:** A new spec version is added for a tenant that already has a workspace. A `postman_tenants` record exists, but no `postman_resources` record exists for this specific version.

**Workflow:**

```
Step 1: Look up existing workspace
    Find in postman_tenants: { tenantName }
    → Get workspaceId
    (No new workspace needed)

Step 2: Upload spec to Spec Hub
    POST /specs?workspaceId=<workspaceId>
    Body: {
      name: "{tenantName} - v{version}",
      type: "OPENAPI:3.0",
      files: [{ path: "openapi.yaml", content: yamlFileContent }]
    }
    → Save specId

Step 3: Generate collection from spec (async)
    POST /specs/:specId/generations/collection
    Body: { name: "{tenantName} - v{version}", options: { ... } }
    → Get taskId

Step 4: Poll for task completion
    GET /specs/:specId/tasks/:taskId
    → Poll every 2-3 seconds until status = "completed"
    → Get collectionIdFromTask from details.resources[0].id

Step 5: Fetch collection details to get both plain ID and UID
    GET /collections/:collectionIdFromTask
    → Save collectionId from response.collection.info._postman_id
    → Save collectionUid from response.collection.info.uid

Step 6: Create mock server
    POST /mocks?workspace=<workspaceId>
    Body: {
      mock: {
        name: "{tenantName} - v{version} Mock",
        collection: collectionUid,     ← must use UID here
        private: false
      }
    }
    → Save mockServerId and mockUrl

Step 7: Save resource record in MongoDB
    Insert into postman_resources:
    {
      tenantName,
      version,
      fileName,
      branchName,
      postmanSpecId: specId,
      postmanSpecFilePath: "openapi.yaml",
      postmanCollectionId: collectionId,
      postmanCollectionUid: collectionUid,
      postmanMockServerId: mockServerId,
      postmanMockServerUrl: mockUrl,
      status: "ready",
      createdAt: now,
      updatedAt: now
    }
```

### 4.3 Scenario 3: Existing Spec Updated

**Trigger:** An existing spec file is modified (e.g., new example responses added, descriptions changed, endpoints added/removed). A `postman_resources` record already exists for this `tenantName` + `version`.

**Workflow:**

```
Step 1: Look up existing resource
    Find in postman_resources: { tenantName, version }
    → Get postmanSpecId, postmanSpecFilePath, postmanCollectionUid

Step 2: Update spec file content
    PATCH /specs/:postmanSpecId/files/:postmanSpecFilePath
    Body: { content: updatedYamlFileContent }

Step 3: Sync collection with updated spec (async)
    PUT /collections/:postmanCollectionUid/synchronizations?specId=<postmanSpecId>
    → Get taskId

Step 4: Poll for task completion
    GET /specs/:postmanSpecId/tasks/:taskId
    → Poll every 2-3 seconds until status = "completed"

Step 5: Update resource record in MongoDB
    Update postman_resources where { tenantName, version }:
    {
      $set: {
        status: "ready",
        updatedAt: now
      }
    }
```

> **Note:** The mock server does NOT need to be recreated or updated. Mock servers are powered by the collection — once the collection is synced with the updated spec, the mock server automatically reflects the changes.

### 4.4 Scenario 4: Spec/Version Deleted

**Trigger:** A spec file for a specific version is deleted from GitHub, but other versions for the same tenant still exist.

**Workflow:**

```
Step 1: Look up existing resource
    Find in postman_resources: { tenantName, version }
    → Get postmanMockServerId, postmanCollectionId, postmanCollectionUid, postmanSpecId

Step 2: Delete mock server
    DELETE /mocks/:postmanMockServerId

Step 3: Delete collection
    DELETE /collections/:postmanCollectionId  (use plain ID, not UID)

Step 4: Delete spec
    DELETE /specs/:postmanSpecId

Step 5: Remove resource record from MongoDB
    Delete from postman_resources where { tenantName, version }
```

> **Deletion order matters:** Delete the mock server first, then the collection, then the spec. This avoids orphaned resources.

### 4.5 Scenario 5: Tenant Deleted Entirely

**Trigger:** All spec files for a tenant are removed, or a tenant is decommissioned.

**Workflow:**

```
Step 1: Get all resources for the tenant
    Find all in postman_resources: { tenantName }
    → List of all version resources

Step 2: For each resource, execute Scenario 4 (steps 2-5)
    Delete mock server → delete collection → delete spec → remove MongoDB record

Step 3: Delete workspace
    Look up postman_tenants: { tenantName }
    → Get postmanWorkspaceId
    DELETE /workspaces/:postmanWorkspaceId

Step 4: Remove tenant record from MongoDB
    Delete from postman_tenants where { tenantName }
```

---

## 5. Usage Pipeline — Serving Mock Responses

This section describes how the Backend Service should change to serve mock responses from Postman instead of Prism.

### 5.1 Current Flow (Prism — to be replaced)

The current `runSandboxWithAtlas` function in the Backend Service:
1. Receives the comboKey from the GraphQL request body.
2. Looks up the spec from `tenant_mock_api` collection using `findOne({ comboKey })`.
3. Parses the spec JSON.
4. Creates an ephemeral Prism server via `createPrismMockServer(specOrURL)`.
5. Sends the request payload to Prism and gets the mock response.
6. Closes the Prism server.
7. Returns the response to the UI.

### 5.2 New Flow (Postman Mock Server)

Replace the Prism logic with:

```typescript
// Step 1: Receive the request (unchanged)
const { body } = req;
const { key: comboKey } = body;

// Step 2: Extract tenantName and version from request parameters
// These are already available in the incoming GraphQL mutation variables:
//   productName, apiFullPath, requestType, apiVersion
// Alternatively, parse them from the comboKey:
//   comboKey format: "{productName}_{apiFullPath}_{requestType}_{version}"
const productName = body.productName;   // e.g., "CommerceHub"
const apiVersion = body.apiVersion;     // e.g., "1.26.0202"
const requestPath = body.requestPath;   // e.g., "/payments-vas/v1/accounts/balance-inquiry"
const requestType = body.requestType;   // e.g., "POST"

// Step 3: Look up the Postman mock server URL
const db = await DatabaseManager.getDatabase();
const postmanResourcesCollection = db.collection('postman_resources');
const resource = await postmanResourcesCollection.findOne({
  tenantName: productName,
  version: apiVersion
});

if (!resource || resource.status !== 'ready') {
  // Fallback: resource not provisioned yet or in error state
  res.status(503).json({ message: 'Mock server not available for this version' });
  return;
}

const mockBaseUrl = resource.postmanMockServerUrl;
// e.g., "https://e3d951bf-873f-49ac-a658-b2dcb91d3289.mock.pstmn.io"

// Step 4: Construct the full mock URL and proxy the request
const mockUrl = `${mockBaseUrl}${requestPath}`;
// e.g., "https://e3d951bf-873f-49ac-a658-b2dcb91d3289.mock.pstmn.io/payments-vas/v1/accounts/balance-inquiry"

// Step 5: Make the HTTP request to the Postman mock server
const headers = {
  'Content-Type': 'application/json',
  // Forward any other relevant headers from the original request
};

// If the UI sends request headers (from body.payload.headers), forward relevant ones
if (body?.payload?.headers) {
  // Forward headers that may affect mock response matching
  Object.assign(headers, body.payload.headers);
}

const response = await fetch(mockUrl, {
  method: requestType,                      // "POST", "GET", etc.
  headers: headers,
  body: requestType !== 'GET' ? JSON.stringify(body?.payload?.body || body?.requestBody) : undefined,
});

// Step 6: Return the response to the UI
const responseBody = await response.json();
const mockResponse = {
  responseBody: responseBody,
  responseHeaders: Object.fromEntries(response.headers.entries()),
  statusCode: response.status,
};

res.setHeader('Content-Type', 'application/json');
res.status(response.status);
res.end(JSON.stringify(mockResponse));
```

### 5.3 Key Differences from Prism Flow

| Aspect | Prism (old) | Postman (new) |
|--------|------------|---------------|
| Server lifecycle | Ephemeral — spin up per request, then close | Persistent — pre-provisioned, always running |
| Spec storage | Raw spec JSON in `tenant_mock_api.spec` | Managed by Postman Spec Hub |
| Response generation | Local Prism library processes spec | Postman cloud generates response from collection examples |
| Lookup | `comboKey` → spec content | `tenantName + version` → mock server URL |
| Latency | Prism startup overhead per request | Direct HTTP call to running mock server |

---

## 6. Backfill Script — One-Time Migration

The backfill script provisions Postman resources for all existing specs. This is a one-time operation run during the migration.

### 6.1 Prerequisites

- A Postman API key with permissions to create workspaces, specs, collections, and mock servers.
- Access to the GitHub repository containing all OpenAPI spec files.
- Access to MongoDB to read the existing `tenant_mock_api` collection and write to the new collections.

### 6.2 Input

The backfill script needs a list of `SpecEntry` objects — one per tenant + version combination. Each entry contains the tenant name, version, file name, branch name, and the full YAML content as a string.

You will need to write a `discoverSpecs()` function that traverses your GitHub repository and produces this list. This could also be derived from the existing `tenant_mock_api` MongoDB collection by grouping by `tenantName` and extracting unique versions from comboKeys — though you'd still need to fetch the original full YAML files from GitHub (the MongoDB records only contain per-endpoint chunks, not the full spec).

### 6.3 Pseudocode

```typescript
const POSTMAN_API_BASE = 'https://api.getpostman.com';
const POSTMAN_API_KEY = process.env.POSTMAN_API_KEY;
const POLL_INTERVAL_MS = 3000;
const MAX_POLL_ATTEMPTS = 60; // 3 minutes max wait

// ─── Helper: Postman API call ────────────────────────────────────
async function postmanApi(method: string, endpoint: string, body?: any) {
  const response = await fetch(`${POSTMAN_API_BASE}${endpoint}`, {
    method,
    headers: {
      'X-API-Key': POSTMAN_API_KEY,
      'Content-Type': 'application/json',
    },
    body: body ? JSON.stringify(body) : undefined,
  });

  if (response.status === 204) return null;

  const data = await response.json();

  if (!response.ok) {
    throw new Error(`Postman API error: ${response.status} - ${JSON.stringify(data)}`);
  }

  return data;
}

// ─── Helper: Poll async task until completion ────────────────────
async function pollTask(specId: string, taskId: string): Promise<any> {
  for (let attempt = 0; attempt < MAX_POLL_ATTEMPTS; attempt++) {
    const result = await postmanApi('GET', `/specs/${specId}/tasks/${taskId}`);

    if (result.status === 'completed') {
      return result;
    }

    if (result.status !== 'pending') {
      throw new Error(`Task failed with status: ${result.status}`);
    }

    await new Promise(resolve => setTimeout(resolve, POLL_INTERVAL_MS));
  }

  throw new Error(`Task timed out after ${MAX_POLL_ATTEMPTS * POLL_INTERVAL_MS / 1000}s`);
}

// ─── Helper: Provision a single spec version ─────────────────────
async function provisionSpecVersion(
  workspaceId: string,
  tenantName: string,
  version: string,
  fileName: string,
  branchName: string,
  yamlContent: string,
  db: Db
): Promise<void> {
  const postmanResources = db.collection('postman_resources');

  // Check if already provisioned (idempotency)
  const existing = await postmanResources.findOne({ tenantName, version });
  if (existing && existing.status === 'ready') {
    console.log(`  Skipping ${tenantName} v${version} — already provisioned`);
    return;
  }

  // Set status to provisioning
  await postmanResources.updateOne(
    { tenantName, version },
    {
      $set: { status: 'provisioning', updatedAt: new Date() },
      $setOnInsert: { tenantName, version, fileName, branchName, createdAt: new Date() }
    },
    { upsert: true }
  );

  try {
    // Step 1: Upload spec
    console.log(`  Uploading spec: ${tenantName} - v${version}`);
    const specResult = await postmanApi('POST', `/specs?workspaceId=${workspaceId}`, {
      name: `${tenantName} - v${version}`,
      type: 'OPENAPI:3.0',
      files: [{ path: 'openapi.yaml', content: yamlContent }],
    });
    const specId = specResult.id;

    // Step 2: Generate collection (async)
    console.log(`  Generating collection for spec: ${specId}`);
    const genResult = await postmanApi('POST', `/specs/${specId}/generations/collection`, {
      name: `${tenantName} - v${version}`,
      options: {
        folderStrategy: 'Tags',
        includeDeprecated: true,
        enableOptionalParameters: true,
      },
    });
    const taskId = genResult.taskId;

    // Step 3: Poll until collection is generated
    console.log(`  Polling task: ${taskId}`);
    const taskResult = await pollTask(specId, taskId);
    const collectionIdFromTask = taskResult.details.resources[0].id;

    // Step 4: Fetch collection details to get both plain ID and UID
    console.log(`  Fetching collection details for: ${collectionIdFromTask}`);
    const collectionDetails = await postmanApi('GET', `/collections/${collectionIdFromTask}`);
    const collectionId = collectionDetails.collection.info._postman_id;
    const collectionUid = collectionDetails.collection.info.uid;

    // Step 5: Create mock server (must use UID for the collection field)
    console.log(`  Creating mock server for collection: ${collectionUid}`);
    const mockResult = await postmanApi('POST', `/mocks?workspace=${workspaceId}`, {
      mock: {
        name: `${tenantName} - v${version} Mock`,
        collection: collectionUid,
        private: false,
      },
    });
    const mockServerId = mockResult.mock.id;
    const mockServerUrl = mockResult.mock.mockUrl;

    // Step 6: Update MongoDB
    await postmanResources.updateOne(
      { tenantName, version },
      {
        $set: {
          postmanSpecId: specId,
          postmanSpecFilePath: 'openapi.yaml',
          postmanCollectionId: collectionId,
          postmanCollectionUid: collectionUid,
          postmanMockServerId: mockServerId,
          postmanMockServerUrl: mockServerUrl,
          status: 'ready',
          updatedAt: new Date(),
        },
      }
    );

    console.log(`  ✓ Done: ${tenantName} v${version} → ${mockServerUrl}`);
  } catch (error) {
    // Mark as error for retry
    await postmanResources.updateOne(
      { tenantName, version },
      {
        $set: {
          status: 'error',
          errorMessage: error.message,
          updatedAt: new Date(),
        },
      }
    );
    console.error(`  ✗ Error provisioning ${tenantName} v${version}: ${error.message}`);
  }
}

// ─── Main backfill function ──────────────────────────────────────
// This function takes a list of spec entries and provisions them.
// YOU must implement discoverSpecs() to traverse your GitHub directory
// structure and produce this list — see section 6.4.

interface SpecEntry {
  tenantName: string;    // e.g., "CommerceHub"
  version: string;       // e.g., "1.26.0202"
  fileName: string;      // GitHub file path, e.g., "reference/1.26.0202/openapi.yaml"
  branchName: string;    // e.g., "main"
  yamlContent: string;   // Full YAML file content as a string
}

async function backfill(specs: SpecEntry[], db: Db) {
  const postmanTenants = db.collection('postman_tenants');
  const postmanResources = db.collection('postman_resources');

  // Create indexes
  await postmanTenants.createIndex({ tenantName: 1 }, { unique: true });
  await postmanResources.createIndex({ tenantName: 1, version: 1 }, { unique: true });
  await postmanResources.createIndex({ tenantName: 1 });

  // Group specs by tenant for workspace reuse
  const specsByTenant = new Map<string, SpecEntry[]>();
  for (const spec of specs) {
    const existing = specsByTenant.get(spec.tenantName) || [];
    existing.push(spec);
    specsByTenant.set(spec.tenantName, existing);
  }

  for (const [tenantName, tenantSpecs] of specsByTenant) {
    console.log(`\nProcessing tenant: ${tenantName}`);

    // Step 1: Ensure workspace exists for this tenant
    let tenantRecord = await postmanTenants.findOne({ tenantName });

    if (!tenantRecord) {
      console.log(`  Creating workspace for ${tenantName}`);
      const wsResult = await postmanApi('POST', '/workspaces', {
        workspace: {
          name: tenantName,
          type: 'private',
          description: `Postman workspace for ${tenantName} mock servers. Auto-provisioned by Dev Studio migration.`,
        },
      });
      const workspaceId = wsResult.workspace.id;

      // Note: Set workspace roles here if needed
      // await postmanApi('PATCH', `/workspaces/${workspaceId}/roles`, { ... });

      await postmanTenants.insertOne({
        tenantName,
        postmanWorkspaceId: workspaceId,
        status: 'ready',
        createdAt: new Date(),
        updatedAt: new Date(),
      });

      tenantRecord = { tenantName, postmanWorkspaceId: workspaceId };
      console.log(`  Workspace created: ${workspaceId}`);
    } else {
      console.log(`  Workspace already exists: ${tenantRecord.postmanWorkspaceId}`);
    }

    // Step 2: Process each version for this tenant
    for (const spec of tenantSpecs) {
      await provisionSpecVersion(
        tenantRecord.postmanWorkspaceId,
        spec.tenantName,
        spec.version,
        spec.fileName,
        spec.branchName,
        spec.yamlContent,
        db
      );
    }
  }

  console.log('\nBackfill complete.');
}
```

### 6.4 Running the Backfill

The backfill script expects a list of `SpecEntry` objects (see the `interface SpecEntry` in section 6.3). **You need to implement a `discoverSpecs()` function** that traverses your GitHub repository's directory structure and produces this list.

Since your team knows the GitHub directory structure best, write the parsing logic to:
1. Walk through the repository and find all OpenAPI YAML files for each tenant/product.
2. Extract the `tenantName`, `version`, `fileName`, and `branchName` from the file paths.
3. Read the file content as a string.
4. Return an array of `SpecEntry` objects.

**Example usage:**
```typescript
// You implement this based on your GitHub directory structure
const specs: SpecEntry[] = await discoverSpecs('/path/to/your/github/repo');

// Connect to MongoDB
const db = await DatabaseManager.getDatabase();

// Set the environment variable before running:
//   export POSTMAN_API_KEY="your-api-key"

// Run the backfill
await backfill(specs, db);
```

**Key properties of the backfill script:**
- It is **idempotent** — it checks for existing records before creating resources. If a run is interrupted, you can safely re-run it and it will skip already-provisioned versions and retry errored ones.
- It groups specs by tenant automatically, so workspaces are only created once per tenant regardless of the order of the input list.
- Failed provisions are marked with `status: "error"` in MongoDB so they can be identified and retried.

### 6.5 Post-Backfill Verification

After the backfill completes:

1. **Check MongoDB:** Verify that `postman_tenants` has one record per tenant and `postman_resources` has one record per tenant+version with `status: "ready"`.

2. **Spot-check Postman:** Open a few workspaces in the Postman UI and verify that specs, collections, and mock servers are present and functional.

3. **Test the usage pipeline:** Make a few test requests through the Dev Studio UI and verify that mock responses are being served from Postman instead of Prism.

---

## 7. Naming Conventions

These are recommended defaults. Adjust as needed to match your organization's standards.

| Resource | Naming Pattern | Example |
|----------|---------------|---------|
| Workspace | `{tenantName}` | `CommerceHub` |
| Spec | `{tenantName} - v{version}` | `CommerceHub - v1.26.0202` |
| Spec file path | `openapi.yaml` | `openapi.yaml` |
| Collection | `{tenantName} - v{version}` | `CommerceHub - v1.26.0202` |
| Mock Server | `{tenantName} - v{version} Mock` | `CommerceHub - v1.26.0202 Mock` |

---

## 8. Notes & Considerations

### 8.1 Status Field

The `status` field on both `postman_tenants` and `postman_resources` is critical for handling partial failures. If a provisioning pipeline fails mid-way (e.g., spec was created but collection generation timed out), the status will be `"error"` with a message in `errorMessage`. The pipeline should be designed to:
- Check the status before attempting to create resources.
- Resume from the last successful step when retrying (check which Postman resources already exist).
- Transition through: `provisioning` → `ready` (or `error`). Scenario 3 uses `syncing` → `ready`.

### 8.2 Rate Limits

The Postman API has rate limits. During the backfill, if you're provisioning hundreds of tenant+version combinations, consider:
- Adding a delay between API calls (e.g., 500ms–1s between operations).
- Implementing exponential backoff when receiving `429 Too Many Requests` responses.
- Rate limit headers are returned in `429` responses: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `RetryAfter`.
- Processing tenants sequentially but monitoring for 429s.

### 8.3 OpenAPI Spec Type Detection

The Fiserv Commerce Hub specs use `openapi: 3.0.2`, which maps to spec type `OPENAPI:3.0` in Postman. If other tenants use different OpenAPI versions, detect the version from the YAML content:
- `openapi: 3.0.x` → `OPENAPI:3.0`
- `openapi: 3.1.x` → `OPENAPI:3.1`
- `swagger: "2.0"` → `OPENAPI:2.0`

### 8.4 Spec File Size Limit

Postman's spec file upload has a **10 MB limit** per file. The CommerceHub specs are ~300-500 KB, well within this limit. If any tenant has specs approaching 10 MB, consider whether they can be split or optimized.

### 8.5 Collection Generation Options

When generating collections from specs, the `options` parameter affects how the collection is structured. Recommended options:
- `folderStrategy: "Tags"` — groups endpoints by their OpenAPI tags (e.g., "3-D Secure", "Payments"), matching how Fiserv organizes their APIs.
- `includeDeprecated: true` — includes deprecated endpoints so they remain available for testing.
- `enableOptionalParameters: true` — includes optional parameters in the generated requests.

### 8.6 Async Task Handling

Both collection generation (section 3.7) and collection sync (section 3.8) are async operations. Key considerations:
- Always poll until `status` is `"completed"` before proceeding to the next step.
- Set a reasonable timeout (e.g., 3 minutes) to avoid infinite polling.
- If a task gets stuck in `"pending"`, log the error and mark the resource as `"error"` for manual investigation.

### 8.7 Branch Handling

The existing `tenant_mock_api` collection has a `branchName` field (values include `develop`, `preview`, `main`). The current implementation stores this field in `postman_resources` for reference, but the initial recommendation is to **provision Postman resources only for the production branch** (typically `main` or `active`). If Fiserv needs mock servers for multiple branches per version, the schema can be extended with `branchName` as part of the compound unique index on `postman_resources`.

### 8.8 comboKey Parsing

The comboKey format is: `{productName}_{apiFullPath}_{requestType}_{version}`

**Example:** `CommerceHub_/payments-vas/v1/3ds/authenticate_post_1.26.0101`

To parse the version from a comboKey, split from the right since the `apiFullPath` may contain underscores. However, in the usage pipeline (section 5), you should prefer using the original request parameters (`productName`, `apiVersion`) that are available in the GraphQL mutation variables — this avoids the complexity of parsing the comboKey.

### 8.9 Fallback Strategy

During the migration transition period, consider keeping the Prism fallback active:
- If a `postman_resources` record is not found (status is not `"ready"`), fall back to the existing Prism flow.
- This allows for a gradual rollout: backfill one tenant at a time and verify before moving on.
- Once all tenants are migrated and verified, remove the Prism fallback code.

### 8.10 Mock Server URL Caching

Since the mock server URL for a given tenant+version changes very infrequently (only when the resource is deleted and recreated), consider caching the `postman_resources` lookup in memory with a TTL (e.g., 5 minutes). This eliminates the MongoDB lookup on every mock request.
