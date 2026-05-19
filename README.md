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

> **A note on environments and Postman cloud.** Fiserv runs four environments (`dev`, `qa`, `stage`, `prod`). Each environment has its own Backend Service instance and its own isolated MongoDB. Postman, however, has a single cloud — so workspaces from all four environments are provisioned into the same Postman tenant. To keep them distinguishable, workspace names append the environment (e.g., `DNA - dev`). Every other resource (spec, collection, mock server) lives inside its environment's workspace, so the environment is implicit beyond that point. See §8.10 for the full model.

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
    │  1. Resolve {tenantName, branchName, filePath} from request params via a Fiserv-owned utility
    │  2. Lookup postmanMockServerUrl from MongoDB (postman_resources)
    │  3. Proxy request to Postman mock server
    │
    ▼
Postman Mock Server ──► mock response ──► Backend Service ──► Dev Studio UI
```

**Flow:**
1. The Dev Studio UI sends the same GraphQL mutation (`RunTenantApiSandboxAtlas`) it sends today, with `productName`, `requestType`, `requestPath`, `apiVersion`, `requestBody`, `requestHeaders`, etc.
2. The Backend Service calls a Fiserv-owned utility — `resolveSpecLocation(request)` — that returns `{tenantName, branchName, filePath}`. The mechanism here is Fiserv's domain (e.g., a static map, a derived path, a separate lookup table). See §5.2.
3. With that triple, the Backend Service:
   - Looks up `postmanMockServerUrl` in `postman_resources` by `{tenantName, branchName, filePath}`.
   - Makes an HTTP request to `{postmanMockServerUrl}{requestPath}` with the appropriate method, headers, and body.
4. Returns the Postman mock server's response to the Dev Studio UI.

**The Dev Studio UI requires zero changes.** The swap happens entirely within the Backend Service.

### 1.3 Postman Resource Hierarchy

For each `(tenant, environment)` pair, the Postman resources are organized as follows:

- **1 Workspace per `(tenant, environment)`** — contains all specs, collections, and mock servers for that product within that environment.
- **N Specs per version** — multiple OpenAPI YAML files can live under the same version directory in the tenant's GitHub repo (different products, services, etc. under `reference/{version}/`). Each YAML file becomes its own Spec Hub entry.
- **1 Collection per spec** — generated from and synced to its spec. When the spec is updated and synced, the collection reflects the changes.
- **1 Mock Server per collection** — powered by the collection. When the collection updates, the mock server automatically serves the updated responses.

The relationship between Spec ↔ Collection ↔ Mock Server is still **1:1:1**. But for a single `{tenantName, version}` pair, there can be **multiple specs** — one per file under the tenant's GitHub `reference/{version}/` directory, and additionally one per branch that the environment ingests from. Specs, collections, and mock servers are all top-level resources within the workspace, linked to each other.

**Example for the DNA tenant in the `dev` environment:**
```
Workspace: "DNA - dev"
  │
  ├── Specs
  │    ├── "DNA - develop - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml"
  │    ├── "DNA - develop - 11.0.0/Accountholder/PersonService-11.0.0_DNA.yaml"
  │    └── "DNA - preview - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml"
  │
  ├── Collections (each generated from & synced to its respective spec)
  │    ├── "DNA - develop - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml"
  │    ├── "DNA - develop - 11.0.0/Accountholder/PersonService-11.0.0_DNA.yaml"
  │    └── "DNA - preview - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml"
  │
  └── Mock Servers (each powered by its respective collection)
       ├── "DNA - develop - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml Mock"  → https://<id-1>.mock.pstmn.io
       ├── "DNA - develop - 11.0.0/Accountholder/PersonService-11.0.0_DNA.yaml Mock"   → https://<id-2>.mock.pstmn.io
       └── "DNA - preview - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml Mock"  → https://<id-3>.mock.pstmn.io
