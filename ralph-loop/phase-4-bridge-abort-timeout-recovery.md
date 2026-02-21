# Phase 4: ralph-bridge.js — POST /abort + Timeout + Startup Recovery

**Depends on:** Phase 3 (ralph-bridge.js must have /run-ralph with process spawning)
**Produces:** Updated `gsnake-n8n/tools/scripts/ralph-bridge.js` (feature-complete bridge)

______________________________________________________________________

## Overview

Add the remaining bridge features: aborting in-flight iterations, automatic timeout
enforcement, and crash recovery on startup. After this phase, the bridge is fully
functional and ready for integration with n8n.

**Working directory:** `~/git/gSnake/gsnake-n8n`

______________________________________________________________________

## New environment variable

| Variable | Default | Description |
|----------|---------|-------------|
| `RALPH_ITERATION_TIMEOUT` | `18000` | **Seconds** per iteration. Bridge multiplies by 1000 for `setTimeout`. |

______________________________________________________________________

## POST /abort endpoint

Sends SIGTERM to the in-flight CLI process. If the process hasn't exited after 5 seconds,
sends SIGKILL. Returns immediately — the callback is sent asynchronously after exit.

### Implementation

```js
function handlePostAbort(req, res) {
  const state = loadState();

  if (!state.running) {
    return sendJson(res, 200, { status: 'idle', jobId: null });
  }

  const jobId = state.jobId;
  log(`Aborting job ${jobId} (PID ${state.childPid})`);

  // Mark as aborting (for the exit handler to know)
  abortRequested = true;

  if (activeChild) {
    // Send SIGTERM
    activeChild.kill('SIGTERM');

    // Escalate to SIGKILL after 5 seconds if still alive
    const killTimer = setTimeout(() => {
      if (activeChild && !activeChild.killed) {
        log(`SIGKILL escalation for job ${jobId}`);
        activeChild.kill('SIGKILL');
      }
    }, 5000);

    // Clean up the kill timer when process exits
    activeChild.once('close', () => clearTimeout(killTimer));
  } else if (state.childPid) {
    // activeChild is null but state says running — try direct PID kill
    try {
      process.kill(state.childPid, 'SIGTERM');
      setTimeout(() => {
        try { process.kill(state.childPid, 'SIGKILL'); } catch (_) {}
      }, 5000);
    } catch (err) {
      log(`PID ${state.childPid} already dead: ${err.message}`);
    }
  }

  sendJson(res, 200, { status: 'aborting', jobId });
}
```

### Module-level abort flag

Add a module-level flag that the exit handler checks:

```js
let abortRequested = false;
```

In the `spawnCli` close handler (from Phase 3), incorporate the abort flag:

```js
child.on('close', (exitCode, signal) => {
  // ... existing logic ...

  const wasAborted = abortRequested;
  abortRequested = false;  // Reset for next iteration

  const success = !wasAborted && exitCode === 0;

  sendCallback(currentState, {
    success,
    exitCode: exitCode ?? null,
    timedOut: false,
    aborted: wasAborted,
    logFile: relativeLogFile,
  });
});
```

______________________________________________________________________

## Iteration timeout

Each iteration has a wall-clock timeout. When exceeded, the bridge kills the CLI
process and sends a callback with `timedOut: true`.

### Implementation

Add a module-level timeout handle:

```js
let iterationTimer = null;
```

In `spawnCli`, after spawning the process, set up the timeout:

```js
// In spawnCli, after child = spawn(...)

const timeoutSeconds = parseInt(process.env.RALPH_ITERATION_TIMEOUT || '18000', 10);
const timeoutMs = timeoutSeconds * 1000;  // CRITICAL: seconds → milliseconds

log(`Iteration timeout set: ${timeoutSeconds}s (${timeoutMs}ms)`);

iterationTimer = setTimeout(() => {
  if (activeChild && !activeChild.killed) {
    log(`Timeout reached for job ${state.jobId} after ${timeoutSeconds}s`);
    timedOut = true;  // Module-level flag

    activeChild.kill('SIGTERM');
    setTimeout(() => {
      if (activeChild && !activeChild.killed) {
        log(`SIGKILL escalation after timeout for job ${state.jobId}`);
        activeChild.kill('SIGKILL');
      }
    }, 5000);
  }
}, timeoutMs);
```

