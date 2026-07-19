---
name: ff-api-contract-writer
description: "Read the source code of a repo/service and write a concise Markdown API contract containing only what a caller needs to know. Use this skill whenever the user calls `/ff-api-contract-writer`, or says phrases like 'write api docs', 'create an api contract', 'document endpoints', 'list the endpoints in service X' — even when the user doesn't explicitly say the word 'skill'. Operates semi-automatically — lists the endpoints found, waits for the user to confirm scope, and only then generates the detailed contract."
---

# API Contract Writer

Create API contract documentation for a service/module; the output is a Markdown file following the format in `references/contract-format.md`.

**Philosophy**: the contract describes what the **caller can observe** — request shape, response shape, status codes, error conditions. It is not a system design document. The reader is a dev integrating with the API, not a dev maintaining the service.

## Language

Default to English (descriptions, notes, field names, paths). If the user requests another language (or is conversing in another language), write descriptions/notes in that language; field names/paths/code stay in English.

## Semi-automatic workflow (follow this exact order)

The user chose this workflow because they want to **confirm scope before detailed writing** — do not generate a contract for the whole repo on your own.

### Step 1 — Clarify scope

If the user hasn't specified the service/module/folder, ask briefly:
- Which service/module?
- Restrict to a specific router/file?
- Where should the output file go? (default suggestion: `<service>/docs/api-contract.md`)

If the user has already specified, skip this.

### Step 2 — Scan and list endpoints

Find all HTTP endpoints in scope. Detect by framework:

- **FastAPI / Starlette**: `@app.<method>`, `@router.<method>` (`get/post/put/patch/delete`), combined with the router's `prefix=` to build the full path.
- **Flask**: `@app.route(..., methods=[...])`, `@blueprint.route(...)`.
- **Express**: `app.<method>()`, `router.<method>()`.
- **Spring Boot**: `@GetMapping`, `@PostMapping`, `@RequestMapping`.
- **NestJS**: `@Get/@Post/@Put/@Delete` in controllers.
- Repo has `openapi.json` / `openapi.yaml` → prefer reading that file.

Present a summary table:

```
Found N endpoints in <scope>:

| # | Method | Path                    | Handler          |
|---|--------|-------------------------|------------------|
| 1 | POST   | /api/v1/items           | items.create     |
| ...

Do you want the contract for all of them, or only some endpoints?
```

**STOP and wait for user confirmation.** Do not continue writing on your own.

### Step 3 — Read details and write the contract

For each confirmed endpoint:

1. Read the handler: parameters, request/response models (Pydantic/DTO), exceptions, auth dependencies.
2. Trace back any referenced schemas.
3. Detect auth (dependency, middleware, decorator).
4. Detect error responses that can actually be returned (`HTTPException`, exception handlers).

Write following the format in `references/contract-format.md` — do not invent a different structure.

### Step 4 — Save and report

Write the file to the agreed path. Report: number of endpoints documented + anything that could not be inferred from the code (for the user to fill in manually).

## Content principles

**Contract, not implementation.** Descriptions contain only behavior the caller can see: idempotency, conditions for each status code, default values, limits (max items, lengths). Do NOT mention internal technology names (DB type, message queue, cache, lock key names, background job names) — the caller doesn't need them and it leaks system details. Example:

- ❌ "Uses a Redis lock to prevent concurrent upserts, enqueues ARQ job `ingest_file`"
- ✅ "Concurrent upsert with the same `(user_id, file_id)` → `409`. If a previous processing job is still running, returns that job's info (`dedup_hit=true`)"

**No filler.**
- Endpoint has no path params / query params / body → **omit that section entirely**, don't write a "None" placeholder.
- Errors common to the whole service (wrong API key, the framework's default schema validation) → document them **once in the Appendix** at the end of the file. Each endpoint only lists its own specific errors (endpoint-specific 404, 409, 422 business errors).
- Endpoint description: 1–3 sentences. Enough to understand when to call it and what comes back.

**Infer from code, don't invent.** If undeterminable:
- Field type → write `unknown` + note `_(needs confirmation)_`.
- Status code → only list what the code actually raises/returns.
- Business description → read docstrings/comments; if none, describe briefly from the handler name, don't make it up.

**Prefer real schemas.** Body is a model class → read the model, list every field with type, required, default, description.

**Generic.** Do not put company/organization names, real internal URLs, or environment-specific information into the contract — the base URL uses a placeholder (`http://localhost:<port>` or `https://api.example.com`) unless the user requests otherwise.

**Inherited auth.** Auth applied at the router level → state it once in the file header ("All endpoints require ... unless noted otherwise"); individual endpoints only mention auth when it **differs from the default**.

## Output format

Read `references/contract-format.md` before writing. Summary:

1. **Header**: service name, base URL (placeholder), common auth.
2. **Table of contents**: Method / Path / short description table, with anchor links.
3. **Each endpoint**: heading `## <number>. \`<METHOD> <path>\`` → Description → Request (only tables with content) → curl example → Response (one sub-section per status, schema table when fields need explanation) → endpoint-specific errors.
4. **Appendix**: table of service-wide common errors.

The "Required" column uses ✅/❌.
