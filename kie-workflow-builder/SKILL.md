---
name: kie-workflow-builder
description: Build n8n sub-workflows from kie.ai model documentation URLs. Use this skill whenever the user provides a kie.ai docs URL and wants to create or duplicate an n8n workflow for that API — including phrases like "create a workflow from this doc", "duplicate this workflow for X model", "implement this API in n8n", or "add a new kie.ai model". Also trigger when the user mentions integrating any kie.ai model (seedream, kling, sora, veo, grok, wan, bytedance, nano-banana, z-image, etc.) into n8n.
---

# Kie.ai → n8n Workflow Builder

This skill builds n8n sub-workflows that call the kie.ai API. Given a model documentation URL, it extracts the payload schema, builds a Zod validator inside a Super Code node, wires up the standard create-task → poll-loop → return pipeline, and outputs a ready-to-use workflow.

## ⚠️ Tool Compatibility (n8n-mcp czlonkowski v2.57.1)

The `n8n_update_partial_workflow` tool is **fully broken** on this version — ALL operations (addNode, addConnection, updateNode, patchNodeField, activateWorkflow, etc.) fail with `"request/body must NOT have additional properties"`. This is a known issue with the MCP server's REST client validation layer. Do not waste time debugging it.

| Tool | Status | Use for |
|------|--------|---------|
| `n8n_create_workflow` | ✅ Works | Cloning / initial creation. Requires `name`, `nodes[]`, `connections{}`. Created inactive. |
| `n8n_update_full_workflow` | ✅ Works | **ALL** post-creation changes. Requires `id`, `name`, `nodes[]`, `connections{}`. |
| `n8n_get_workflow` | ✅ Works | Fetching source workflow for duplication. Use `mode: "full"`. |
| `n8n_delete_workflow` | ✅ Works | Cleanup. |
| `n8n_validate_workflow` | ✅ Works | Validation with errors/warnings/suggestions. |
| `n8n_test_workflow` | ✅ Works | Triggering workflows with `triggerType: "webhook"`. Requires workflow to be active. |
| `n8n_executions` | ✅ Works | Listing/getting/deleting execution records. Use `mode: "error"` to debug failures. |
| `n8n_autofix_workflow` | ⚠️ Preview only | Preview mode works. `applyFixes: true` fails (uses partial-update internally). Apply fixes manually. |
| `n8n_update_partial_workflow` | ❌ Broken | **NEVER USE.** Every operation type is blocked by the validation layer. |

## Workflow Creation Process

### Step 1: Fetch and Parse Documentation

1. Fetch the documentation page from the provided URL (use `crawlSinglePage` or equivalent)
2. Extract the following from the docs:
   - **Model identifier** — the `model` string (e.g. `grok-imagine-video-1-5-preview`)
   - **Input parameters** — name, type, required/optional, options/enum values, defaults, max constraints
   - **Output type** — image, video, or text (determined from `resultUrls` context in the response example and model name suffixes)
   - **Note any `nsfw_checker` parameter** — must be excluded from the schema (see Critical Rules)

### Step 2: Determine Model Category

Infer from the model name what the output is:

| Suffix | Output | Return field name | "Get Generated" node name |
|--------|--------|-------------------|--------------------------|
| T2I / I2I / image-to-image | Image | `generated_image` | Get Generated Image |
| T2V / I2V / V2V / image-to-video | Video | `generated_video` | Get Generated Video |
| text / other | Text (via `resultObject`) | `generated_text` | Get Generated Result |

### Step 3: Build the Zod Schema

Read the parameters table from the docs and construct a Zod schema inside the Super Code node. This is the **only** schema content that changes — all helper functions below the `// ####` divider stay verbatim.

**Schema structure:**
```javascript
const Schema = z.object({
  model: z.string().default("<model-identifier>"),
  input: z.object({
    // ... one field per input parameter from docs
  })
})
```

