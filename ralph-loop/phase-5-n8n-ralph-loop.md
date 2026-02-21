# Phase 5: n8n ralph-loop Workflow

**Depends on:** Phases 1-4 (all must be complete and deployed)
**Produces:** `gsnake-n8n/tools/n8n-flows/ralph-loop.json`

______________________________________________________________________

## Overview

Create the main n8n orchestration workflow that drives the ralph loop. This is the most
complex workflow — it handles both the "start" path (new campaign) and the "done" path
(iteration callback), routing through status checks, PRD evaluation, and bridge API calls.

**Working directory:** `~/git/gSnake/gsnake-n8n`
**SOP:** `workflows/n8n-workflow/ralph-loop.md`

______________________________________________________________________

## Workflow metadata

```json
{
  "id": "ralph-loop",
  "name": "Ralph Loop",
  "active": false,
  "settings": { "executionOrder": "v1" }
}
```

______________________________________________________________________

## Node inventory

This workflow has ~20 nodes. Below is every node with its exact configuration.

### Node 1 — Execute Workflow Trigger

Entry point. Receives payload from `ralph-loop-auth` via Execute Workflow.

```json
{
  "parameters": {},
  "id": "trigger",
  "name": "Execute Workflow Trigger",
  "type": "n8n-nodes-base.executeWorkflowTrigger",
  "typeVersion": 1,
  "position": [250, 400]
}
```

Input payload shape (from auth gateway):

For `action: "start"`:

```json
{ "action": "start", "tool": "claude", "maxIterations": 20 }
```

For `action: "done"` (from bridge callback):

```json
{
  "action": "done", "jobId": "uuid", "iteration": 3, "tool": "claude",
  "success": true, "callbackUrl": "https://...", "maxIterations": 20,
  "exitCode": 0, "timedOut": false, "aborted": false, "logFile": "..."
}
```

### Node 2 — Switch: action

**Data flow from auth gateway → ralph-loop:**

In the auth gateway (Phase 1), the Code node returns `{ valid, body }` where `body` is
the original webhook request body. The Execute Workflow node in ralph-loop-auth must be
configured to pass `$json.body` as input data to the ralph-loop sub-workflow.

This means the Execute Workflow Trigger in ralph-loop receives the original request
body directly. Fields are at `$json.action`, `$json.tool`, etc. — NOT nested under
`.body`.

**The Phase 1 auth workflow's Execute Workflow node must pass `$json.body` as input.**
If it passes `$json` instead, the data would be nested under `$json.body` in ralph-loop.
Verify this during integration testing.

Type: `n8n-nodes-base.switch`, typeVersion 3.

Rules:

- Output 0: `$json.action` equals `"start"` → start path
- Output 1: `$json.action` equals `"done"` → done path
- Fallback (output 2): → error notification

```json
{
  "parameters": {
    "rules": {
      "values": [
        {
          "conditions": {
            "options": { "caseSensitive": true, "leftValue": "" },
            "conditions": [
              { "leftValue": "={{ $json.action }}", "rightValue": "start", "operator": { "type": "string", "operation": "equals" } }
            ]
          },
          "renameOutput": true,
          "outputKey": "start"
        },
        {
          "conditions": {
            "options": { "caseSensitive": true, "leftValue": "" },
            "conditions": [
              { "leftValue": "={{ $json.action }}", "rightValue": "done", "operator": { "type": "string", "operation": "equals" } }
            ]
          },
          "renameOutput": true,
          "outputKey": "done"
        }
      ]
    },
    "options": { "fallbackOutput": "extra" }
  },
  "id": "switch-action",
  "name": "Switch Action",
  "type": "n8n-nodes-base.switch",
  "typeVersion": 3,
  "position": [450, 400]
}
```

______________________________________________________________________

## START PATH (output 0 from Switch)

### Node 3 — HTTP: GET /status (start)

```json
{
  "parameters": {
    "url": "http://host-gateway:8765/status",
    "options": {}
  },
  "id": "get-status-start",
  "name": "GET Status (start)",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [650, 200]
}
```

### Node 4 — If: running? (start)

Condition: `{{ $json.running }}` equals `true`.

- True (output 0) → Notify "already running"
- False (output 1) → GET /prd.json

### Node 5 — Notify: already running (start)

