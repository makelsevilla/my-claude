---
name: n8n-workflow
description: Build, update, test, publish, and diagnose n8n workflows using the n8n MCP SDK. Use this skill whenever the user wants to create a new workflow, add or fix nodes, update an existing workflow, test with pin data, publish or unpublish, debug a failed execution, or asks about n8n node parameters or SDK patterns. Trigger on phrases like "create a workflow", "add a node", "fix this node", "publish it", "test it", "why did this workflow fail", "build me an automation", or any mention of n8n node types, triggers, or workflow design — even if the user doesn't say "n8n" explicitly when the context is already an n8n session.
---

# n8n Workflow Skill

Standards and process for building n8n workflows via the MCP SDK. Follow this every time — do not rely on memorised node parameter names or SDK shapes; they change between versions.

---

## Process: Research → Plan → Implement

Never jump straight to building. Always do these three steps in order.

### 1. Research

Before touching SDK code, ground yourself in current syntax using the n8n MCP tools. The tools are deferred — call `ToolSearch` first to load their schemas.

**Required tool call order (run discovery steps in parallel where independent):**

| Step | Tool | When |
|------|------|------|
| 1 | `get_sdk_reference` | Always — load patterns, expressions, connection syntax before writing any code |
| 2 | `get_suggested_nodes` | Pass all relevant technique categories — use recommendations to pick nodes |
| 3 | `search_nodes` | Search by service name + utility nodes you'll use |
| 4 | `get_node_types` | **Source of truth** — get TypeScript defs for every node ID you plan to use, including discriminators from search results. Never skip this. |

### 2. Plan

**Stop here. Do not write SDK code yet.**

Write out and present to the user:
- **Trigger**: what starts this workflow
- **Data flow**: each node and what it does to the data, in order
- **Nodes**: which you'll use and why
- **Failure modes**: what can go wrong and how you'll handle it

Then **wait for the user to confirm the plan** before moving to step 3. If the spec is ambiguous, ask one focused question instead of assuming. A plan takes 30 seconds to confirm; a wrong workflow takes much longer to fix.

### 3. Implement

```
validate_workflow → fix all errors AND warnings
→ create_workflow_from_code (new) or update_workflow (existing)
→ prepare_test_pin_data + test_workflow
→ publish_workflow
```

Never publish a workflow that hasn't run end-to-end at least once with pinned data.

---

## Naming Conventions

- **Workflow name**: `[Domain] - [Action] - [Trigger]`
  - e.g. `Billing - Send invoice reminders - Daily 9am`
  - e.g. `Leads - Sync new contacts - Webhook`
- **Node names**: verb-first, describe what it does — not the integration
  - `Fetch overdue invoices` not `MySQL`
  - `Forward to CRM` not `HTTP Request`
- **Variables/credential names**: `snake_case`, no spaces or special chars

---

## Workflow Design Rules

**Modularity**
- 5–15 nodes per workflow. If it grows beyond that, split into a sub-workflow via Execute Workflow.
- One workflow = one responsibility.

**Connections**
- IF nodes: wire BOTH branches (true and false). Never leave a branch dangling — add a Stop and Error or explicit handler.
- Switch nodes: always add a fallback output for unmatched cases.

**Idempotency**
- Any workflow that writes to an external system must be safe to re-run.
- Use idempotency keys, upserts, or an "already processed" check at the top.

**Pagination & batching**
- Never load large datasets in one node. Use Split In Batches (start at 50, tune from there).

---

## Error Handling (Non-Negotiable)

- Every workflow touching an external API gets either:
  - An **Error Trigger workflow** wired up, OR
  - `onError: 'continueErrorOutput'` with an explicit error branch downstream
- HTTP Request nodes: enable retry (3 attempts, exponential backoff) for flaky external services.
- Log failures to a known destination (Slack, error table, or Sentry — pick one per project).
- **Never silently swallow errors.** "Continue on fail" without a downstream branch is a silent swallow.

---

## Credentials & Secrets

- Never hardcode API keys, tokens, or secrets in node parameters or Code nodes.
- Reference credentials by name from n8n's credential manager only.
- **When updating an existing workflow**: omit the `credentials` block entirely for nodes the user has already wired up — do not overwrite them.
- Webhook URLs containing secret paths count as secrets; never paste them into chat or commit them to git.

---

## Webhook Node — Data Shape

The Webhook node v2.1 wraps the incoming request:

```
$json = {
  headers: { ... },
  query:   { ... },
  params:  { ... },
  body:    { ...actual payload... }
}
```

Always use `$json.body` (not `$json`) to access the request payload in downstream nodes.

---

## HTTP Request Node — Body Formatting

When sending a JSON body:
- `specifyBody: 'json'`  ← mode selector only — values: `'keypair'` | `'json'`
- Put the body content in `jsonBody`, not `specifyBody`

