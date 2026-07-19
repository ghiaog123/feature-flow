# Standard format for the API Contract Markdown

**Mandatory** template for the output of the `ff-api-contract-writer` skill. Copy the structure and fill it in — do not invent a different structure. The example below is written in English (the default output language); if the user requests another language, translate the descriptions/sub-headings ("Table of Contents", etc.) but keep the structure unchanged.

---

## Full template

```markdown
# API Contract — <Service Name>

> **Base URL**: `http://localhost:<port>` (placeholder — replace per environment)
> **Authentication**: <default auth, e.g.: API Key via header `X-API-Key`, applies to every endpoint unless noted otherwise>

## Table of Contents

| # | Method | Path | Short description |
|---|--------|------|-------------------|
| 1 | `POST` | [`/api/v1/items`](#1-post-apiv1items) | Create an item and enqueue it into the background processing pipeline |
| 2 | `GET`  | [`/api/v1/items/{item_id}`](#2-get-apiv1itemsitem_id) | Get an item's status |

---

## 1. `POST /api/v1/items`

**Description**: Create (or update) an item and enqueue it for background processing. Idempotent per `(user_id, item_id)` — if a previous processing job is still running, returns that job's info (`dedup_hit=true`) instead of creating a new one.

### Request

#### Body (`application/json`)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `item_id` | `string` | ✅ | — | Item ID (client-provided) |
| `user_id` | `string` | ✅ | — | ID of the owning user |
| `name` | `string` | ✅ | — | Display name |
| `group_id` | `string` | ❌ | `"default"` | Group containing the item. Length 1–128 characters |

#### Example

```bash
curl -X POST "http://localhost:8000/api/v1/items" \
  -H "X-API-Key: <your-key>" \
  -H "Content-Type: application/json" \
  -d '{"item_id": "it_001", "user_id": "u_123", "name": "Item A"}'
```

### Response

#### `202 Accepted` — Accepted and enqueued (or dedup hit)

```json
{
  "item_id": "it_001",
  "status": "pending",
  "job_id": "trace_abc123",
  "dedup_hit": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | `string` | `pending` / `processing` / `ready` / `failed` |
| `job_id` | `string` | Job tracking ID. Dedup hit → keeps the running job's `job_id` |
| `dedup_hit` | `boolean` | `true` when a previous job is still running and no new job was created |

#### `409 Conflict` — A concurrent write request for the same item is in progress

```json
{
  "detail": "Concurrent upsert in progress for this item, retry shortly"
}
```

---

## 2. `GET /api/v1/items/{item_id}`

**Description**: Get the current status of an item.

### Request

#### Path params

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `item_id` | `string` | ✅ | Item ID |

#### Query params

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `user_id` | `string` | ✅ | — | ID of the owner (query scope) |

#### Example

```bash
curl -X GET "http://localhost:8000/api/v1/items/it_001?user_id=u_123" \
  -H "X-API-Key: <your-key>"
```

### Response

#### `200 OK`

```json
{
  "item_id": "it_001",
  "status": "ready",
  "error": null,
  "updated_at": "2026-01-01T10:00:30Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | `string` | `pending` / `processing` / `ready` / `failed` |
| `error` | `string \| null` | Error message when `status=failed`, otherwise `null` |
| `updated_at` | `string (ISO 8601)` | Time of the most recent update |

#### `404 Not Found`

```json
{
  "detail": "Item not found"
}
```

---

## Appendix — Common errors

Apply to every endpoint (unless noted otherwise); do not repeat them per endpoint:

| Status | When | Shape |
|--------|------|-------|
| `401 Unauthorized` | Missing or invalid `X-API-Key` header | `{"detail": "Invalid API key"}` |
| `422 Unprocessable Entity` | Body/query/path fails schema validation | `{"detail": [{"type": "...", "loc": [...], "msg": "...", "input": ...}]}` |
| `500 Internal Server Error` | Unhandled error in the handler | — |
```

---

## Presentation rules

1. **Endpoint heading**: `## <number>. \`<METHOD> <path>\`` — the sequence number is for table-of-contents links.
2. **"Required" column**: only `✅` / `❌`.
3. **Empty sections**: endpoint with no Path params / Query params / Body / dedicated Headers → **omit the section entirely**, no placeholder. The common auth header declared at the top of the file → do not repeat a Headers table per endpoint.
4. **Status codes**: one sub-heading per status `#### \`<code>\` <reason> — <short condition>`. Only list errors **specific to the endpoint** (endpoint-specific 404, 409, 422 business errors with their concrete conditions); common errors → Appendix.
5. **Response field table**: only add it when fields need explanation (enum values, null conditions, non-obvious meaning). If the example JSON is self-explanatory → drop the table. Obvious fields (`item_id` = "item ID") don't need their own row if the whole table would consist only of such rows.
6. **JSON examples**: must be parseable. Undetermined values → clear placeholder `"<item_id>"`.
7. **Descriptions**: 1–3 sentences, caller-observable behavior. No internal technology mentions (DB, queue, cache, lock).
8. **Uncertain inferred fields**: note `_(inferred from code, needs confirmation)_` right after the description.
