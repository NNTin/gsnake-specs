# Phase 2: ralph-bridge.js — Read-only Endpoints + State Management

**Depends on:** Nothing (can run in parallel with Phase 1)
**Produces:** `gsnake-n8n/tools/scripts/ralph-bridge.js` (partial — read-only endpoints only)

______________________________________________________________________

## Overview

Create the Node.js HTTP bridge service with the foundational endpoints that don't spawn
processes. This establishes the state management layer that all subsequent phases build on.

**Working directory:** `~/git/gSnake/gsnake-n8n`
**Runtime directory (cwd for the bridge):** `~/git/gSnake`

**Constraint:** Node.js stdlib only — no npm dependencies. Use `node:http`, `node:fs`,
`node:path`, `node:crypto`. The file should be ESM (`import` syntax).

______________________________________________________________________

## File to create

`tools/scripts/ralph-bridge.js`

______________________________________________________________________

## Environment variables

The bridge reads these from `process.env` (injected by systemd EnvironmentFile in
production, or exported manually in dev):

| Variable | Default | Description |
|----------|---------|-------------|
| `RALPH_BRIDGE_PORT` | `8765` | Port to listen on |
| `RALPH_REPO_PATH` | (required) | Absolute path to gSnake repo root |
| `RALPH_N8N_PATH` | (required) | Absolute path to gsnake-n8n submodule |
| `RALPH_PRD_JSON` | (required) | Relative to `RALPH_REPO_PATH` — e.g. `scripts/ralph/prd.json` |
| `RALPH_STATE_FILE` | (required) | Relative to `RALPH_N8N_PATH` — e.g. `tools/scripts/ralph-bridge.state.json` |

For this phase, only the above variables are needed. Future phases will add more.

______________________________________________________________________

## State file management

### State shape

```json
{
  "running": false,
  "jobId": null,
  "iteration": 0,
  "maxIterations": 10,
  "startedAt": null,
  "tool": null,
  "callbackUrl": null,
  "childPid": null
}
```

### State helpers to implement

```js
const STATE_PATH = path.join(RALPH_N8N_PATH, RALPH_STATE_FILE);

function loadState() {
  // Read STATE_PATH, parse JSON, return object
  // If file doesn't exist, return default state (all fields at initial values)
  // If file is corrupted/invalid JSON, log warning and return default state
}

function saveState(state) {
  // Write state to STATE_PATH as pretty-printed JSON
  // Use writeFileSync for simplicity (state transitions are infrequent)
}
```

The default state (used when file is missing or corrupt):

```json
{
  "running": false,
  "jobId": null,
  "iteration": 0,
  "maxIterations": 10,
  "startedAt": null,
  "tool": null,
  "callbackUrl": null,
  "childPid": null
}
```

______________________________________________________________________

## HTTP server

Create an HTTP server using `node:http`. Listen on `0.0.0.0:${RALPH_BRIDGE_PORT}`.

### Request routing

Simple URL + method matching. For unrecognized routes, return 404:

```json
{ "error": "not_found", "message": "Unknown endpoint" }
```

All responses are `Content-Type: application/json`.

### Helper: parse JSON body

For POST endpoints, read the request body and parse as JSON:

```js
function parseBody(req) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    req.on('data', chunk => chunks.push(chunk));
    req.on('end', () => {
      try {
        resolve(JSON.parse(Buffer.concat(chunks).toString()));
      } catch (e) {
        reject(new Error('Invalid JSON'));
      }
    });
    req.on('error', reject);
  });
}
```

### Helper: send JSON response

```js
function sendJson(res, statusCode, data) {
  res.writeHead(statusCode, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify(data));
}
```

______________________________________________________________________

## Endpoints to implement

### GET /status

Returns the current state object.

```js
// Handler:
const state = loadState();
sendJson(res, 200, state);
```

### GET /prd.json

Reads and returns the PRD file.

```js
// Handler:
const prdPath = path.join(RALPH_REPO_PATH, RALPH_PRD_JSON);
try {
  const content = fs.readFileSync(prdPath, 'utf-8');
  const prd = JSON.parse(content);
  sendJson(res, 200, prd);
} catch (err) {
  sendJson(res, 500, {
    error: 'read_failed',
    message: `Could not read PRD file at ${RALPH_PRD_JSON}: ${err.message}`
  });
}
```

### POST /reset

Resets the iteration counter and job metadata. Returns 409 if currently running.

```js
// Handler:
const state = loadState();

if (state.running) {
  sendJson(res, 409, {
    error: 'conflict',
    message: 'Cannot reset while running. Call POST /abort first.'
  });
  return;
}

// Reset all fields except maxIterations (preserved for reference)
state.iteration = 0;
state.jobId = null;
state.startedAt = null;
state.tool = null;
state.callbackUrl = null;
state.childPid = null;
// running is already false
saveState(state);

sendJson(res, 200, { status: 'reset' });
```

______________________________________________________________________

## Startup behavior

On startup:

1. Load state from file (or create default)
1. Log: `ralph-bridge listening on 0.0.0.0:${port}`
1. Log: `state: running=${state.running}, iteration=${state.iteration}`