```

Parallel workspaces — `DNA - qa`, `DNA - stage`, `DNA - prod` — exist for the other environments. All four live in the same Postman cloud and are provisioned by the four env-scoped Backend Service instances, each reading and writing its own env-isolated MongoDB.

**The chain of updates:** Spec updated → sync collection → collection reflects changes → mock server automatically serves updated responses.

---

## 2. MongoDB Schema Design

The migration introduces **two new MongoDB collections** — `postman_tenants` and `postman_resources` — within each environment's MongoDB instance. The previous `tenant_mock_api` collection is being retired as part of this migration; no schema there is touched.

### 2.1 New Collection: `postman_tenants`

**Purpose:** Maps each tenant/product to its Postman workspace within the current environment. This is a simple lookup table used to determine whether a workspace already exists for a given tenant in this environment.

**One document per tenant** (within each env-scoped MongoDB instance).

```typescript
interface PostmanTenant {
  tenantName: string;              // e.g., "DNA" — unique index
  postmanWorkspaceId: string;      // e.g., "1f0df51a-8658-4ee8-a2a1-d2567dfa09a9"
  status: "provisioning" | "ready" | "error";
  createdAt: Date;
  updatedAt: Date;
}
```

**Indexes:**
- Unique index on `tenantName` (each env's MongoDB is isolated, so `tenantName` alone is unique within an env's DB).

> The current environment is implicit via the env-scoped MongoDB connection and is NOT stored on the document. The Backend Service reads `environmentName` from its own app config when constructing the workspace name (see §3.1 / §7), but the value never round-trips through MongoDB.

**Example document (from the dev environment's MongoDB):**
```json
{
  "tenantName": "DNA",
  "postmanWorkspaceId": "1f0df51a-8658-4ee8-a2a1-d2567dfa09a9",
  "status": "ready",
  "createdAt": "2026-04-01T00:00:00.000Z",
  "updatedAt": "2026-04-01T00:00:00.000Z"
}
```

### 2.2 New Collection: `postman_resources`

**Purpose:** Stores the full set of Postman resource IDs for each spec file. This is the primary lookup table used by the usage pipeline to resolve a request to a mock server URL.

**One document per `(tenant, branch, filePath)` combination** — i.e., one document per spec file per branch within the env-scoped MongoDB. A single `{tenantName, version}` can have many documents because multiple files can live under the same `reference/{version}/` directory and across multiple branches.

```typescript
interface PostmanResource {
  tenantName: string;              // e.g., "DNA"
  branchName: string;              // e.g., "develop", "preview", "stage", "main" — from GitHub webhook
  filePath: string;                // full webhook path, e.g., "reference/11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml"
  version: string;                 // derived from filePath (first segment after "reference/"), e.g., "11.0.0" — stored for query convenience, NOT in lookup
  fileName: string;                // leaf only, e.g., "AddressService-11.0.0_DNA.yaml" — stored for convenience, NOT in lookup
  postmanSpecId: string;           // Spec Hub spec ID
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

> The spec's internal file path (the `path` field used when uploading to Postman's Spec Hub) is **not** stored — it's a code-level constant, `SPEC_FILE_PATH = "openapi.yaml"`. See §3.3 and §3.4.

**Indexes:**
- Compound unique index on `{ tenantName, branchName, filePath }` — this is the real identifier for a single spec/collection/mock triple, and the lookup key used by the usage pipeline.

> Environment is implicit via the env-scoped MongoDB connection and is NOT stored on the document.