Add a module-level timeout flag:

```js
let timedOut = false;
```

In the close handler, incorporate the timeout flag:

```js
child.on('close', (exitCode, signal) => {
  // Clear the iteration timer
  if (iterationTimer) {
    clearTimeout(iterationTimer);
    iterationTimer = null;
  }

  const wasAborted = abortRequested;
  const wasTimedOut = timedOut;
  abortRequested = false;
  timedOut = false;
  activeChild = null;

  // ... rest of existing close handler ...

  const success = !wasAborted && !wasTimedOut && exitCode === 0;

  sendCallback(currentState, {
    success,
    exitCode: wasTimedOut ? 124 : (exitCode ?? null),  // 124 = timeout convention
    timedOut: wasTimedOut,
    aborted: wasAborted,
    logFile: relativeLogFile,
  });
});
```

______________________________________________________________________

## Startup PID liveness check

When the bridge starts and finds `running: true` in state.json, it must check whether
the process is actually alive. If the process is dead (bridge crashed mid-iteration),
it sends a failure callback and resets state.

### Implementation

Add to the startup sequence (after `loadState()`, before `server.listen()`):

```js
function checkOrphanedProcess(state) {
  if (!state.running) return;  // Nothing to check

  const pid = state.childPid;
  if (!pid) {
    log('State shows running=true but no childPid — resetting');
    resetOrphanedState(state);
    return;
  }

  // Check if process is alive
  try {
    process.kill(pid, 0);  // Signal 0 = liveness check only
    log(`Process ${pid} is still alive — re-attaching is not supported. Killing.`);
    // We can't re-attach stdout/stderr to a running process,
    // so we kill it and send a failure callback.
    try {
      process.kill(pid, 'SIGTERM');
      setTimeout(() => {
        try { process.kill(pid, 'SIGKILL'); } catch (_) {}
      }, 5000);
    } catch (_) {}
    // Wait a moment then reset
    setTimeout(() => {
      resetOrphanedState(state);
    }, 6000);
  } catch (err) {
    // Process is dead — reset state and send callback
    log(`Orphaned process ${pid} is dead — sending failure callback`);
    resetOrphanedState(state);
  }
}

function resetOrphanedState(state) {
  state.running = false;
  state.childPid = null;
  saveState(state);

  if (state.callbackUrl) {
    sendCallback(state, {
      success: false,
      exitCode: null,
      timedOut: false,
      aborted: false,
      logFile: null,
    });
  } else {
    log('No callbackUrl stored — cannot notify n8n of orphaned process');
  }
}
```

Call `checkOrphanedProcess(state)` during startup, before starting the HTTP server.

______________________________________________________________________

## Verification

### Test 1: POST /abort kills running process

```bash
# Start bridge
node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 1

# Start callback listener (reuse from Phase 3)
python3 -c "
import http.server, json
class H(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers['Content-Length'] or 0)
        body = json.loads(self.rfile.read(length))
        print(json.dumps(body, indent=2))
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'{\"status\":\"accepted\"}')
    def log_message(self, *args): pass
http.server.HTTPServer(('', 9000), H).serve_forever()
" &
LISTENER_PID=$!

# Start an iteration (use real claude or a long-running mock)
# For testing, you can temporarily modify the bridge to spawn 'sleep 300' instead
curl -s -X POST http://localhost:8765/run-ralph \
  -H 'Content-Type: application/json' \
  -d '{"tool":"claude","callbackUrl":"http://localhost:9000/callback","maxIterations":5}' | jq .

# Wait a moment, then abort
sleep 2
curl -s -X POST http://localhost:8765/abort | jq .
# Expected: { "status": "aborting", "jobId": "..." }

# Wait for callback
sleep 3
# Expected callback in listener: { "success": false, "aborted": true, ... }

# Status should show not running
curl -s http://localhost:8765/status | jq '.running'
# Expected: false
```

