# Phase 1: n8n Workflows — notify + ralph-loop-auth

**Depends on:** Nothing
**Produces:** `gsnake-n8n/tools/n8n-flows/notify.json`, `gsnake-n8n/tools/n8n-flows/ralph-loop-auth.json`

______________________________________________________________________

## Overview

Create two small n8n workflows as JSON files. Both are internal building blocks for the
ralph-loop system. Neither depends on the bridge service — they can be implemented and
tested independently.

**Working directory:** `~/git/gSnake/gsnake-n8n`

______________________________________________________________________

## 1A: notify workflow

**SOP:** `workflows/n8n-workflow/notify.md`
**Output:** `tools/n8n-flows/notify.json`
**Workflow ID:** `notify`

### Nodes to create (5 nodes)

#### Node 1 — Execute Workflow Trigger

```json
{
  "parameters": {},
  "id": "trigger",
  "name": "Execute Workflow Trigger",
  "type": "n8n-nodes-base.executeWorkflowTrigger",
  "typeVersion": 1,
  "position": [250, 300]
}
```

This node receives the payload from callers via n8n's Execute Workflow mechanism.
It is NOT a webhook — it has no external URL.

Input payload shape (passed by caller):

```json
{
  "title": "Short headline (required)",
  "body": "Optional detail text",
  "level": "info | success | warning | error",
  "source": "ralph | ci | manual",
  "context": {}
}
```

#### Node 2 — Code: format message

Type: `n8n-nodes-base.code`, typeVersion 2, mode `runOnceForAllItems`.

JavaScript logic:

```js
const item = $input.first().json;

const colorMap = {
  info:    0x5865F2,
  success: 0x57F287,
  warning: 0xFEE75C,
  error:   0xED4245,
};

const level = item.level || 'info';
const color = colorMap[level] || colorMap.info;
const title = item.title || 'Notification';
const body = item.body || '';

return [{
  json: {
    title,
    body,
    color,
    level,
    source: item.source || 'unknown',
    context: item.context || {},
  }
}];
```

#### Node 3 — Switch: channel routing

Type: `n8n-nodes-base.switch`, typeVersion 3.

Currently always routes to Discord (output 0). The Switch exists as a structural
placeholder. Configure it with a single fallback/default output that goes to the
Discord node. No actual routing condition needed yet — everything goes to output 0.

#### Node 4 — Discord: send notification

Type: `n8n-nodes-base.discord`, typeVersion 2.

Configuration:

- Credential: `discordWebhookApi` (existing credential in n8n — bind after import)
- Send an embed with:
  - `title`: `{{ $json.title }}`
  - `description`: `{{ $json.body }}`
  - `color`: `{{ $json.color }}`
- `continueOnFail: true` — Discord outages must not propagate errors to callers

#### Node 5 — NoOp (done)

Type: `n8n-nodes-base.noOp`, typeVersion 1.

Terminal node. Just marks the end of the flow.

### Node connections

```
Execute Workflow Trigger → Code(format) → Switch(channel) → Discord → NoOp(done)
```

### Workflow-level settings

```json
{
  "id": "notify",
  "name": "Notify",
  "active": false,
  "settings": { "executionOrder": "v1" }
}
```

______________________________________________________________________

## 1B: ralph-loop-auth workflow

**SOP:** `workflows/n8n-workflow/ralph-loop-auth.md`
**Output:** `tools/n8n-flows/ralph-loop-auth.json`
**Workflow ID:** `ralph-loop-auth`

### Nodes to create (6 nodes)

#### Node 1 — Webhook

Type: `n8n-nodes-base.webhook`, typeVersion 2.

```json
{
  "parameters": {
    "httpMethod": "POST",
    "path": "ralph",
    "responseMode": "lastNode",
    "options": {}
  },
  "id": "webhook",
  "name": "Webhook",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "position": [250, 300],
  "webhookId": "ralph-webhook"
}
```

Key: `responseMode: "lastNode"` — the workflow waits for a terminal node to produce
the HTTP response (either 202 or 401).

#### Node 2 — Code: validate auth

Type: `n8n-nodes-base.code`, typeVersion 2, mode `runOnceForAllItems`.

```js
const crypto = require('crypto');
const headers = $input.first().json.headers;
const authHeader = headers['authorization'] ?? '';
const expected = 'Bearer ' + $vars.RALPH_WEBHOOK_TOKEN;

const a = Buffer.from(authHeader);
const b = Buffer.from(expected);
const valid = a.length === b.length && crypto.timingSafeEqual(a, b);

return [{ json: { valid, body: $input.first().json.body } }];
```

Important:

- Uses `$vars.RALPH_WEBHOOK_TOKEN` (n8n Variable, set in UI → Settings → Variables)
- Uses `crypto.timingSafeEqual` to resist timing attacks
- Passes the original `body` through on success

#### Node 3 — If: auth valid?

Type: `n8n-nodes-base.if`, typeVersion 2.

Condition: `{{ $json.valid }}` equals `true`.

- True output (index 0) → Execute Workflow
- False output (index 1) → Respond 401

#### Node 4 — Execute Workflow: ralph-loop

Type: `n8n-nodes-base.executeWorkflow`, typeVersion 1.1.