**Never put an expression in `specifyBody`.** It is a dropdown, not a content field.

**Never wrap a whole JSON object in one expression for `jsonBody`.** Interpolate per-property with `={{ }}` on each dynamic field value — static values stay as plain JSON literals:

```js
// WRONG — single expression wrapping the whole object
jsonBody: expr('={{ { "model": "voyage-4-large", "input": [$json.name] } }}')

// CORRECT — static values as literals, only dynamic values use ={{ }}
jsonBody:
  '{\n' +
  '  "model": "voyage-4-large",\n' +
  '  "input": [={{ $json.name }}]\n' +
  '}'
```

**Interpolating a value that is already an array (e.g. an embedding vector) — wrap it in literal `[ ]` around a bare `{{ }}`.** When you interpolate an array with `{{ }}`, n8n renders its elements comma-joined *without* the surrounding brackets, so you must supply them yourself. This is the opposite of the string case above: a *string* going into an array uses `[={{ $json.name }}]` (n8n quotes it → `["text"]`), but an *array* value uses `[{{ ... }}]`.

```js
// WRONG — `={{ array }}` renders the elements without the surrounding [ ] brackets
'  "query_embedding": ={{ $(\'Get Embedding\').item.json.data[0].embedding }},\n' +

// CORRECT — literal [ ] around a bare {{ }} → "query_embedding": [0.1, 0.2, ...]
'  "query_embedding": [{{ $(\'Get Embedding\').item.json.data[0].embedding }}],\n' +
```

**Always write `jsonBody` as pretty-printed JSON** — one key per line, indented with `\n` and spaces. A minified one-liner is valid but unreadable in the n8n editor and hard to diff. The `\n` approach (string concatenation or escaped newlines) is required because the SDK disallows backtick template literals.

The `contentType: 'json'` setting handles serialization automatically — you pass an object, n8n sends JSON. No need to call `.toJsonString()`.

---

## Code Node Guidelines

- Prefer standard nodes over Code nodes — standard nodes survive upgrades better.
- Use Code nodes only for: transformations that need 3+ chained Set nodes, custom date math, or logic that genuinely doesn't fit a node.
- Default to JavaScript. Always handle the `items` array shape.

```js
return items.map(item => {
  try {
    const v = item.json;
    return { json: { ok: true, result: doWork(v) } };
  } catch (e) {
    return { json: { ok: false, error: e.message, input: item.json } };
  }
});
```

---

## Safety Rules

- **Never edit production workflows directly.** Duplicate first, edit the copy, validate, test, then swap.
- Before any `update_workflow` or `archive_workflow` on a workflow Claude didn't just create: call `get_workflow_details` first, then confirm the target with the user.
- Destructive operations (archive, delete data table rows, unpublish) require explicit user confirmation — do not infer permission from the original task.
- If a tool returns an unexpected schema or empty result, stop and re-check parameter names via `ToolSearch` before retrying with guesses.

---

## Things That Have Burned Us

- **`responseFormat` on HTTP Request** defaults to `autodetect` — set it explicitly to `json` for JSON APIs or you'll occasionally get a string back and downstream nodes will break.
- **`specifyBody` is not the body field** — it's the mode selector. Putting an expression there causes "JSON payload not supported" errors. Expression goes in `jsonBody`.
- **Interpolating an array into `jsonBody` drops its brackets.** `{{ someArray }}` renders as comma-joined elements with no surrounding `[ ]` — wrap it yourself: `"query_embedding": [{{ ...embedding }}]`. Writing `={{ array }}` (bare, unbracketed) produces invalid JSON. Contrast a *string* into an array, which is `[={{ $json.text }}]`.
- **`$json` vs `$json.body` on Webhook v2.1** — the body is nested. Using `$json` forwards the whole request envelope (headers, query, params, body) instead of the payload.
- **Don't overwrite credentials on update** — omit the `credentials` block when updating nodes the user has already wired up in the n8n UI. Re-specifying credentials replaces them with blank `newCredential()` references.
- **SDK fan-out: `.to(A).to(B)` chains from A, not from the source node.** If one node must branch into two independent parallel paths, use `.add(sourceNode)` before each branch — NOT chained `.to()` calls. Chaining silently wires the end of branch A into the start of branch B.
  ```js
  // WRONG — secretly wires setTier → cleanHtml
  extractJobData
    .to(determineTier.to(setTier))
    .to(cleanHtml.to(...))

  // CORRECT — both branches fan out independently from extractJobData
  .add(extractJobData).to(determineTier.to(setTier))
  .add(extractJobData).to(cleanHtml.to(...))
  ```

---

## When You're Stuck

1. Re-read `get_sdk_reference` for the specific area (expressions, connections, triggers).
2. Check `search_executions` for a similar workflow that worked.
3. Tell the user what you tried and what the actual error was. Don't loop on retries silently.
