---
name: kie-workflow-builder
description: Build n8n sub-workflows from kie.ai model documentation URLs. Use this skill whenever the user provides a kie.ai docs URL and wants to create or duplicate an n8n workflow for that API — including phrases like "create a workflow from this doc", "duplicate this workflow for X model", "implement this API in n8n", or "add a new kie.ai model". Also trigger when the user mentions integrating any kie.ai model (seedream, kling, sora, veo, grok, wan, bytedance, nano-banana, z-image, etc.) into n8n.
---

# Kie.ai → n8n Workflow Builder

This skill builds n8n sub-workflows that call the kie.ai API. Given a model documentation URL, it extracts the payload schema, builds a Zod validator inside a Super Code node (with Code node fallback), wires up the standard create-task → poll-loop → return pipeline, and outputs a ready-to-use workflow.

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
| **Start** | Execute Workflow Trigger | Workflow inputs — one per schema field |
| **Super Code** | `@kenkaiii/n8n-nodes-supercode.superCodeNodeVmSafe` v1 OR `n8n-nodes-base.code` v2 | Zod schema validation + payload construction |
| **Create Task** | HTTP Request v4.4 | `POST https://api.kie.ai/api/v1/jobs/createTask` |
| **Queued** | Switch v3.4 | Checks `$json.code === 200` (generating) vs not (error) |
| **Stop and Error1** | Stop and Error v1 | Shows `$json.msg` when queueing fails |
| **Loop Over Items** | Split In Batches v3 | Two outputs: done → Return, continue → poll |
| **Get Generated [Media]** | HTTP Request v4.4 | `GET https://api.kie.ai/api/v1/jobs/recordInfo?taskId=...` |
| **Switch** | Switch v3.4 | Checks `$json.data.state`: success / waiting / failed |
| **Wait** | Code v2 | `await new Promise(r => setTimeout(r, 5000))` — 5s polling delay |
| **Return** | Set v3.4 | Shapes the output: `generated_image`, `generated_video`, etc. |
| **No Operation** | NoOp v1 | Terminal node after Return |
| **Stop and Error** | Stop and Error v1 | Shows `$json.data.failMsg || $json.data.errorMessage` |
| **Sticky Note** | Sticky Note v1 | Docs URL reference |

### Credentials

All HTTP Request nodes use `httpBearerAuth` credential named **"Kie.ai Token"** (credential ID: `gv9noTXHGc9OkhZy`).

> **IMPORTANT**: The credential key in the `credentials` object MUST be `httpBearerAuth`, NOT `genericAuthType` or any other key. Example:
> ```json
> "credentials": {
>   "httpBearerAuth": {
>     "id": "gv9noTXHGc9OkhZy",
>     "name": "Kie.ai Token"
>   }
> }
> ```

## Step-by-step workflow

### 1. Fetch and parse the docs URL

Fetch the documentation page the user provides. Extract:

- **Model identifier** — the `model` string (e.g. `seedream/5-lite-image-to-image`)
- **Input parameters** — name, type, required/optional, options/enum values, defaults, max constraints
- **Output type** — image, video, or text (determined from `resultUrls` context and model name suffixes like T2I, I2I, T2V, I2V, V2V)

### 2. Determine the model category

Infer from the model name what the output is:

| Suffix | Output | Return field name | "Get Generated" node name |
|--------|--------|-------------------|---------------------------|
| T2I / I2I / image-to-image | Image | `generated_image` | Get Generated Image |
| T2V / I2V / V2V / image-to-video | Video | `generated_video` | Get Generated Video |
| text / other | Text (via `resultObject`) | `generated_text` | Get Generated Result |

### 3. Build the Zod schema

Read the parameters table from the docs and construct a Zod schema inside the Super Code node (or Code node fallback — see section 3b). Follow these rules:

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
- `prompt` (string): always include, use `max()` from docs, no `.optional()`
- `image_urls` (array of URLs): always include when docs say "Required: Yes" and type is array; use `z.array(z.string().url())`; add `.max(N)` only if docs specify a max count
- Enum fields: use `z.enum([...])` with exact values from docs; include `.default()` with the docs default
- Number fields with range: use `z.number().int().min(X).max(Y).default(Z)`
- Boolean fields: use `z.boolean().default(value)` if docs provide a default
- Optional fields: append `.optional()`
- **ALWAYS exclude `nsfw_checker`** — never include this parameter in the schema or payload, even if present in the docs

**Refinements** — Only add `.refine()` when there are genuine cross-field dependencies visible in the docs (e.g., "cannot use both X and Y simultaneously"). Do not invent refinements that aren't in the docs.