**Example document (from the dev environment's MongoDB):**
```json
{
  "tenantName": "DNA",
  "branchName": "develop",
  "filePath": "reference/11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml",
  "version": "11.0.0",
  "fileName": "AddressService-11.0.0_DNA.yaml",
  "postmanSpecId": "b4fc1bdc-6587-4f9b-95c9-f768146089b4",
  "postmanCollectionId": "12ece9e1-2abf-4edc-8e34-de66e74114d2",
  "postmanCollectionUid": "12345678-12ece9e1-2abf-4edc-8e34-de66e74114d2",
  "postmanMockServerId": "e3d951bf-873f-49ac-a658-b2dcb91d3289",
  "postmanMockServerUrl": "https://e3d951bf-873f-49ac-a658-b2dcb91d3289.mock.pstmn.io",
  "status": "ready",
  "createdAt": "2026-04-01T00:00:00.000Z",
  "updatedAt": "2026-04-01T00:00:00.000Z"
}
```

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
    "name": "DNA - dev",
    "type": "private",
    "description": "Postman workspace for DNA mock servers (dev environment)"
  }
}
```

> **Note:** The `type` must be `"private"` to ensure workspace contents are not publicly accessible. Workspace name follows the convention `{tenantName} - {environmentName}` (see §7); the same tenant gets one workspace per Fiserv environment in the shared Postman cloud.

**Response (200 OK):**
```json
{
  "workspace": {
    "id": "1f0df51a-8658-4ee8-a2a1-d2567dfa09a9",
    "name": "DNA - dev"
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
  "name": "DNA - develop - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml",
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

> **Important:** The `content` field must contain the full YAML file as a **JSON-escaped string** — newlines as `\n`, double quotes as `\"`, backslashes as `\\`, tabs as `\t`, and any other control characters as `\uXXXX`. A typical OpenAPI YAML contains all of these (multi-line indentation, quoted `description` / `example` values, `pattern:` regexes), so the request body **must** be assembled through a real JSON serializer that handles escaping for you (e.g., Jackson `ObjectMapper.writeValueAsString(...)`, Gson `Gson().toJson(...)`, `JSON.stringify(...)`, Python `json.dumps(...)`) — passing the raw YAML string as a value. **Do not** concatenate or template the YAML into a JSON skeleton; the embedded newlines and quotes will produce a malformed body and the API will reject the request with `HTTP 400 "The JSON submitted is invalid, please check the syntax"` before any spec validation runs.
>
> The `path` field is the spec's internal file path. Always use the **`SPEC_FILE_PATH` constant** (`"openapi.yaml"`) — this value is never parameterized and is not stored in `postman_resources`. Files cannot exceed 10 MB. UTF-8 encoding is fine; there are no charset-specific requirements.
>
> The `name` field follows the naming convention in §7: `{tenantName} - {branchName} - {pathFromVersion}`, where `pathFromVersion` is the webhook file path with the `reference/` prefix stripped.
>
> A reference request body in the canonical shape, with a real escaped `content` value, is available in Postman's public workspace: [Create a spec — example](https://www.postman.com/postman/postman-public-workspace/example/12959542-6e5f003c-7511-474d-b2c0-898a4dfd20a1?sideView=agentMode).

**Response (201 Created):**
```json
{
  "name": "DNA - develop - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml",
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
PATCH /specs/:specId/files/openapi.yaml
```

- `:specId` — the spec's ID from `postman_resources.postmanSpecId`
- The file path segment is always the `SPEC_FILE_PATH` constant (`openapi.yaml`) — never parameterized and not stored.

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
  "name": "DNA - develop - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml",
  "options": {
    "folderStrategy": "Tags",
    "includeDeprecated": true,
    "enableOptionalParameters": true
  }
}
```

> Collection name matches the spec name (see §7).

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
    "name": "DNA - develop - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml Mock",
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
    "name": "DNA - develop - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml Mock",
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

Each Fiserv tenant has its own GitHub repository. An existing webhook listener service receives GitHub events and forwards a payload to the Backend Service. The payload carries the `tenantName`, `branchName`, and three arrays of file paths:

```json
{
  "tenantName": "DNA",
  "branchName": "develop",
  "added":    ["reference/11.0.0/Accountholder/PersonService-11.0.0_DNA.yaml"],
  "removed":  ["reference/10.9.0/Accountholder/AddressService-10.9.0_DNA.yaml"],
  "modified": ["reference/11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml"]
}
```

> **Note (POC).** The exact payload shape above is illustrative — for the POC we assume this is what the webhook listener forwards, but the production payload may look different. What this design depends on is only that each event identifies (a) which spec file path(s) changed and (b) the action per path — added, modified, or removed. As long as those two pieces of information are reachable from the payload, the Backend Service can normalize the inputs into the iteration in §4.0.2 without any other changes to this design.

The Backend Service iterates over each path in each array and routes it to the appropriate scenario.

### 4.0.1 Webhook Path Parsing Convention

All spec file paths follow the convention:

```
reference/{version}/<arbitrary intermediate directories>/{fileName}
```

From a path, derive:
- `filePath` — the **full string** as received from the webhook (used as the storage key in `postman_resources`).
- `version` — the segment immediately after `reference/` (first path segment).
- `fileName` — the leaf segment (the YAML file name).
- `pathFromVersion` — `filePath` with the `reference/` prefix stripped (used in resource naming, see §7).

**Example:** for `reference/11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml`:
- `version` = `11.0.0`
- `fileName` = `AddressService-11.0.0_DNA.yaml`
- `pathFromVersion` = `11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml`

### 4.0.2 Decision Tree

For each `filePath` in each array (`added` / `modified` / `removed`):

```
For each entry in webhook (tenantName, branchName, filePath, arrayName):
    │
    ├── arrayName = "added"
    │   │
    │   ├── Does postman_tenants record exist for this tenantName?
    │   │   │
    │   │   ├── NO  → Scenario 1: New tenant + new spec file
    │   │   └── YES → Scenario 2: New spec file in existing workspace
    │   │
    ├── arrayName = "modified"
    │   │
    │   └── Does postman_resources record exist for { tenantName, branchName, filePath }?
    │       │
    │       ├── YES → Scenario 3: Spec file update
    │       └── NO  → Scenario 2: Treat as new (defensive — webhook claims modify, but we have no record)
    │
    └── arrayName = "removed"
        │
        └── After this delete, do any postman_resources records remain for this tenantName (within this env's MongoDB)?
            │
            ├── YES → Scenario 4: Spec file deleted
            └── NO  → Scenario 5: Tenant deleted entirely (workspace cleanup)
```

> A single webhook event can fan out into multiple scenario executions — e.g., a PR that adds two files and modifies one will trigger Scenario 2 twice and Scenario 3 once.

### 4.1 Scenario 1: New Tenant + First Spec File

**Trigger:** A path in `added` arrives for a tenant that has never been provisioned in this environment. No `postman_tenants` record exists for this `tenantName`.

**Inputs:** `tenantName`, `branchName`, `filePath` (full webhook path). The Backend Service also reads `environmentName` from its own config to construct the workspace name in Step 1 — it is not stored on any document. Derive `version`, `fileName`, `pathFromVersion` per §4.0.1.

**Workflow:**

```
Step 1: Create workspace
    POST /workspaces
    Body: { workspace: { name: "{tenantName} - {environmentName}", type: "private", description: "..." } }
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
      name: "{tenantName} - {branchName} - {pathFromVersion}",
      type: "OPENAPI:3.0",
      files: [{ path: SPEC_FILE_PATH, content: yamlFileContent }]   // SPEC_FILE_PATH = "openapi.yaml"
    }
    → Save specId

Step 5: Generate collection from spec (async)
    POST /specs/:specId/generations/collection
    Body: { name: "{tenantName} - {branchName} - {pathFromVersion}", options: { ... } }
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
        name: "{tenantName} - {branchName} - {pathFromVersion} Mock",
        collection: collectionUid,     ← must use UID here
        private: false
      }
    }
    → Save mockServerId and mockUrl

Step 9: Save resource record in MongoDB
    Insert into postman_resources:
    {
      tenantName,
      branchName,
      filePath,
      version,
      fileName,
      postmanSpecId: specId,
      postmanCollectionId: collectionId,
      postmanCollectionUid: collectionUid,
      postmanMockServerId: mockServerId,
      postmanMockServerUrl: mockUrl,
      status: "ready",
      createdAt: now,
      updatedAt: now
    }