### Test 2: POST /abort when idle

```bash
curl -s -X POST http://localhost:8765/abort | jq .
# Expected: { "status": "idle", "jobId": null }
```

### Test 3: Timeout fires

For this test, set a very short timeout:

```bash
kill $BRIDGE_PID; sleep 1
export RALPH_ITERATION_TIMEOUT=5  # 5 seconds

node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 1

# Start an iteration (claude will take longer than 5 seconds)
curl -s -X POST http://localhost:8765/run-ralph \
  -H 'Content-Type: application/json' \
  -d '{"tool":"claude","callbackUrl":"http://localhost:9000/callback","maxIterations":5}' | jq .

# Wait for timeout + kill
sleep 12

# Expected callback: { "success": false, "timedOut": true, "exitCode": 124, ... }
curl -s http://localhost:8765/status | jq '.running'
# Expected: false
```

### Test 4: Startup recovery — dead orphan

```bash
kill $BRIDGE_PID; sleep 1

# Manually write state simulating a crash (dead PID)
cat > $RALPH_N8N_PATH/$RALPH_STATE_FILE << 'EOF'
{"running":true,"jobId":"orphan-job","iteration":3,"maxIterations":20,"startedAt":"2026-02-21T10:00:00Z","tool":"claude","callbackUrl":"http://localhost:9000/callback","childPid":999999}
EOF

# Start bridge — should detect dead PID and send failure callback
node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 2

# Expected in bridge logs: "Orphaned process 999999 is dead — sending failure callback"
# Expected callback: { "success": false, "aborted": false, "timedOut": false, ... }

curl -s http://localhost:8765/status | jq '{running, iteration}'
# Expected: { "running": false, "iteration": 3 }
```

### Test 5: Startup recovery — no callbackUrl

```bash
kill $BRIDGE_PID; sleep 1

# State with running=true but no callbackUrl
cat > $RALPH_N8N_PATH/$RALPH_STATE_FILE << 'EOF'
{"running":true,"jobId":"orphan-job","iteration":3,"maxIterations":20,"startedAt":"2026-02-21T10:00:00Z","tool":"claude","callbackUrl":null,"childPid":999999}
EOF

node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 2

# Expected in bridge logs: "No callbackUrl stored — cannot notify n8n"
curl -s http://localhost:8765/status | jq '.running'
# Expected: false (state was reset even without callback)
```

### Test 6: SIGTERM → SIGKILL escalation

Test that if SIGTERM is ignored, SIGKILL follows after 5 seconds:

```bash
# This is hard to test with real tools. One approach:
# Temporarily modify spawnCli to spawn a script that traps SIGTERM:
#   trap '' SIGTERM; sleep 300
# Then verify SIGKILL arrives after 5s and the process dies.
# This is an edge case — manual verification is acceptable.
```

______________________________________________________________________

## Final state of ralph-bridge.js after Phase 4

The bridge should now handle all endpoints:

| Endpoint | Method | Status |
|----------|--------|--------|
| `/status` | GET | Phase 2 |
| `/prd.json` | GET | Phase 2 |
| `/reset` | POST | Phase 2 |
| `/run-ralph` | POST | Phase 3 |
| `/abort` | POST | Phase 4 |

Plus:

- Iteration timeout enforcement (Phase 4)
- Startup PID liveness check + orphan recovery (Phase 4)
- Callback POSTs with auth header and state threading (Phase 3)

The bridge is now feature-complete and ready for integration with n8n (Phase 5).

______________________________________________________________________

## Cleanup

```bash
kill $BRIDGE_PID $LISTENER_PID 2>/dev/null
rm -f $RALPH_N8N_PATH/$RALPH_STATE_FILE
unset RALPH_ITERATION_TIMEOUT  # Reset to default
```
