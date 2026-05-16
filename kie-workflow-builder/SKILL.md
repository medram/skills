---
name: kie-workflow-builder
description: Build n8n sub-workflows from kie.ai model documentation URLs. Use this skill whenever the user provides a kie.ai docs URL and wants to create or duplicate an n8n workflow for that API — including phrases like "create a workflow from this doc", "duplicate this workflow for X model", "implement this API in n8n", or "add a new kie.ai model". Also trigger when the user mentions integrating any kie.ai model (seedream, kling, sora, veo, grok, wan, bytedance, nano-banana, z-image, etc.) into n8n.
---

# Kie.ai → n8n Workflow Builder

This skill builds n8n sub-workflows that call the kie.ai API. Given a model documentation URL, it extracts the payload schema, builds a Zod validator inside a Super Code node, wires up the standard create-task → poll-loop → return pipeline, and outputs a ready-to-use workflow.

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
| **Super Code** | `@kenkaiii/n8n-nodes-supercode.superCodeNodeVmSafe` v1 | Zod schema validation + payload construction |
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

Read the parameters table from the docs and construct a Zod schema inside the Super Code node. Follow these rules:

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

**Final formatter** — The Super Code node always transforms the validated data into the API format:
```javascript
(data) => {
  const {model, ...rest} = data
  return {model, input: rest}
}
```

### 4. Preserve the helper functions

The Super Code node includes four helper functions that must be kept verbatim:

1. `getDefaults(schema)` — recursively extracts default values from a Zod schema
2. `setDefaults(item, defaultConfig)` — merges input with defaults, removes null/undefined
3. `processData(item, defaultConfig, formatDataFn)` — validates with safeParse, throws formatted error on failure
4. Entry logic — handles both `item` (single) and `items` (batch) execution modes

### 5. Set the Start node inputs

Map each Zod `input` field to a workflow input. Order: `prompt` first, then required fields, then optional fields. Set `type: "array"` on array fields. Do NOT include `nsfw_checker`.

### 6. Build the Return node

The Return (Set) node shapes the output:

- **For image/video models** (output via `resultUrls`):
  ```
  generated_image / generated_video = {
    url: "{{ $json.data.resultJson.parseJson().resultUrls }}",
    filename: "{{ $json.data.resultJson.parseJson().resultUrls[0].split('/').at(-1) }}"
  }
  ```
- **For text models** (output via `resultObject`):
  ```
  generated_text = "{{ $json.data.resultJson.parseJson().resultObject }}"
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

**If duplicating** an existing workflow (user says "duplicate X for Y model"):
1. Use `n8n_get_workflow` on the source with `mode: "full"`
2. Modify the nodes as described above (Super Code schema, Start inputs, Return output, node names, Sticky Note)
3. Create with `n8n_create_workflow`

**If creating from scratch** (no source workflow to duplicate):
1. Build all 13 nodes with the positions and connections from the architecture table
2. Create with `n8n_create_workflow`

### 10. Post-creation steps

1. **Auto-fix typeVersions** — run `n8n_autofix_workflow` with `fixTypes: ["typeversion-upgrade"]`
2. **Validate** — run `n8n_validate_workflow` to check for issues
3. **Report** — summarize what was created and what changed vs. the source (if any)

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

## Error handling notes

- The **Queued** Switch checks `code === 200` (generating) vs not (error → Stop and Error1 with `$json.msg`)
- The **Switch** (after polling) checks `data.state`: `success` → loop done, `waiting|queuing|generating` → wait & retry, `fail` → Stop and Error with `data.failMsg || data.errorMessage`
- The **Wait** node uses a 5-second delay between polls

## Critical rules

- **Never include `nsfw_checker`** in any schema, input, or payload — always skip it regardless of what the docs say
- Always use the `@kenkaiii/n8n-nodes-supercode.superCodeNodeVmSafe` type for the Super Code node
- Always use the "Kie.ai Token" bearer auth credential
- Always auto-fix type versions after creation
- The Super Code node must contain the complete Zod schema + all helper functions — no external dependencies
