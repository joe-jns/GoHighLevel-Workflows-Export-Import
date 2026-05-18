# GoHighLevel Workflow API — Reconnaissance

Source: live capture against `app.business-toolbox.com` (white-label GHL), tenant
`locationId=vQ2IFiUBooR3e9F6hl1f`, 2026-05-18.

## Architecture

- White-label wrapper page: `https://app.business-toolbox.com/v2/location/{locationId}/automation/workflows`
- Inside it, an `<iframe name="workflow-builder">` loads the real workflows SPA at
  `https://client-app-automation-workflows.leadconnectorhq.com/...`
- The iframe makes all API calls to `https://backend.leadconnectorhq.com`.
- The wrapper and iframe are cross-origin, so direct DOM access between them is blocked.

## Auth

- Header `token-id: <Firebase JWT>` on every API request.
- Header `channel: APP`.
- Token is stored in the **iframe origin's** localStorage as `refreshedToken` (auto-refreshed by GHL every ~hour).
- A content script running in the iframe origin can read it directly.

## Endpoints

### 1. List folders + workflows (paginated)

```
GET https://backend.leadconnectorhq.com/workflow/{locationId}/list
    ?parentId={folderId|root}
    &limit=10&offset=0
    &sortBy=name&sortOrder=asc
    &includeCustomObjects=true&includeObjectiveBuilder=true
```

Variants observed:
- `parentId=root` → top-level folders
- `parentId={folderId}` → workflows inside a folder
- `type=workflow` → flat list of all workflows (no folder filter)

Response shape:
```json
{
  "rows": [
    {
      "_id": "fb7e5d48-...",
      "id": "fb7e5d48-...",
      "name": "AUTO | ...",
      "type": "workflow" | "directory",
      "status": "published" | "draft",
      "parentId": "f8511591-..." | null,
      "locationId": "...",
      "dataVersion": 7,
      "createdAt": "...",
      "updatedAt": "..."
    }
  ],
  "count": N,
  "isLocationRateLimited": false
}
```

### 2. Workflow detail (includes all actions/steps)

```
GET https://backend.leadconnectorhq.com/workflow/{locationId}/{workflowId}
    ?includeScheduledPauseInfo=true
    &sessionId={uuid}
```

Response shape:
```json
{
  "_id": "fb7e5d48-...",
  "name": "AUTO | Chatbot — Entrée pipeline nouveau lead",
  "status": "published",
  "version": 10,
  "type": "workflow",
  "parentId": "f8511591-...",
  "locationId": "...",
  "filePath": "location/.../workflows/.../10",
  "fileUrl": "https://firebasestorage.googleapis.com/...?token=...",
  "triggersFilePath": "location/.../workflow-triggers/.../10",
  "workflowData": {
    "templates": [
      {
        "id": "a548275e-...",
        "order": 0,
        "name": "Internal Notification",
        "type": "internal_notification",
        "attributes": {
          "type": "notification",
          "notification": {
            "type": "send_notification",
            "body": "...",
            "title": "...",
            "redirectPage": "opportunity",
            "userType": "all"
          }
        },
        "next": "01624041-..."
      },
      {
        "id": "01624041-...",
        "order": 1,
        "name": "Create Opportunity",
        "type": "internal_create_opportunity",
        "attributes": { "pipelineId": "...", "__customInputFields__": [...] },
        "parentKey": "a548275e-...",
        "next": "3046d8cd-..."
      }
    ]
  }
}
```