```

### 4.2 Scenario 2: New Spec File in Existing Workspace

**Trigger:** A path in `added` (or a `modified` path with no matching record) arrives for a tenant that already has a workspace in this environment. A `postman_tenants` record exists, but no `postman_resources` record exists for `{tenantName, branchName, filePath}`. This covers two real-world cases:
- A new spec file added to a brand-new version directory.
- A new spec file added under an existing version directory (different product or service under the same `reference/{version}/`).
- A new branch ingested for a previously-seen file.

**Workflow:**

```
Step 1: Look up existing workspace
    Find in postman_tenants: { tenantName }
    → Get workspaceId
    (No new workspace needed — env is implicit via DB isolation)

Step 2: Upload spec to Spec Hub
    POST /specs?workspaceId=<workspaceId>
    Body: {
      name: "{tenantName} - {branchName} - {pathFromVersion}",
      type: "OPENAPI:3.0",
      files: [{ path: SPEC_FILE_PATH, content: yamlFileContent }]
    }
    → Save specId

Step 3: Generate collection from spec (async)
    POST /specs/:specId/generations/collection
    Body: { name: "{tenantName} - {branchName} - {pathFromVersion}", options: { ... } }
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
        name: "{tenantName} - {branchName} - {pathFromVersion} Mock",
        collection: collectionUid,     ← must use UID here
        private: false
      }
    }
    → Save mockServerId and mockUrl

Step 7: Save resource record in MongoDB
    Insert into postman_resources:
    {
      tenantName,
      branchName,
      filePath,
      version,
      fileName,
      postmanSpecId: specId,
      postmanCollectionId: collectionId,
      postmanCollectionUid: collectionUid,
      postmanMockServerId: mockServerId,
      postmanMockServerUrl: mockUrl,
      status: "ready",
      createdAt: now,
      updatedAt: now
    }
```

### 4.3 Scenario 3: Existing Spec File Updated

**Trigger:** A path in `modified` matches an existing `postman_resources` record for `{tenantName, branchName, filePath}`.

**Workflow:**

```
Step 1: Look up existing resource
    Find in postman_resources: { tenantName, branchName, filePath }
    → Get postmanSpecId, postmanCollectionUid

Step 2: Update spec file content
    PATCH /specs/:postmanSpecId/files/openapi.yaml
    Body: { content: updatedYamlFileContent }
    // file path segment is always the SPEC_FILE_PATH constant

Step 3: Sync collection with updated spec (async)
    PUT /collections/:postmanCollectionUid/synchronizations?specId=<postmanSpecId>
    → Get taskId

Step 4: Poll for task completion
    GET /specs/:postmanSpecId/tasks/:taskId
    → Poll every 2-3 seconds until status = "completed"

Step 5: Update resource record in MongoDB
    Update postman_resources where { tenantName, branchName, filePath }:
    {
      $set: {
        status: "ready",
        updatedAt: now
      }
    }
```

> **Note:** The mock server does NOT need to be recreated or updated. Mock servers are powered by the collection — once the collection is synced with the updated spec, the mock server automatically reflects the changes.

### 4.4 Scenario 4: Spec File Deleted

**Trigger:** A path in `removed` matches an existing `postman_resources` record, AND other `postman_resources` records still exist for this `tenantName` in this env's MongoDB.

**Workflow:**

```
Step 1: Look up existing resource
    Find in postman_resources: { tenantName, branchName, filePath }
    → Get postmanMockServerId, postmanCollectionId, postmanCollectionUid, postmanSpecId

Step 2: Delete mock server
    DELETE /mocks/:postmanMockServerId

Step 3: Delete collection
    DELETE /collections/:postmanCollectionId  (use plain ID, not UID)

Step 4: Delete spec
    DELETE /specs/:postmanSpecId

Step 5: Remove resource record from MongoDB
    Delete from postman_resources where { tenantName, branchName, filePath }