For this phase, do NOT implement PID liveness checking (that comes in Phase 4).
Just load state and start the server.

______________________________________________________________________

## Logging

Use `console.log` for info, `console.error` for errors. Include timestamps:

```js
function log(msg) {
  console.log(`[${new Date().toISOString()}] ${msg}`);
}
function logError(msg) {
  console.error(`[${new Date().toISOString()}] ERROR: ${msg}`);
}
```

Log every incoming request: `[timestamp] METHOD /path`

______________________________________________________________________

## Verification

### Test 1: Startup and GET /status

```bash
# Set required env vars
export RALPH_BRIDGE_PORT=8765
export RALPH_REPO_PATH=/home/nntin/git/gSnake
export RALPH_N8N_PATH=/home/nntin/git/gSnake/gsnake-n8n
export RALPH_PRD_JSON=scripts/ralph/prd.json
export RALPH_STATE_FILE=tools/scripts/ralph-bridge.state.json

# Remove any existing state file to test fresh start
rm -f $RALPH_N8N_PATH/$RALPH_STATE_FILE

# Start bridge
node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 1

# Test: GET /status returns default state
curl -s http://localhost:8765/status | jq .
# Expected: { "running": false, "iteration": 0, "maxIterations": 10, ... }
```

### Test 2: GET /prd.json returns parsed PRD

```bash
curl -s http://localhost:8765/prd.json | jq '.userStories | length'
# Expected: a number (the count of user stories in prd.json)

curl -s http://localhost:8765/prd.json | jq '.project'
# Expected: "gSnake"
```

### Test 3: POST /reset resets iteration

```bash
# First, manually create a state with iteration > 0
cat > $RALPH_N8N_PATH/$RALPH_STATE_FILE << 'EOF'
{"running":false,"jobId":"old-job","iteration":15,"maxIterations":20,"startedAt":"2026-02-21T10:00:00Z","tool":"claude","callbackUrl":"http://example.com","childPid":null}
EOF

# Restart bridge to load the state
kill $BRIDGE_PID; sleep 1
node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 1

# Verify state loaded
curl -s http://localhost:8765/status | jq '.iteration'
# Expected: 15

# Reset
curl -s -X POST http://localhost:8765/reset | jq .
# Expected: { "status": "reset" }

# Verify reset
curl -s http://localhost:8765/status | jq '{iteration, running, jobId}'
# Expected: { "iteration": 0, "running": false, "jobId": null }
```

### Test 4: POST /reset rejected when running

```bash
# Manually set running state
cat > $RALPH_N8N_PATH/$RALPH_STATE_FILE << 'EOF'
{"running":true,"jobId":"active-job","iteration":3,"maxIterations":20,"startedAt":"2026-02-21T10:00:00Z","tool":"claude","callbackUrl":"http://example.com","childPid":12345}
EOF

kill $BRIDGE_PID; sleep 1
node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 1

curl -s -X POST http://localhost:8765/reset -w "\nHTTP %{http_code}\n"
# Expected: { "error": "conflict", "message": "Cannot reset while running..." }
# Expected: HTTP 409
```

### Test 5: State persists across restart

```bash
# After test 3, state file should exist with reset values
kill $BRIDGE_PID; sleep 1
node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 1

curl -s http://localhost:8765/status | jq '.iteration'
# Expected: 0 (persisted from the reset)

# Cleanup
kill $BRIDGE_PID
```

### Test 6: Unknown route returns 404

```bash
curl -s http://localhost:8765/unknown -w "\nHTTP %{http_code}\n"
# Expected: { "error": "not_found", ... }
# Expected: HTTP 404
```

### Test 7: Corrupt state file handled gracefully

```bash
echo "not json" > $RALPH_N8N_PATH/$RALPH_STATE_FILE
node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 1

curl -s http://localhost:8765/status | jq '.running'
# Expected: false (default state)

kill $BRIDGE_PID
```

______________________________________________________________________

## Code structure suggestion

```
#!/usr/bin/env node

import http from 'node:http';
import fs from 'node:fs';
import path from 'node:path';

// --- Config ---
const PORT = parseInt(process.env.RALPH_BRIDGE_PORT || '8765', 10);
const REPO_PATH = process.env.RALPH_REPO_PATH;
const N8N_PATH = process.env.RALPH_N8N_PATH;
const PRD_JSON = process.env.RALPH_PRD_JSON;
const STATE_FILE = process.env.RALPH_STATE_FILE;
// Validate required vars...

// --- State helpers ---
// loadState(), saveState()

// --- HTTP helpers ---
// parseBody(), sendJson(), log()

// --- Route handlers ---
// handleGetStatus(), handleGetPrd(), handlePostReset()

// --- Server ---
const server = http.createServer(async (req, res) => {
  log(`${req.method} ${req.url}`);
  // Route to handlers...
});

// --- Startup ---
const state = loadState();
log(`state: running=${state.running}, iteration=${state.iteration}`);
server.listen(PORT, '0.0.0.0', () => {
  log(`ralph-bridge listening on 0.0.0.0:${PORT}`);
});
```

Keep the code flat and simple. Avoid classes or complex abstractions. This is a ~150-line
file at this phase.