- `workflowData.templates[]` is a linked structure: each step has `next` (and optionally `parentKey` for branches).
- `attributes` is type-specific (the action's config).
- `fileUrl` is a Firebase Storage signed URL pointing to the full snapshot incl. canvas positions. Not needed for JSON export.

### 3. Workflow triggers

```
GET https://backend.leadconnectorhq.com/workflow/{locationId}/trigger?workflowId={workflowId}
```

Response shape:
```json
[
  {
    "id": "W83dPI5CyYKtbIYClAMQ",
    "name": "Contact Changed",
    "type": "contact_changed",
    "masterType": "highlevel",
    "workflow_id": "...",
    "location_id": "...",
    "belongs_to": "workflow",
    "active": true,
    "conditions": [
      { "operator": "has-changed", "field": "contact.email", "title": "Email", "type": "input" }
    ],
    "actions": [{ "workflow_id": "...", "type": "add_to_workflow" }],
    "schedule_config": {},
    "date_added": "...",
    "date_updated": "..."
  }
]
```

### 4. Sticky notes (workflow canvas annotations — optional)

```
GET https://backend.leadconnectorhq.com/workflows/sticky-notes-all
    ?workflowId={workflowId}&locationId={locationId}
```

## White-label detection

URL-based detection won't work (white-label domain varies).
Stable signals to detect "we're on a GHL workflows page":

1. Iframe with `name="workflow-builder"` whose `src` starts with `https://client-app-automation-workflows.leadconnectorhq.com/`
2. URL path matches `/(v2/)?location/[^/]+/automation/workflows` OR `/location/[^/]+/workflow/[^/]+` (the latter is the builder route the wrapper rewrites to)
3. Inside the iframe origin: the SPA loads at `client-app-automation-workflows.leadconnectorhq.com` — content scripts can match on this host directly.

## Write endpoints (for import)

Captured 2026-05-19 against the same tenant.

### A. Create new workflow

```
POST https://backend.leadconnectorhq.com/workflow/{locationId}
```

Body:
```json
{
  "name": "New Workflow : 1779143322341",
  "status": "draft",
  "parentId": null,
  "updatedBy": "{userId}",
  "modifiedSteps": [],
  "deletedSteps": [],
  "createdSteps": [],
  "senderAddress": {},
  "stopOnResponse": false,
  "allowMultiple": true,
  "allowMultipleOpportunity": true,
  "autoMarkAsRead": false,
  "eventStartDate": "",
  "timezone": "",
  "workflowData": { "templates": [] },
  "triggersChanged": false,
  "company_id": "{companyId}",
  "company_age": 26
}
```
Response: `{ "id": "<new-workflow-uuid>" }`.

### B. Duplicate existing workflow (server-side clone, same tenant only)

Same URL, different body shape:
```json
{
  "new_workflow_name": "Copy - X",
  "workflow_id": "<source-workflow-id>",
  "parentId": null,
  "company_id": "{companyId}",
  "company_age": 26
}
```

### C. Save / update workflow steps

```
PUT https://backend.leadconnectorhq.com/workflow/{locationId}/{workflowId}/auto-save
```

Body is the **full workflow snapshot** plus diff arrays:
```json
{
  "_id": "{workflowId}",
  "locationId": "...", "companyId": "...", "companyAge": 26,
  "name": "...", "status": "draft", "version": N, "dataVersion": 7,
  "id": "{workflowId}",

  "workflowData": {
    "templates": [
      { "id": "<step-uuid>", "order": 0, "name": "Add Tag",
        "type": "add_contact_tag", "attributes": { "tags": ["lead_froid"] } }
    ]
  },

  "modifiedSteps": [],
  "deletedSteps": [],
  "createdSteps": ["<step-uuid>"],
  "triggersChanged": false,
  "oldTriggers": [ /* previous trigger array */ ],
  "newTriggers": [ /* current trigger array, same shape as GET /trigger */ ],
  "isAutoSave": true,
  "autoSaveSession": { "workflowId": "...", "id": "<uuid>", "userId": "...", "version": 1 }
}
```

Key fields:
- `createdSteps[]` / `modifiedSteps[]` / `deletedSteps[]` — arrays of step IDs that changed; the server uses these as a diff.
- `workflowData.templates[]` — the FULL current state (not a diff).
- `triggersChanged: true` when triggers were added/removed.
- `oldTriggers` / `newTriggers` — both arrays, server reconciles.
- `version` and `__v` — optimistic-concurrency tokens; preserve from previous GET.

### D. Create trigger

```
POST https://backend.leadconnectorhq.com/workflow/{locationId}/trigger
```

Body:
```json
{
  "status": "draft",
  "workflowId": "{workflowId}",
  "schedule_config": {},
  "conditions": [],
  "type": "contact_created",
  "masterType": "highlevel",
  "name": "Contact Created",
  "actions": [{ "workflow_id": "{workflowId}", "type": "add_to_workflow" }],
  "active": true,
  "triggersChanged": true,
  "location_id": "{locationId}",
  "company_id": "{companyId}",
  "company_age": 26
}
```
Response: `{ "id": "<trigger-id>" }`.

### Import recipe (derived)

1. `POST /workflow/{locId}` with empty `workflowData.templates`, get `newId`.
2. `GET /workflow/{locId}/{newId}` to obtain the fresh envelope (with correct
   `version`, `__v`, `filePath`, `fileUrl`).
3. For each trigger in the import payload: `POST /workflow/{locId}/trigger`
   with `workflowId`, `actions[].workflow_id`, `location_id` rewritten to the
   new workflow. Collect returned IDs.
4. `PUT /workflow/{locId}/{newId}/auto-save` using the envelope from step 2 plus:
   - `workflowData.templates[]` = imported steps (preserve their original IDs).
   - `createdSteps[]` = every step ID in templates.
   - `newTriggers[]` = the triggers from step 3 (with their server-assigned IDs).
   - `oldTriggers[]` = `[]`.
   - `triggersChanged: true`.
   - `isAutoSave: true`, generate fresh `autoSaveSession` UUID.

Open risks (need real-world testing):
- Whether step IDs in `next` / `parentKey` pointers survive as-is or must be regenerated.
- Whether step-internal IDs referenced inside `attributes` (e.g., `pipelineId`,
  `pipelineStageId`, `customField` IDs, calendar IDs, user IDs) must exist in
  the target tenant. **They do** — for cross-tenant imports, either rewrite
  these or accept the workflow will reference dangling targets (the steps
  themselves still save; they just won't function until remapped).
- Whether triggers must be created *after* `auto-save` or *before*. We chose
  before so the trigger IDs are known and can be embedded in `newTriggers[]`.

## Implications for the extension

- The most reliable injection point is **inside the iframe origin** (`client-app-automation-workflows.leadconnectorhq.com`). The content script can:
  - Read the auth token from `localStorage.refreshedToken`
  - Detect "we're on workflows" via URL path
  - Inject a button into the SPA's DOM
  - Call the 3 read endpoints directly to bulk-export
  - Call the 4 write endpoints to import
- No need to scrape the DOM for workflow definitions — the API returns clean JSON.
- Triggers and actions both come from documented endpoints; no reverse-engineering needed.
- The user's existing JWT (`token-id` from localStorage) has write scope when
  `workflows_read_only: false` in the token payload — verified for this user.