**Field rules:**
- `prompt` (string): always include, use `max()` from docs, make `.optional()` only if docs say "Required: No"
- `image_urls` (array of URLs): when docs say "Required: Yes", use `z.array(z.string().url())` without `.optional()`. Do NOT add `.max(N)` unless the docs explicitly specify a maximum count — don't carry over constraints from the source workflow's model
- Enum fields: use `z.enum([...])` with exact values from docs; include `.default()` with the docs default (not the source workflow's default)
- Number fields with range: use `z.number().int().min(X).max(Y).default(Z)` — use the docs defaults and ranges, not the source workflow's
- Boolean fields: use `z.boolean().default(value)` if docs provide a default
- Optional fields: append `.optional()` when docs say "Required: No"
- **ALWAYS exclude `nsfw_checker`** — never include this parameter in the schema or payload, even if present in the docs

**Refinements** — Only add `.refine()` when there are genuine cross-field dependencies visible in the docs (e.g., "cannot use both X and Y simultaneously"). Do not invent refinements that aren't in the docs. If the new model has no cross-field constraints, **remove all `.refine()` calls** that were in the source schema — leaving stale refinements from a different model will break validation.

**Final formatter** — The Super Code node always transforms the validated data into the API format:
```javascript
(data) => {
  const {model, ...rest} = data
  return {model, input: rest}
}
```

### Step 4: Duplicate Source Workflow

**MANDATORY**: Always duplicate from an existing kie.ai workflow. Never build node-by-node from scratch. Duplicating preserves all subtle configuration details (Switch rules with condition IDs, HTTP Request options, authentication setup, etc.).

**4a. Pick a source workflow:**
1. Use `n8n_list_workflows` to see available kie.ai workflows
2. Pick the best match: same provider and same output type (I2V for I2V, T2I for T2I, etc.)
3. The closest source ensures the fewest changes needed

**4b. Clone the source:**
1. Use `n8n_get_workflow` on the source with `mode: "full"` to get the complete JSON
2. Create the clone using `n8n_create_workflow` with:
   - New `name` (see Step 7 naming convention)
   - Same `nodes[]` — **node `id` fields are required by `create_workflow`, preserve them all**
   - Same `connections{}`
   - Same `settings`

The result is a 100% identical clone except for the name. It will be created inactive.

**4c. Modify the clone — what to change:**
Use `n8n_update_full_workflow` in a **single call** to change only these five things:

| # | Node | Field | What to change |
|---|------|-------|---------------|
| 1 | **Start** | `parameters.workflowInputs.values` | Replace with new inputs: `prompt` first, then required fields, then optional. Remove old fields (`mode`, `index`, `task_id` etc). Fix duplicates (the grok I2V source had two `resolution` entries). Set `"type": "array"` on array fields. |
| 2 | **Super Code** | `parameters.code` | Replace the Zod schema and model identifier only. Keep all helper functions (`getDefaults`, `setDefaults`, `processData`) and execution mode logic verbatim. |
| 3 | **Return** | `parameters.assignments.assignments[0]` | Update `name` if output type changed (`generated_image` ↔ `generated_video` ↔ `generated_text`). Update `type` too for text models (`"stringValue"`). |
| 4 | **Get Generated [Media]** | `name` | Rename to match output type (Get Generated Image / Get Generated Video / Get Generated Result). Also update the connection key in `connections{}` to match. |
| 5 | **Sticky Note** | `parameters.content` | Update docs URL to `## Docs\nhttps://kie.ai/<provider>?model=<url-encoded-model-identifier>` |

**4d. What to NEVER change — preserve exactly from source:**
- All HTTP Request node configurations (`authentication`, `genericAuthType`, URLs, query parameters, body settings)
- All Switch node rules and conditions — including the exact condition IDs (`c38b2807-2d59-...`, `db04a8bf-1bea-...`, etc.)
- All credential references: `"httpBearerAuth": {"id": "gv9noTXHGc9OkhZy", "name": "Kie.ai Token"}`
- The credential key MUST be `httpBearerAuth`, not `genericAuthType`
- All connections (reference by node `name`, not `id`)
- Wait node: `await new Promise(resolve => setTimeout(resolve, 5000));`
- Loop Over Items: `"reset": false`
- Stop and Error nodes: error message expressions
- No Operation node
- All node positions

**When calling `n8n_update_full_workflow`:**
- Always include: `id`, `name`, `nodes[]` (full array, ALL nodes), `connections{}` (full object)
- Only the fields listed in 4c should differ from the clone
- If `n8n_autofix_workflow` preview showed typeVersion upgrades needed, apply them in this same call (change `typeVersion: 4.3` → `typeVersion: 4.4` on HTTP Request nodes)

### Step 5: Set the Start Node Inputs

Map each Zod `input` field to a workflow input:

```json
{
  "parameters": {
    "workflowInputs": {
      "values": [
        {"name": "prompt"},
        {"name": "image_urls", "type": "array"},
        {"name": "aspect_ratio"},
        {"name": "resolution"},
        {"name": "duration"}
      ]
    }
  }
}
```

Order: `prompt` first, then required fields, then optional fields. Set `type: "array"` on array fields. Do NOT include `nsfw_checker`. Remove any leftover inputs from the source that don't exist in the new model.

### Step 6: Build the Return Node

The Return (Set) node shapes the output using the source workflow's assignments format (object type with inline expression):

- **For image/video models** (output via `resultUrls`):
  ```json
  {
    "assignments": {
      "assignments": [
        {
          "name": "generated_video",
          "type": "object",
          "value": "={\n  url: \"{{ $json.data.resultJson.parseJson().resultUrls }}\",\n  filename: \"{{ $json.data.resultJson.parseJson().resultUrls[0].split('/').at(-1) }}\"\n}"
        }
      ]
    },
    "options": {}
  }
  ```

- **For text models** (output via `resultObject`):
  ```json
  {
    "assignments": {
      "assignments": [
        {
          "name": "generated_text",
          "type": "stringValue",
          "value": "={{ $json.data.resultJson.parseJson().resultObject }}"
        }
      ]
    },
    "options": {}
  }
  ```

### Step 7: Name the Workflow

Format: `🧩 <provider> / <short-model-name>`

| Model ID | Workflow name |
|----------|--------------|
| `seedream/5-lite-text-to-image` | `🧩 seedream / 5-lite T2I` |
| `kling/v2.6-image-to-video` | `🧩 kling / v2.6-I2V` |
| `grok-imagine/image-to-video` | `🧩 grok / imagine I2V` |
| `grok-imagine-video-1-5-preview` | `🧩 grok / imagine-video-1.5-preview I2V` |

### Step 8: Update the Sticky Note

```
## Docs
https://kie.ai/<provider>?model=<url-encoded-model-identifier>
```

## Architecture

Every kie.ai workflow follows the same pattern (13 nodes, 9 connections):

```
Start → Super Code → Create Task → Queued → Loop Over Items → Return → No Op
                                      ↓           ↑
                                 Stop Error1     Get Generated [Media]
                                                    ↓
                                                  Switch ──success──→ Loop Over Items
                                                    │──waiting──→ Wait → Get Generated [Media]
                                                    │──failed──→ Stop and Error
```

### Node roles

| Node | Type | Purpose |
|------|------|---------|
| **Start** | `n8n-nodes-base.executeWorkflowTrigger` v1.1 | Workflow inputs — one per schema field. Called as sub-workflow via Execute Workflow node. |
| **Super Code** | `@kenkaiii/n8n-nodes-supercode.superCodeNodeVmSafe` v1 | Zod schema validation + payload construction. Community node — must be installed. |
| **Create Task** | `n8n-nodes-base.httpRequest` v4.3/4.4 | `POST https://api.kie.ai/api/v1/jobs/createTask` |
| **Queued** | `n8n-nodes-base.switch` v3.4 | Checks `$json.code === 200` (generating) vs not (error) |
| **Stop and Error1** | `n8n-nodes-base.stopAndError` v1 | Shows `$json.msg` when queueing fails |
| **Loop Over Items** | `n8n-nodes-base.splitInBatches` v3 | Output 0: done → Return. Output 1: continue → poll |
| **Get Generated [Media]** | `n8n-nodes-base.httpRequest` v4.3/4.4 | `GET .../recordInfo?taskId=${taskId}` |
| **Switch** | `n8n-nodes-base.switch` v3.4 | Checks `$json.data.state`: success / waiting\|queuing\|generating / fail |
| **Wait** | `n8n-nodes-base.code` v2 | 5-second delay |
| **Return** | `n8n-nodes-base.set` v3.4 | Shapes output: `generated_video` / `generated_image` / `generated_text` |
| **No Operation** | `n8n-nodes-base.noOp` v1 | Terminal node after Return |
| **Stop and Error** | `n8n-nodes-base.stopAndError` v1 | Shows `$json.data.failMsg \|\| $json.data.errorMessage` |
| **Sticky Note** | `n8n-nodes-base.stickyNote` v1 | Docs URL reference |

### Credentials

All HTTP Request nodes use the same `httpBearerAuth` credential from the source:

```json
"credentials": {
  "httpBearerAuth": {
    "id": "gv9noTXHGc9OkhZy",
    "name": "Kie.ai Token"
  }
}
```

The credential key MUST be `httpBearerAuth`, never `genericAuthType`.

## Post-Creation Steps

### 1. Validate

Run `n8n_validate_workflow`. These warnings are expected — ignore them:
- "Unknown node type: @kenkaiii/..." — community node, works when installed
- "Outdated typeVersion: 4.3" — cosmetic, source uses same version
- "missing onError" / "without error handling" — pre-existing from source

### 2. Auto-fix typeVersions (best effort)

Run `n8n_autofix_workflow` in **preview only**. Apply fixes manually in the next `update_full_workflow` call if desired. Never use `applyFixes: true`.

### 3. Test via Sub-Workflow

Since `activateWorkflow` is blocked, testing requires manual activation and a wrapper workflow.

**3a. Create test wrapper** (`n8n_create_workflow`):
```
Webhook (POST) → Code → Execute Workflow (calls main)
```
- **Code node**: `return [{json: {image_urls: ["https://..."], duration: 5}}]` — actual JS arrays, NOT Set node expressionValue (passes as string)
- **Execute Workflow**: `source:"database"`, `workflowId:"<main-id>"`, `mode:"each"`, `options:{"waitForSubExecutions":true}`

**3b. Tell user**: "Activate both workflows: [urls]"

**3c. Run**: `n8n_test_workflow(workflowId:"<test-id>", triggerType:"webhook", waitForResponse:true)`

**3d. Debug** failures with `n8n_executions(action:"get", mode:"error")` — check `upstreamContext.sampleItems` for data issues

**3e. Cleanup**: `n8n_delete_workflow(test-id)`

### 4. Report

Summarize: workflow ID/name/URL, architecture, changes vs source, preserved items, test result.

## Super Code Node Code Template

Only the **Schema constant** changes between models:

```javascript
const Schema = z.object({
  model: z.string().default("<model-identifier>"),
  input: z.object({
    // ← ONLY THESE FIELDS CHANGE
  })
})
// ← REFINEMENTS (IF ANY FROM DOCS) GO HERE

// Get defaults
const defaults = getDefaults(Schema);

// ################################################################# //
// EVERYTHING BELOW THE DIVIDER IS IMMUTABLE — NEVER MODIFY
// ################################################################# //

if (typeof item !== "undefined"){
  return processData(item, defaults, (data) => {
    const {model, ...rest} = data
    return {model, input: rest}
  })
}

if (typeof items !== "undefined"){
  return items.map(item => processData(item, defaults, (data) => {
    const {model, ...rest} = data
    return {model, input: rest}
  }))
}

function getDefaults(schema) {
  if (!(schema instanceof z.ZodObject)) return undefined;
  const shape = schema.shape;
  return Object.fromEntries(
    Object.entries(shape).map(([key, value]) => {
      if (value instanceof z.ZodDefault) return [key, value._def.defaultValue()];
      if (value instanceof z.ZodObject) return [key, getDefaults(value)];
      return [key, undefined];
    })
  );
}

function setDefaults(item, defaultConfig) {
  const data = item?.json ? item.json : item;
  const allKeys = [...new Set([...Object.keys(data), ...Object.keys(defaultConfig)])];
  const processed = {};
  allKeys.forEach(key => {
    const value = data[key];
    const finalValue = (value === null || value === undefined) ? defaultConfig[key] : value;
    if (finalValue !== null && finalValue !== undefined) processed[key] = finalValue;
  });
  return processed;
}

function processData(item, defaultConfig, formatDataFn = (data) => data){
  item = item?.json ? item.json : item;
  const data = formatDataFn(setDefaults(item, defaultConfig));
  const result = Schema.safeParse(data);
  if (result.success) return result.data;
  else {
    const issues = result.error.issues.map(issue =>
      `  • ▶️ ${issue.path.join('.')}: ${issue.message} (expected ${issue.expected}, got ${issue.received})`
    ).join('\n');
    throw new Error(`⚠️ Validation Error:\n${issues}`);
  }
}
```

## Switch Node Configurations

### Queued Switch (checks createTask response)
- **Output 0 "generating"**: `$json.code` equals `200` (number)
- **Output 1 "error"**: `$json.code` not equals `200` (number)

### Status Switch (checks poll response state)
- **Output 0 "success"**: `$json.data.state` equals `"success"` (string)
- **Output 1 "waiting"**: `$json.data.state` matches regex `^(waiting|queuing|generating)?$`
- **Output 2 "failed"**: `$json.data.state` equals `"fail"` (string)

## Quick Reference

### Standard Node Positions
```
Start: [336,-208]    Super Code: [560,-208]    Create Task: [784,-208]
Queued: [1008,-208]   Stop Error1: [1232,0]     Loop: [1232,-224]
Get Media: [1456,-208] Return: [1456,-400]      Switch: [1696,-224]
NoOp: [1664,-400]    Stop Error: [1872,0]       Wait: [1904,-208]
Sticky: [368,-448]
```

### Standard Connections
```json
{
  "Start": {"main": [[{"node": "Super Code", "type": "main", "index": 0}]]},
  "Super Code": {"main": [[{"node": "Create Task", "type": "main", "index": 0}]]},
  "Create Task": {"main": [[{"node": "Queued", "type": "main", "index": 0}]]},
  "Queued": {"main": [[{"node": "Loop Over Items", "type": "main", "index": 0}], [{"node": "Stop and Error1", "type": "main", "index": 0}]]},
  "Loop Over Items": {"main": [[{"node": "Return", "type": "main", "index": 0}], [{"node": "Get Generated [Media]", "type": "main", "index": 0}]]},
  "Get Generated [Media]": {"main": [[{"node": "Switch", "type": "main", "index": 0}]]},
  "Switch": {"main": [[{"node": "Loop Over Items", "type": "main", "index": 0}], [{"node": "Wait", "type": "main", "index": 0}], [{"node": "Stop and Error", "type": "main", "index": 0}]]},
  "Return": {"main": [[{"node": "No Operation, do nothing", "type": "main", "index": 0}]]},
  "Wait": {"main": [[{"node": "Get Generated [Media]", "type": "main", "index": 0}]]}
}
```

### HTTP Node Configs
- **Create Task**: `POST https://api.kie.ai/api/v1/jobs/createTask`, `jsonBody: "={{ $json.toJsonString() }}"`
- **Get Media**: `GET https://api.kie.ai/api/v1/jobs/recordInfo`, query `taskId={{ $json.data.taskId }}`

## Critical Rules

1. ❌ **Never `nsfw_checker`** — exclude from schema, inputs, payload
2. ❌ **Never `n8n_update_partial_workflow`** — use `n8n_update_full_workflow` for everything
3. ✅ **Always Super Code node** (`@kenkaiii/n8n-nodes-supercode.superCodeNodeVmSafe`)
4. ✅ **Always duplicate from source** — preserves Switch rule IDs, HTTP configs, credentials
5. ✅ **Change only 5 things**: Start inputs, Super Code schema, Return field, Media node name, Sticky note
6. ✅ **Node `id` fields required** by `n8n_create_workflow`
7. ✅ **Test Code node uses real JS arrays** — not Set node expressionValue
8. ✅ **Manual activation** — tell user, don't try `activateWorkflow`