```

> **Deletion order matters:** Delete the mock server first, then the collection, then the spec. This avoids orphaned resources.

### 4.5 Scenario 5: Tenant Deleted Entirely (in this environment)

**Trigger:** The last spec file for a tenant is removed within this env, or the tenant is decommissioned. Scope is `(tenant, environment)` — parallel workspaces in other envs (e.g., `DNA - prod`) are not affected.

**Workflow:**

```
Step 1: Get all resources for the tenant (in this env's DB)
    Find all in postman_resources: { tenantName }
    → List of remaining resource documents

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
1. Receives request parameters from the GraphQL request body (`productName`, `apiVersion`, `requestPath`, `requestType`, etc.).
2. Looks up the spec from MongoDB (the legacy `tenant_mock_api` lookup path).
3. Parses the spec JSON.
4. Creates an ephemeral Prism server via `createPrismMockServer(specOrURL)`.
5. Sends the request payload to Prism and gets the mock response.
6. Closes the Prism server.
7. Returns the response to the UI.

The legacy `tenant_mock_api` collection (and the comboKey concept that keyed into it) is **retired** as part of this migration. Neither survives in the new flow.

### 5.2 New Flow (Postman Mock Server)

The lookup is **single-step** against `postman_resources`, keyed by `{tenantName, branchName, filePath}`. The triple comes from a Fiserv-owned utility — `resolveSpecLocation(request)` — that translates the incoming GraphQL request into the corresponding spec file location. The utility's implementation is Fiserv's domain (similar to `fetchYamlFromGitHub()` in §6.3); this document only defines its **contract**.

**`resolveSpecLocation(request)` contract:**
- **Input:** the incoming request — at minimum `productName`, `apiVersion`, `requestPath`, `requestType` and any other GraphQL mutation variables the utility needs.
- **Output:** `{ tenantName: string, branchName: string, filePath: string }`, where:
  - `tenantName` matches the tenant whose GitHub repo owns this spec.
  - `branchName` is the Git branch this environment currently serves for that tenant.
  - `filePath` is the full webhook-style path (e.g., `reference/11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml`) and matches `postman_resources.filePath`.

The mechanism inside the utility is up to Fiserv — possibilities include a config-driven map, a derived rule based on `apiVersion` + `requestPath`, or a lightweight lookup against any new directory/inventory Fiserv chooses to maintain.

```typescript
// Step 1: Receive the request (unchanged)
const { body } = req;
const requestPath = body.requestPath;   // e.g., "/payments-vas/v1/accounts/balance-inquiry"
const requestType = body.requestType;   // e.g., "POST"

// Step 2: Resolve where this request maps to in the spec catalog.
// resolveSpecLocation() is YOUR utility (see contract above).
const { tenantName, branchName, filePath } = await resolveSpecLocation(body);

// Step 3: Look up the Postman mock server URL.
// Lookup key: { tenantName, branchName, filePath }.
// Environment is implicit via env-isolated MongoDB connection.
const db = await DatabaseManager.getDatabase();
const resource = await db.collection('postman_resources').findOne({
  tenantName,
  branchName,
  filePath,
});

if (!resource || resource.status !== 'ready') {
  res.status(503).json({ message: 'Mock server not available for this spec file' });
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
|--------|-------------|---------------|
| Server lifecycle | Ephemeral — spin up per request, then close | Persistent — pre-provisioned, always running |
| Spec storage | Raw spec JSON in MongoDB (`tenant_mock_api`) | Managed by Postman Spec Hub |
| Response generation | Local Prism library processes spec | Postman cloud generates response from collection examples |
| Lookup | `comboKey` → spec content in `tenant_mock_api` | `resolveSpecLocation(request)` → `{tenantName, branchName, filePath}` → `postman_resources` → mock server URL |
| Latency | Prism startup overhead per request | Direct HTTP call to running mock server |

---

## 6. Backfill Script — One-Time Migration

The backfill script provisions Postman resources for all existing specs. This is a one-time operation **run once per environment** — four runs in total (dev, qa, stage, prod). Each run connects to its env's own MongoDB and uses its env name to construct workspace names; the env name itself is never persisted on a document.

### 6.1 Prerequisites

- A Postman API key with permissions to create workspaces, specs, collections, and mock servers.
- Access to the env-scoped MongoDB to write the new `postman_tenants` and `postman_resources` collections. (The legacy `tenant_mock_api` collection is NOT read by the backfill.)
- Access to each tenant's GitHub repository — one repo per tenant — so the backfill can list the spec files under `reference/{version}/…` and fetch their contents.
- A configured mapping of branches per environment (which branches this env serves). See §8.7 for the recommended mapping.
- The list of tenants to provision (e.g., from a config file or by enumerating the GitHub org).
- An environment identifier (`dev`, `qa`, `stage`, or `prod`) available via app config / env var.

### 6.2 Input & Discovery Strategy

The backfill discovers what to provision **by walking each tenant's GitHub repo** for the branches this environment serves. For each `(tenantName, branchName)` pair, it lists every YAML file under `reference/{version}/…` and treats each file path as one provisioning unit.

For each environment, the script:

1. **Enumerates `(tenantName, branchName)` pairs** for the tenants this env serves, using the configured branch-per-env mapping (§8.7).
2. **Walks GitHub** for each pair — listing every file matching `reference/{version}/**/*.yaml` (or `*.yml`) on the given branch in the tenant's repo. Each listed path becomes the `filePath` stored in `postman_resources`.
3. **Fetches the full YAML file from GitHub** for each path on its branch.
4. **Feeds each `(tenantName, branchName, filePath)` triple** into the provisioning workflow.

`environmentName` is read from the Backend Service's config and used to construct workspace names. It is not stored on any MongoDB document — env-scoped DBs already isolate each env's records. No cross-env coordination is required.

Two Fiserv-owned utility functions are needed here. Both are left as stubs in the pseudocode below for Fiserv to implement (they depend on how Fiserv organizes its GitHub access):

- **`listSpecFilesForTenantBranch(tenantName, branchName) → string[]`** — returns the list of webhook-style file paths (e.g., `reference/11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml`) present on the given branch in the tenant's GitHub repo.
- **`fetchYamlFromGitHub(tenantName, filePath, branchName) → string`** — returns the raw YAML content of one specific file.

### 6.3 Pseudocode

```typescript
const POSTMAN_API_BASE = 'https://api.getpostman.com';
const POSTMAN_API_KEY  = process.env.POSTMAN_API_KEY;
const ENVIRONMENT_NAME = process.env.ENVIRONMENT_NAME;   // "dev" | "qa" | "stage" | "prod"
const SPEC_FILE_PATH   = 'openapi.yaml';                 // internal path inside every Postman spec
const POLL_INTERVAL_MS = 3000;
const MAX_POLL_ATTEMPTS = 60; // 3 minutes max wait

// ─── Helper: parse webhook path ──────────────────────────────────
// reference/{version}/.../{fileName}
function parseFilePath(filePath: string): { version: string; fileName: string; pathFromVersion: string } {
  const stripped = filePath.startsWith('reference/') ? filePath.slice('reference/'.length) : filePath;
  const segments = stripped.split('/');
  return {
    version: segments[0],
    fileName: segments[segments.length - 1],
    pathFromVersion: stripped,
  };
}

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

// ─── Helper: Provision a single spec file ────────────────────────
async function provisionSpecFile(
  workspaceId: string,
  tenantName: string,
  branchName: string,
  filePath: string,           // full webhook path
  yamlContent: string,
  db: Db
): Promise<void> {
  const postmanResources = db.collection('postman_resources');
  const { version, fileName, pathFromVersion } = parseFilePath(filePath);
  const resourceName = `${tenantName} - ${branchName} - ${pathFromVersion}`;

  // Check if already provisioned (idempotency)
  const existing = await postmanResources.findOne({ tenantName, branchName, filePath });
  if (existing && existing.status === 'ready') {
    console.log(`  Skipping ${resourceName} — already provisioned`);
    return;
  }

  // Set status to provisioning
  await postmanResources.updateOne(
    { tenantName, branchName, filePath },
    {
      $set: { status: 'provisioning', updatedAt: new Date() },
      $setOnInsert: {
        tenantName,
        branchName,
        filePath,
        version,
        fileName,
        createdAt: new Date(),
      },
    },
    { upsert: true }
  );

  try {
    // Step 1: Upload spec
    console.log(`  Uploading spec: ${resourceName}`);
    const specResult = await postmanApi('POST', `/specs?workspaceId=${workspaceId}`, {
      name: resourceName,
      type: 'OPENAPI:3.0',
      files: [{ path: SPEC_FILE_PATH, content: yamlContent }],
    });
    const specId = specResult.id;

    // Step 2: Generate collection (async)
    console.log(`  Generating collection for spec: ${specId}`);
    const genResult = await postmanApi('POST', `/specs/${specId}/generations/collection`, {
      name: resourceName,
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
    const collectionId  = collectionDetails.collection.info._postman_id;
    const collectionUid = collectionDetails.collection.info.uid;

    // Step 5: Create mock server (must use UID for the collection field)
    console.log(`  Creating mock server for collection: ${collectionUid}`);
    const mockResult = await postmanApi('POST', `/mocks?workspace=${workspaceId}`, {
      mock: {
        name: `${resourceName} Mock`,
        collection: collectionUid,
        private: false,
      },
    });
    const mockServerId  = mockResult.mock.id;
    const mockServerUrl = mockResult.mock.mockUrl;

    // Step 6: Update MongoDB
    await postmanResources.updateOne(
      { tenantName, branchName, filePath },
      {
        $set: {
          postmanSpecId: specId,
          postmanCollectionId: collectionId,
          postmanCollectionUid: collectionUid,
          postmanMockServerId: mockServerId,
          postmanMockServerUrl: mockServerUrl,
          status: 'ready',
          updatedAt: new Date(),
        },
      }
    );

    console.log(`  ✓ Done: ${resourceName} → ${mockServerUrl}`);
  } catch (error) {
    // Mark as error for retry
    await postmanResources.updateOne(
      { tenantName, branchName, filePath },
      {
        $set: {
          status: 'error',
          errorMessage: error.message,
          updatedAt: new Date(),
        },
      }
    );
    console.error(`  ✗ Error provisioning ${resourceName}: ${error.message}`);
  }
}

// ─── Config: which tenants and which branches this env serves ────
// Source these from your config — example shown inline for clarity.
const TENANTS: string[] = ['DNA', 'CommerceHub', /* ... */];

const BRANCHES_FOR_ENV: Record<string, string[]> = {
  dev:   ['develop', 'preview'],
  qa:    ['develop'],
  stage: ['stage'],
  prod:  ['main'],
};

// ─── Fiserv-owned helper: list spec files in a tenant's GitHub repo ───
// YOU must implement this. It should return every webhook-style spec file path
// under `reference/{version}/.../*.yaml` (or `.yml`) present on the given branch
// in the tenant's repo. Examples:
//   ["reference/11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml",
//    "reference/11.0.0/Accountholder/PersonService-11.0.0_DNA.yaml", ...]
//
// Implementation options:
//   - GitHub API: GET /repos/{owner}/{tenantName}/git/trees/{branch}?recursive=1, filter to .yaml/.yml under reference/
//   - Local clone: walk the filesystem of a checked-out copy of the branch
//
async function listSpecFilesForTenantBranch(
  tenantName: string,
  branchName: string,
): Promise<string[]> {
  throw new Error('listSpecFilesForTenantBranch() not implemented — see comments above');
}

// ─── Fiserv-owned helper: fetch full YAML from GitHub ────────────
// YOU must implement this. It should fetch the raw file content from the tenant's
// GitHub repository for a specific filePath on a specific branch.
//
// Implementation options:
//   - GitHub API: GET /repos/{owner}/{tenantName}/contents/{filePath}?ref={branch}
//   - Direct raw URL: https://raw.githubusercontent.com/{owner}/{tenantName}/{branch}/{filePath}
//   - Local clone: fs.readFileSync() if you have the repo checked out
//
async function fetchYamlFromGitHub(
  tenantName: string,
  filePath: string,
  branchName: string,
): Promise<string> {
  throw new Error('fetchYamlFromGitHub() not implemented — see comments above');
}

// ─── Main backfill function ──────────────────────────────────────
// Run once per environment. For each configured (tenant, branch) pair, walks
// the tenant's GitHub repo for spec files, fetches each YAML, and provisions
// the corresponding Postman workspace, spec, collection, and mock server.

async function backfill(db: Db) {
  if (!ENVIRONMENT_NAME) {
    throw new Error('ENVIRONMENT_NAME env var must be set (dev | qa | stage | prod)');
  }

  const branchesForThisEnv = BRANCHES_FOR_ENV[ENVIRONMENT_NAME];
  if (!branchesForThisEnv || branchesForThisEnv.length === 0) {
    throw new Error(`No branches configured for environment "${ENVIRONMENT_NAME}"`);
  }

  const postmanTenants   = db.collection('postman_tenants');
  const postmanResources = db.collection('postman_resources');

  // Create indexes for the new collections
  await postmanTenants.createIndex({ tenantName: 1 }, { unique: true });
  await postmanResources.createIndex(
    { tenantName: 1, branchName: 1, filePath: 1 },
    { unique: true },
  );

  // ── Step 1: Process each tenant ──
  for (const tenantName of TENANTS) {
    console.log(`Processing tenant: ${tenantName}`);

    // Ensure workspace exists for this tenant in this env
    let tenantRecord = await postmanTenants.findOne({ tenantName });

    if (!tenantRecord) {
      const workspaceName = `${tenantName} - ${ENVIRONMENT_NAME}`;
      console.log(`  Creating workspace: ${workspaceName}`);
      const wsResult = await postmanApi('POST', '/workspaces', {
        workspace: {
          name: workspaceName,
          type: 'private',
          description: `Postman workspace for ${tenantName} mock servers (${ENVIRONMENT_NAME} environment). Auto-provisioned by Dev Studio migration.`,
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

    // ── Step 2: For each branch this env serves, list spec files and provision ──
    for (const branchName of branchesForThisEnv) {
      console.log(`  Listing spec files: ${tenantName} @ ${branchName}`);
      let filePaths: string[];
      try {
        filePaths = await listSpecFilesForTenantBranch(tenantName, branchName);
      } catch (error) {
        console.error(`  ✗ Failed to list specs for ${tenantName} @ ${branchName}: ${error.message}`);
        continue;
      }
      console.log(`    Found ${filePaths.length} spec file(s) on ${branchName}`);

      for (const filePath of filePaths) {
        console.log(`    Fetching YAML: ${filePath}`);
        let yamlContent: string;
        try {
          yamlContent = await fetchYamlFromGitHub(tenantName, filePath, branchName);
        } catch (error) {
          console.error(`    ✗ Failed to fetch YAML for ${tenantName} ${branchName} ${filePath}: ${error.message}`);
          continue;
        }

        await provisionSpecFile(
          tenantRecord.postmanWorkspaceId,
          tenantName,
          branchName,
          filePath,
          yamlContent,
          db,
        );
      }
    }
  }

  console.log(`\n[${ENVIRONMENT_NAME}] Backfill complete.`);
}
```

### 6.4 Running the Backfill

The backfill is run **once per environment** — four total invocations (dev, qa, stage, prod). Each run connects to its own env's MongoDB, walks each tenant's GitHub repo for the branches this env serves, and stamps that env's name on every write.

**Before running,** implement the two Fiserv-owned helpers from §6.3:
- `listSpecFilesForTenantBranch(tenantName, branchName)` — lists every spec file path on a branch.
- `fetchYamlFromGitHub(tenantName, filePath, branchName)` — fetches one file's YAML content.

Also confirm `TENANTS` and `BRANCHES_FOR_ENV` reflect Fiserv's actual configuration.

**Example usage (per environment):**
```typescript
// Set env vars before running:
//   export POSTMAN_API_KEY="your-api-key"
//   export ENVIRONMENT_NAME="dev"      // or "qa" / "stage" / "prod"

// Connect to this environment's MongoDB
const db = await DatabaseManager.getDatabase();

await backfill(db);
```

Repeat for each of the four environments, each pointing at its own MongoDB instance.

**How it works:**
1. For the current env, looks up the branches it serves from `BRANCHES_FOR_ENV`.
2. For each tenant in `TENANTS`, ensures a workspace exists in Postman named `{tenantName} - {environmentName}` and recorded in `postman_tenants`.
3. For each `(tenantName, branchName)` pair, calls `listSpecFilesForTenantBranch()` to enumerate every spec file path on that branch.
4. For each `filePath`, derives `version` / `fileName` / `pathFromVersion`, fetches the YAML via `fetchYamlFromGitHub()`, runs the provisioning workflow (spec → collection → mock server), and persists a `postman_resources` document keyed by `{tenantName, branchName, filePath}`.

**Key properties:**
- **GitHub-driven discovery** — the source of truth is each tenant's GitHub repo. No dependency on any legacy MongoDB collection.
- **Per-env isolation** — each run reads/writes only its own MongoDB. No cross-env coordination.
- **Idempotent** — checks for existing `postman_resources` records (by `{tenantName, branchName, filePath}`) before creating Postman resources. Safe to re-run after interruptions; skips already-provisioned files and retries errored ones.
- **Automatic tenant grouping** — workspaces are created at most once per `(tenant, env)`.
- **Error isolation** — a failed GitHub list/fetch or Postman API call for one spec file does not block others. Failed entries are marked `status: "error"` for identification and retry.

### 6.5 Post-Backfill Verification

After each environment's backfill completes:

1. **Check MongoDB:** Verify that `postman_tenants` has one record per tenant (in this env's DB) and `postman_resources` has one record per `{tenantName, branchName, filePath}` with `status: "ready"`.

2. **Spot-check Postman:** Open a few `{tenant} - {env}` workspaces in the Postman UI and verify that specs, collections, and mock servers are present and functional.

3. **Test the usage pipeline:** Make a few test requests through the Dev Studio UI in this environment and verify that mock responses are being served from Postman instead of Prism.

---

## 7. Naming Conventions

| Resource | Naming Pattern | Example |
|----------|----------------|---------|
| Workspace | `{tenantName} - {environmentName}` | `DNA - dev` |
| Spec | `{tenantName} - {branchName} - {pathFromVersion}` | `DNA - develop - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml` |
| Spec internal file path (code constant) | `SPEC_FILE_PATH = "openapi.yaml"` | `openapi.yaml` |
| Collection | same as Spec | same as Spec |
| Mock Server | `{Spec name} Mock` | `DNA - develop - 11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml Mock` |

Where:
- `environmentName` ∈ `{"dev", "qa", "stage", "prod"}` (from Backend Service config).
- `branchName` is the Git branch reported by the webhook (e.g., `develop`, `preview`, `stage`, `main`).
- `pathFromVersion` is the webhook `filePath` with the `reference/` prefix stripped — e.g., `11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml`. The version is already the first segment of this path, so it's not repeated as a separate `v{version}` prefix.

> Lookups never read the resource name; they query `postman_resources` by `{tenantName, branchName, filePath}`. The name exists only for human readability in the Postman UI.

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

Postman's spec file upload has a **10 MB limit** per file. The Fiserv specs we've sampled (e.g., CommerceHub) are ~300–500 KB, well within this limit. If any tenant has specs approaching 10 MB, consider whether they can be split or optimized.

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

Branches are **first-class** in this design — they're part of the compound unique key on `postman_resources` (`{tenantName, branchName, filePath}`) and appear in spec / collection / mock server names.

Each Backend Service instance ingests from one or more branches based on its environment. The current mapping (subject to confirmation per tenant):

| Environment | Ingests from branch(es) |
|---|---|
| `dev`   | `develop` (plus `preview` where used) |
| `qa`    | `develop` |
| `stage` | `stage` |
| `prod`  | `main` |

A single workspace (e.g., `DNA - dev`) can therefore contain multiple specs that share the same file path but differ in `branchName` — for example, an `AddressService-…yaml` from `develop` alongside the same path from `preview`. This is expected; they are independent Postman resources.

### 8.8 Fallback Strategy

During the migration transition period, consider keeping the Prism fallback active:
- If a `postman_resources` record is not found (status is not `"ready"`), fall back to the existing Prism flow.
- This allows for a gradual rollout: backfill one tenant at a time and verify before moving on.
- Once all tenants are migrated and verified, remove the Prism fallback code.

### 8.9 Mock Server URL Caching

Since the mock server URL for a given `{tenantName, branchName, filePath}` changes very infrequently (only when the resource is deleted and recreated), consider caching the `postman_resources` lookup in memory with a TTL (e.g., 5 minutes). This eliminates the MongoDB lookup on every mock request.

### 8.10 Environment Handling

Postman has a **single cloud**; Fiserv has **four environments** (`dev`, `qa`, `stage`, `prod`). The reconciliation is per-environment workspaces in the same shared Postman tenant:

- Each Fiserv environment runs its own Backend Service instance.
- Each Backend Service instance has its own isolated MongoDB — env-scoped reads and writes.
- All four instances target the **same Postman cloud**. They distinguish their resources by **workspace name only** — `{tenantName} - {environmentName}` (e.g., `DNA - dev`, `DNA - qa`, `DNA - stage`, `DNA - prod`).
- `environmentName` is sourced from each Backend Service's app config (env var, config file, etc.) — **never** from the GitHub webhook payload.
- `environmentName` is **not** stored on any MongoDB document. The env is implicit via the env-scoped MongoDB connection, so persisting it would just duplicate that value on every record. The Backend Service uses its config-supplied env name only when constructing the workspace name.

Because the four MongoDB instances are independent, there is no global view of "all Postman resources for a tenant" within the database — the global view exists only in Postman (one workspace per env). For audit, query each env's MongoDB separately.

### 8.11 Webhook Payload Structure

The existing GitHub webhook listener service emits payloads in the following shape (other fields may also be present and can be ignored):

```json
{
  "tenantName": "DNA",
  "branchName": "develop",
  "added":    ["reference/11.0.0/Accountholder/PersonService-11.0.0_DNA.yaml"],
  "removed":  ["reference/10.9.0/Accountholder/AddressService-10.9.0_DNA.yaml"],
  "modified": ["reference/11.0.0/Accountholder/AddressService-11.0.0_DNA.yaml"]
}
```

**File path convention:** every path follows `reference/{version}/<arbitrary intermediate directories>/{fileName}`. A single tenant + version can therefore contain many different YAML files (one per product or service under that version's directory). See §4.0.1 for the parsing rules used to derive `version`, `fileName`, and `pathFromVersion`.

The Backend Service iterates over each path in each array independently — one webhook event can therefore trigger several scenario executions in §4.