```json
{
  "parameters": {
    "workflowId": {
      "__rl": true,
      "value": "ralph-loop",
      "mode": "id"
    },
    "options": {
      "waitForSubWorkflow": false
    }
  }
}
```

Key: `waitForSubWorkflow: false` — fire-and-forget. The ralph-loop is long-running
and async; we must not block the HTTP response waiting for it.

Input data: the `$json.body` from the Code node (the original request payload).

**Important n8n detail:** When using `waitForSubWorkflow: false`, the Execute Workflow
node fires the sub-workflow asynchronously. The calling workflow continues to the next
node immediately.

#### Node 5 — Respond: 202 Accepted

Type: `n8n-nodes-base.respondToWebhook`, typeVersion 1.1.

```json
{
  "parameters": {
    "respondWith": "json",
    "responseBody": "={{ JSON.stringify({ status: 'accepted' }) }}",
    "options": {
      "responseCode": 202
    }
  }
}
```

#### Node 6 — Respond: 401 Unauthorized

Type: `n8n-nodes-base.respondToWebhook`, typeVersion 1.1.

```json
{
  "parameters": {
    "respondWith": "json",
    "responseBody": "={{ JSON.stringify({ error: 'unauthorized', message: 'Invalid or missing bearer token.' }) }}",
    "options": {
      "responseCode": 401
    }
  }
}
```

### Node connections

```
Webhook → Code(validate auth) → If(valid?)
  true  → Execute Workflow(ralph-loop) → Respond 202
  false → Respond 401
```

The If node has two outputs:

- Output index 0 (true) → Execute Workflow → Respond 202
- Output index 1 (false) → Respond 401

### Workflow-level settings

```json
{
  "id": "ralph-loop-auth",
  "name": "Ralph Loop Auth",
  "active": false,
  "settings": { "executionOrder": "v1" }
}
```

______________________________________________________________________

## Post-implementation steps

### Update CLAUDE.md SOP mapping table

In `gsnake-n8n/CLAUDE.md`, update the SOP mapping table:

```
| `workflows/n8n-workflow/notify.md` | `tools/n8n-flows/notify.json` | ✅ Implemented |
| `workflows/n8n-workflow/ralph-loop-auth.md` | `tools/n8n-flows/ralph-loop-auth.json` | ✅ Implemented |
```

### Update frontmatter

In both SOP files, set:

```yaml
implementation_status: implemented
last_updated: "<today's date>"
```

______________________________________________________________________

## Verification

### notify verification

1. Import workflow:

   ```bash
   ./tools/scripts/sync-workflows.sh import
   ```

1. In n8n UI, open the "Notify" workflow. Verify node layout matches the spec.

1. Test with n8n "Test Workflow" button using this input:

   ```json
   {
     "title": "Test notification",
     "body": "Phase 1 verification.",
     "level": "info",
     "source": "manual"
   }
   ```

1. Expected: Discord message appears with blue embed (#5865F2), title "Test notification".

1. Test error level:

   ```json
   {
     "title": "Error test",
     "body": "Should be red.",
     "level": "error",
     "source": "manual"
   }
   ```

1. Expected: Discord message with red embed (#ED4245).

### ralph-loop-auth verification

1. Import and **activate** the workflow in n8n UI.

1. Test valid token:

   ```bash
   curl -s -o /dev/null -w "%{http_code}" \
     -X POST https://n8n.labs.lair.nntin.xyz/webhook/ralph \
     -H 'Content-Type: application/json' \
     -H "Authorization: Bearer $RALPH_WEBHOOK_TOKEN" \
     -d '{"action":"start","tool":"claude","maxIterations":1}'
   ```

   Expected: `202`

1. Test invalid token:

   ```bash
   curl -s -o /dev/null -w "%{http_code}" \
     -X POST https://n8n.labs.lair.nntin.xyz/webhook/ralph \
     -H 'Content-Type: application/json' \
     -H 'Authorization: Bearer wrong-token' \
     -d '{"action":"start"}'
   ```

   Expected: `401`

1. Test missing header:

   ```bash
   curl -s -o /dev/null -w "%{http_code}" \
     -X POST https://n8n.labs.lair.nntin.xyz/webhook/ralph \
     -H 'Content-Type: application/json' \
     -d '{"action":"start"}'
   ```

   Expected: `401`

**Note:** The Execute Workflow node targeting `ralph-loop` will fail during Phase 1
testing because that workflow doesn't exist yet. This is expected — the auth validation
and HTTP response codes are what we verify here. The `waitForSubWorkflow: false` setting
means the 202 response is sent before the sub-workflow execution is attempted, so the
curl tests should still return 202.

______________________________________________________________________

## n8n JSON structure reference

Use the existing workflows in `tools/n8n-flows/` as structural reference. Key patterns:

- Top-level fields: `id`, `name`, `active`, `nodes`, `connections`, `settings`, etc.
- Each node has: `parameters`, `id`, `name`, `type`, `typeVersion`, `position`
- Connections are keyed by node name, with `main` array of output arrays
- `settings.executionOrder` should be `"v1"`
- Set `active: false` — workflows are activated manually after import

Refer to `tools/n8n-flows/github-discord-notify.json` for a complete working example
of the JSON structure, node format, and connection wiring.