**Final formatter** — The node always transforms the validated data into the API format:
```javascript
(data) => {
  const {model, ...rest} = data
  return {model, input: rest}
}
```

### 3b. Super Code vs Code node fallback

**Preferred**: Use `@kenkaiii/n8n-nodes-supercode.superCodeNodeVmSafe` v1 (the Super Code node). This node supports Zod natively and has a built-in schema validation UI.

**Fallback**: If the Super Code community node is NOT installed on the n8n instance (you'll get a validation error referencing `@kenkaiii/n8n-nodes-supercode`), fall back to the standard `n8n-nodes-base.code` v2 (Code node). When using the Code node:

1. Replace `superCodeNodeVmSafe` with `n8n-nodes-base.code` in the node type
2. Change `typeVersion` from `1` to `2`
3. Set `parameters.mode` to `"runOnceForAllItems"`
4. Move all the Zod + helper logic into `parameters.jsCode`
5. **Handle webhook input nesting**: When the workflow is triggered via webhook, data arrives nested under `body`. The Code node must handle both formats:
   ```javascript
   const rawInput = items[0].json;
   const input = rawInput.body || rawInput;
   ```
6. Remove the Super Code-specific `parameters.schema` and `parameters.formatter` fields — the Code node doesn't use them

### 4. Preserve the helper functions

The Super Code / Code node includes four helper functions that must be kept verbatim:

1. `getDefaults(schema)` — recursively extracts default values from a Zod schema
2. `setDefaults(item, defaultConfig)` — merges input with defaults, removes null/undefined
3. `processData(item, defaultConfig, formatDataFn)` — validates with safeParse, throws formatted error on failure
4. Entry logic — handles both `item` (single) and `items` (batch) execution modes

When using the Code node fallback, these helpers go inside `jsCode` along with the schema and the entry logic.

### 5. Set the Start node inputs

Map each Zod `input` field to a workflow input. Order: `prompt` first, then required fields, then optional fields. Set `type: "array"` on array fields. Do NOT include `nsfw_checker`.

### 6. Build the Return node

The Return (Set) node shapes the output using Assignments mode (`mode: "manual"`):

> **CRITICAL**: Use `mode: "manual"` with `assignments.assignments[]` — do NOT use `mode: "raw"` with `jsonOutput`. The `jsonOutput` mode fails validation when expressions like `$json.data.resultJson.parseJson()` are used.

- **For image/video models** (output via `resultUrls`):
  ```json
  {
    "mode": "manual",
    "assignments": {
      "assignments": [
        {
          "name": "generated_image",
          "value": "={{ $json.data.resultJson.parseJson().resultUrls }}",
          "type": "arrayValue"
        },
        {
          "name": "filename",
          "value": "={{ $json.data.resultJson.parseJson().resultUrls[0].split('/').at(-1) }}",
          "type": "stringValue"
        }
      ]
    }
  }
  ```
  (Replace `generated_image` with `generated_video` and `arrayValue` stays the same for video models)

- **For text models** (output via `resultObject`):
  ```json
  {
    "mode": "manual",
    "assignments": {
      "assignments": [
        {
          "name": "generated_text",
          "value": "={{ $json.data.resultJson.parseJson().resultObject }}",
          "type": "stringValue"
        }
      ]
    }
  }
  ```

### 7. Name the workflow

Format: `🧩 <provider> / <short-model-name>`

Examples:
- `seedream/5-lite-image-to-image` → `🧩 seedream / 5-lite-I2I`
- `kling/v2.6-image-to-video` → `🧩 kling / v2.6-I2V`
- `grok-imagine/text-to-image` → `🧩 grok / imagine T2I`

Use the suffix convention: T2I (text-to-image), I2I (image-to-image), T2V (text-to-video), I2V (image-to-video), V2V (video-to-video).

### 8. Update the Sticky Note

Set content to:
```
## Docs
https://kie.ai/<provider>?model=<url-encoded-model-identifier>
```

### 9. Create or duplicate the workflow

**Preferred: Duplicate from an existing kie.ai workflow** (fastest, most reliable):
1. Use `n8n_get_workflow` on the source workflow with `mode: "full"`
2. Clear the `id` field from the workflow JSON and from every node
3. Modify the nodes:
   - Super Code or Code node: update Zod schema + model identifier
   - Start node: update workflow inputs to match new schema
   - Return node: update output field names and expressions
   - "Get Generated [Media]" node: rename to match output type (Image/Video/Result)
   - Sticky Note: update docs URL
4. Create with `n8n_create_workflow`

**If creating from scratch** (no source workflow to duplicate):
1. Build all 13 nodes with the positions and connections from the reference tables
2. Create with `n8n_create_workflow`

> **IMPORTANT**: Always prefer duplicating an existing workflow over building node-by-node. Duplicating preserves all the subtle configuration details (request options, authentication setup, switch rules, etc.) that are easy to miss when constructing from scratch.

### 10. Post-creation steps

1. **Auto-fix typeVersions** — run `n8n_autofix_workflow` with `fixTypes: ["typeversion-upgrade"]`
2. **Validate** — run `n8n_validate_workflow` to check for issues
3. **Test the workflow** (optional but recommended):
   - Add a **temporary Webhook node** (`n8n-nodes-base.webhook` v2) to the workflow
     - Set `path` to something unique like `<model-shortname>-t2i` (e.g., `seedream-t2i`)
     - Set `httpMethod` to `"POST"`
     - Position it before the Super Code / Code node
   - Connect Webhook → Super Code (or Code node)
   - Update the Start (Execute Workflow Trigger) connections to disconnect it (or remove it temporarily)
   - Use `n8n_test_workflow` with the webhook path and a test payload
   - **After successful test**: remove the temporary Webhook node and restore the Execute Workflow Trigger connections
4. **Report** — summarize what was created and what changed vs. the source (if any)

## Switch node configurations

### Queued Switch (checks createTask response)

- **Output 0 (true/success)**: `={{ $json.code === 200 ? 0 : 1 }}`
- **Output 1 (false/error)**: routes to Stop and Error1

### Status Switch (checks poll response state)

- **Output 0 (success)**: `={{ ['waiting', 'queuing', 'generating'].includes($json.data.state) ? 1 : ($json.data.state === 'success' ? 0 : 2) }}`
- **Output 1 (waiting/retrying)**: routes to Wait node
- **Output 2 (failed)**: routes to Stop and Error

## Quick reference: Standard node positions

```
Start:                    [336, -208]
Super Code:               [560, -208]
Create Task:              [784, -208]
Queued:                   [1008, -208]
Stop and Error1:          [1232, 0]
Loop Over Items:          [1232, -224]
Get Generated [Media]:    [1456, -208]
Return:                   [1456, -400]
Switch:                   [1696, -224]
No Operation:             [1664, -400]
Stop and Error:           [1872, 0]
Wait:                     [1904, -208]
Sticky Note:              [368, -448]
```

## Quick reference: Standard connections

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

## HTTP Request node configurations

### Create Task (POST)

```json
{
  "method": "POST",
  "url": "https://api.kie.ai/api/v1/jobs/createTask",
  "authentication": "genericCredentialType",
  "genericAuthType": "httpBearerAuth",
  "sendBody": true,
  "bodyParameters": {
    "parameters": [
      {
        "name": "body",
        "value": "={{ JSON.stringify($json) }}"
      }
    ]
  },
  "options": {
    "response": {
      "response": {
        "neverError": true
      }
    }
  }
}
```

### Get Generated [Media] (GET)

```json
{
  "method": "GET",
  "url": "=https://api.kie.ai/api/v1/jobs/recordInfo?taskId={{ $json.data.taskId }}",
  "authentication": "genericCredentialType",
  "genericAuthType": "httpBearerAuth",
  "options": {}
}
```

## Error handling notes

- The **Queued** Switch checks `code === 200` (generating) vs not (error → Stop and Error1 with `$json.msg`)
- The **Switch** (after polling) checks `data.state`: `success` → loop done, `waiting|queuing|generating` → wait & retry, `fail` → Stop and Error with `data.failMsg || data.errorMessage`
- The **Wait** node uses a 5-second delay between polls
- The Create Task HTTP Request should have `neverError: true` in response options so error responses can be checked by the Switch node instead of failing immediately

## Critical rules

- **Never include `nsfw_checker`** in any schema, input, or payload — always skip it regardless of what the docs say
- **Preferred: Super Code node** (`@kenkaiii/n8n-nodes-supercode.superCodeNodeVmSafe` v1) — but if not installed, **fall back to Code node** (`n8n-nodes-base.code` v2) with `mode: "runOnceForAllItems"` and webhook `body` nesting handling
- **Credential key MUST be `httpBearerAuth`** — never use `genericAuthType` or any other key in the `credentials` object
- **Return node MUST use `mode: "manual"` with `assignments`** — never use `mode: "raw"` with `jsonOutput` (it fails validation with expressions)
- **Always prefer duplicating** an existing kie.ai workflow over building node-by-node — it preserves subtle config details
- Always use the "Kie.ai Token" bearer auth credential
- Always auto-fix type versions after creation
- The Super Code or Code node must contain the complete Zod schema + all helper functions — no external dependencies
- For testing: add a temporary Webhook node, test, then clean up and remove it afterward