Execute Workflow node calling `notify`:

```json
{
  "parameters": {
    "workflowId": { "__rl": true, "value": "notify", "mode": "id" },
    "options": { "waitForSubWorkflow": false }
  }
}
```

Input data (use a Code node before this, or use expressions in the Execute Workflow):

```json
{
  "title": "Ralph already running",
  "body": "An iteration is already in flight. Wait for it to complete or call POST /abort.",
  "level": "info",
  "source": "ralph"
}
```

**Implementation note:** n8n's Execute Workflow node with `waitForSubWorkflow: false`
requires the input data to be structured. Use a Code node before each Notify call to
format the payload, OR use the Execute Workflow node's "Workflow Input" mapping.

**Simpler approach:** Put a Code node before each Execute Workflow (notify) call that
formats `{ title, body, level, source }`. This is verbose but unambiguous.

### Node 6 — HTTP: GET /prd.json (start)

```json
{
  "parameters": {
    "url": "http://host-gateway:8765/prd.json",
    "options": {}
  }
}
```

### Node 7 — Code: count remaining (start)

```js
const prd = $input.first().json;
const stories = prd.userStories || [];
const remaining = stories.filter(s => s.passes === false).length;
const total = stories.length;
const nextStory = stories.find(s => s.passes === false);

// Thread the original request data through
// Access it from the trigger node
const triggerData = $node["Execute Workflow Trigger"].json;
const tool = triggerData.tool;
const maxIterations = triggerData.maxIterations;

return [{
  json: {
    remaining,
    total,
    nextStory: nextStory ? { id: nextStory.id, title: nextStory.title } : null,
    tool,
    maxIterations
  }
}];
```

### Node 8 — If: all done? (start)

Condition: `{{ $json.remaining }}` equals `0`.

- True (output 0) → Notify "nothing to do"
- False (output 1) → POST /reset

### Node 9 — Notify: nothing to do (start)

```json
{ "title": "Nothing to do", "body": "All stories in PRD already pass.", "level": "success", "source": "ralph" }
```

### Node 10 — HTTP: POST /reset

```json
{
  "parameters": {
    "method": "POST",
    "url": "http://host-gateway:8765/reset",
    "options": {}
  }
}
```

### Node 11 — HTTP: POST /run-ralph (start)

```json
{
  "parameters": {
    "method": "POST",
    "url": "http://host-gateway:8765/run-ralph",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ JSON.stringify({ tool: $node['Code Count Remaining (start)'].json.tool, callbackUrl: 'https://n8n.labs.lair.nntin.xyz/webhook/ralph', maxIterations: $node['Code Count Remaining (start)'].json.maxIterations }) }}",
    "options": {}
  }
}
```

**Note:** The `callbackUrl` is hardcoded here since it's always the same webhook URL.
`tool` and `maxIterations` come from the original request (threaded through the Code node).

### Node 12 — Switch: run-ralph status (start)

Routes on `$json.status`:

- `"started"` → Notify "started"
- `"max_iterations_reached"` → Notify "max hit"
- `"already_running"` → Notify "busy"
- Fallback → Notify "error"

### Node 13 — Notify: started

```json
{
  "title": "Ralph started",
  "body": "Iteration 1 starting. {{ remaining }} stories remaining. tool={{ tool }}, maxIterations={{ maxIterations }}",
  "level": "info",
  "source": "ralph",
  "context": { "remaining": "...", "tool": "...", "maxIterations": "..." }
}
```

### Node 14 — Notify: max iterations reached (start)

```json
{ "title": "Max iterations reached", "body": "...", "level": "warning", "source": "ralph" }
```

### Node 15 — Notify: already running (bridge guard)

```json
{ "title": "Ralph already running", "body": "Bridge reports a process is already in flight.", "level": "info", "source": "ralph" }
```

______________________________________________________________________

## DONE PATH (output 1 from Switch)

### Node 16 — If: success? (done)

Condition: `{{ $json.success }}` equals `true`.

- True (output 0) → GET /status (done)
- False (output 1) → Notify "iteration failed"

### Node 17 — Notify: iteration failed

Format based on `$json.timedOut`, `$json.aborted`, `$json.exitCode`:

```js
const data = $input.first().json;
let title, level;
if (data.aborted) {
  title = `Iteration ${data.iteration} aborted`;
  level = 'warning';
} else if (data.timedOut) {
  title = `Iteration ${data.iteration} timed out`;
  level = 'error';
} else {
  title = `Iteration ${data.iteration} failed (exitCode=${data.exitCode})`;
  level = 'error';
}
return [{ json: { title, body: `tool=${data.tool}, logFile=${data.logFile || 'n/a'}`, level, source: 'ralph', context: data } }];
```

### Node 18 — HTTP: GET /status (done)

Same as Node 3 but on the done path. Separate node instance.

### Node 19 — If: running? (done)

Same logic as Node 4. If running=true → Notify "already running" (race guard).

### Node 20 — HTTP: GET /prd.json (done)

Same as Node 6.

### Node 21 — Code: count remaining (done)

Similar to Node 7 but extracts `tool`, `maxIterations`, and `callbackUrl` from the
callback payload (from the trigger node):

```js
const prd = $input.first().json;
const stories = prd.userStories || [];
const remaining = stories.filter(s => s.passes === false).length;

const triggerData = $node["Execute Workflow Trigger"].json;
const tool = triggerData.tool;
const maxIterations = triggerData.maxIterations;
const callbackUrl = triggerData.callbackUrl;

return [{
  json: {
    remaining,
    total: stories.length,
    tool,
    maxIterations,
    callbackUrl,
    iteration: triggerData.iteration
  }
}];
```

### Node 22 — If: all done? (done)

Condition: `$json.remaining === 0`.

- True → Notify "all complete"
- False → POST /run-ralph (done)

### Node 23 — Notify: all complete

```json
{ "title": "All stories complete!", "body": "All user stories in PRD pass.", "level": "success", "source": "ralph" }
```

### Node 24 — HTTP: POST /run-ralph (done)

Same structure as Node 11 but reads `tool`, `maxIterations`, and `callbackUrl` from
the Code node on the done path (these were threaded through the callback payload).

### Node 25 — Switch: run-ralph status (done)

Same routing as Node 12. On `"started"`, notify "iteration done":

```json
{
  "title": "Iteration {{ iteration }} done",
  "body": "{{ remaining }} stories remaining. Starting next iteration.",
  "level": "info",
  "source": "ralph"
}
```

______________________________________________________________________

## ERROR PATH (fallback from Switch: action)

### Node 26 — Notify: error (unknown action)

```json
{ "title": "Ralph error", "body": "Unknown action received.", "level": "error", "source": "ralph" }
```

______________________________________________________________________

## Connection map

```
Trigger → Switch(action)

  "start" → GET /status(start) → If(running?)
              true  → [Code format] → Notify(already-running)
              false → GET /prd.json(start) → Code(remaining-start) → If(allDone?)
                        true  → [Code format] → Notify(nothing-to-do)
                        false → POST /reset → POST /run-ralph(start) → Switch(status-start)
                                  "started"              → [Code format] → Notify(started)
                                  "max_iterations_reached" → [Code format] → Notify(max-hit)
                                  "already_running"      → [Code format] → Notify(busy)
                                  fallback               → [Code format] → Notify(error)

  "done"  → If(success?)
              false → [Code format] → Notify(iteration-failed)
              true  → GET /status(done) → If(running-done?)
                        true  → [Code format] → Notify(already-running)
                        false → GET /prd.json(done) → Code(remaining-done) → If(allDone-done?)
                                  true  → [Code format] → Notify(all-complete)
                                  false → POST /run-ralph(done) → Switch(status-done)
                                            "started"              → [Code format] → Notify(iteration-done)
                                            "max_iterations_reached" → [Code format] → Notify(max-hit)
                                            "already_running"      → [Code format] → Notify(busy)
                                            fallback               → [Code format] → Notify(error)

  fallback → [Code format] → Notify(error)
```

Each `[Code format]` is a small Code node that produces `{ title, body, level, source, context }`
for the notify workflow. These can be combined with the Notify Execute Workflow node if the
Execute Workflow node supports inline expression mapping for input data.

______________________________________________________________________

## Post-implementation steps

### Update CLAUDE.md SOP mapping table

```
| `workflows/n8n-workflow/ralph-loop.md` | `tools/n8n-flows/ralph-loop.json` + `tools/scripts/ralph-bridge.js` | ✅ Implemented |
```

### Update frontmatter in all three SOP files

Set `implementation_status: implemented` in:

- `workflows/n8n-workflow/notify.md`
- `workflows/n8n-workflow/ralph-loop-auth.md`
- `workflows/n8n-workflow/ralph-loop.md`

### Check off remaining implementation checklist items in ralph-loop.md

______________________________________________________________________

## Verification

### Pre-requisites

Before testing, ensure:

1. Bridge service is running (`node ralph-bridge.js` or systemd)
1. `notify` workflow is imported and active (has `discordWebhookApi` credential bound)
1. `ralph-loop-auth` workflow is imported and active
1. `RALPH_WEBHOOK_TOKEN` is set in both n8n Variables and bridge env
1. n8n container has `extra_hosts: - "host-gateway:host-gateway"` in docker-compose
1. Bridge is reachable from n8n: `docker exec n8n curl -s http://host-gateway:8765/status`

### Test 1: Import and verify

```bash
./tools/scripts/sync-workflows.sh import
```

Open `ralph-loop` in n8n UI. Verify node count and connections match the spec.

### Test 2: Start with nothing to do

Set all stories to `passes: true` in prd.json, then:

```bash
curl -X POST https://n8n.labs.lair.nntin.xyz/webhook/ralph \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $RALPH_WEBHOOK_TOKEN" \
  -d '{"action":"start","tool":"claude","maxIterations":1}'
```

Expected: 202 response. Discord notification: "Nothing to do" (success/green).

### Test 3: Start with remaining stories

Set at least one story to `passes: false`, then:

```bash
curl -X POST https://n8n.labs.lair.nntin.xyz/webhook/ralph \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $RALPH_WEBHOOK_TOKEN" \
  -d '{"action":"start","tool":"claude","maxIterations":1}'
```

Expected: 202 response. Discord notification: "Ralph started — N stories remaining" (info/blue).
Bridge spawns claude. When claude finishes, callback arrives, "done" path executes.

### Test 4: maxIterations=1 stops after one iteration

With `maxIterations: 1`, after the first iteration completes:

Expected: Discord notification "Max iterations (1) reached" (warning/yellow).

### Test 5: Full loop with maxIterations=2

Set `maxIterations: 2` with 2+ remaining stories. Verify:

1. "Ralph started" notification
1. First iteration runs
1. "Iteration 1 done — N remaining" notification
1. Second iteration runs
1. Either "Iteration 2 done" + "Max iterations reached" or "All complete" depending on
   whether stories were completed

### Test 6: Bridge not reachable

Stop the bridge, then start a loop:

Expected: n8n HTTP Request node fails. Verify the error is handled gracefully and
an error notification is sent.

### Test 7: Already running guard

Start a loop, then immediately start another:

```bash
curl -X POST https://n8n.labs.lair.nntin.xyz/webhook/ralph \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $RALPH_WEBHOOK_TOKEN" \
  -d '{"action":"start","tool":"claude","maxIterations":5}'

# Immediately:
curl -X POST https://n8n.labs.lair.nntin.xyz/webhook/ralph \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $RALPH_WEBHOOK_TOKEN" \
  -d '{"action":"start","tool":"claude","maxIterations":5}'
```

Expected: Second request triggers "Ralph already running" info notification.

______________________________________________________________________

## Implementation tips

1. **Node naming:** Give every node a unique, descriptive name. Duplicate names cause
   issues with `$node["Name"]` references.

1. **Node positioning:** Use a grid layout. Start path at y=200, done path at y=600.
   Space nodes 200px apart horizontally.

1. **Code nodes for notification formatting:** Each terminal event needs a Code node
   that formats the notify payload before the Execute Workflow (notify) call. This is
   the most tedious but most reliable approach.

1. **Expression references:** Use `$node["Node Name"].json.field` to reference data
   from earlier nodes. Be careful with node names — they must match exactly.

1. **If node outputs:** In n8n typeVersion 2, output 0 is "true" and output 1 is "false".

1. **Test incrementally:** Import after adding a few nodes, test the start path first,
   then add the done path.

1. **Refer to existing workflows** in `tools/n8n-flows/` for the exact JSON structure,
   especially `github-multi-ci-suite-parent.json` which uses Execute Workflow and
   Switch nodes.
