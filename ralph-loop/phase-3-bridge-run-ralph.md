# Phase 3: ralph-bridge.js — POST /run-ralph + Exit Handler + Callback

**Depends on:** Phase 2 (ralph-bridge.js must exist with state management)
**Produces:** Updated `gsnake-n8n/tools/scripts/ralph-bridge.js` with process spawning

______________________________________________________________________

## Overview

Add the core iteration engine to ralph-bridge.js: the `POST /run-ralph` endpoint that
spawns AI CLI processes, streams their output to archive logs, and POSTs a callback to
n8n when they exit.

**Working directory:** `~/git/gSnake/gsnake-n8n`

______________________________________________________________________

## New environment variables

These must be added to the env var loading (in addition to Phase 2 vars):

| Variable | Default | Description |
|----------|---------|-------------|
| `RALPH_CLAUDE_MD` | (required) | Relative to `RALPH_REPO_PATH` — e.g. `scripts/ralph/CLAUDE.md` |
| `RALPH_ARCHIVE_DIR` | (required) | Relative to `RALPH_REPO_PATH` — e.g. `scripts/ralph/archive` |
| `RALPH_WEBHOOK_TOKEN` | (required) | Bearer token for callback POST to n8n |

Note: `RALPH_ITERATION_TIMEOUT` is used in Phase 4. Do NOT implement timeout in this phase.

______________________________________________________________________

## POST /run-ralph endpoint

### Request body

```json
{
  "tool": "claude",
  "callbackUrl": "https://n8n.labs.lair.nntin.xyz/webhook/ralph",
  "maxIterations": 20
}
```

All three fields are required.

### Evaluation order (critical — check before increment)

```js
async function handlePostRunRalph(req, res) {
  const body = await parseBody(req);
  const { tool, callbackUrl, maxIterations } = body;

  // Validate required fields
  if (!tool || !callbackUrl || !maxIterations) {
    return sendJson(res, 400, {
      error: 'bad_request',
      message: 'Missing required fields: tool, callbackUrl, maxIterations'
    });
  }
  if (!['claude', 'codex'].includes(tool)) {
    return sendJson(res, 400, {
      error: 'bad_request',
      message: 'tool must be "claude" or "codex"'
    });
  }

  const state = loadState();

  // Step 1: Check if already running
  if (state.running) {
    return sendJson(res, 200, {
      status: 'already_running',
      jobId: state.jobId
    });
  }

  // Step 2: Update maxIterations from request, THEN check cap
  state.maxIterations = maxIterations;
  if (state.iteration >= maxIterations) {
    return sendJson(res, 200, {
      status: 'max_iterations_reached',
      iteration: state.iteration,
      maxIterations: state.maxIterations
    });
  }

  // Step 3: Increment iteration and set running state
  const jobId = crypto.randomUUID();
  state.iteration += 1;
  state.running = true;
  state.jobId = jobId;
  state.startedAt = new Date().toISOString();
  state.tool = tool;
  state.callbackUrl = callbackUrl;

  // Step 4: Spawn CLI (sets childPid, saves state)
  spawnCli(state, jobId);

  // Step 5: Respond immediately (async — n8n execution ends here)
  sendJson(res, 200, { status: 'started', jobId });
}
```

______________________________________________________________________

## CLI spawning

### Tool invocation commands

```bash
# claude
claude --dangerously-skip-permissions --no-session-persistence --print "$PROMPT" </dev/null

# codex
codex exec --dangerously-bypass-approvals-and-sandbox "$PROMPT" </dev/null
```

### Implementation

```js
import { spawn } from 'node:child_process';

function spawnCli(state, jobId) {
  // Read prompt file
  const promptPath = path.join(REPO_PATH, RALPH_CLAUDE_MD);
  let prompt;
  try {
    prompt = fs.readFileSync(promptPath, 'utf-8');
  } catch (err) {
    logError(`Failed to read prompt file: ${err.message}`);
    // Immediately fail — send callback with error
    state.running = false;
    saveState(state);
    sendCallback(state, { success: false, exitCode: null, timedOut: false, aborted: false });
    return;
  }

  // Build command + args
  let cmd, args;
  if (state.tool === 'claude') {
    cmd = 'claude';
    args = ['--dangerously-skip-permissions', '--no-session-persistence', '--print', prompt];
  } else {
    cmd = 'codex';
    args = ['exec', '--dangerously-bypass-approvals-and-sandbox', prompt];
  }

  // Create archive log file
  const archiveDir = path.join(REPO_PATH, RALPH_ARCHIVE_DIR);
  fs.mkdirSync(archiveDir, { recursive: true });
  const timestamp = new Date().toISOString().replace(/[:.]/g, '').replace('T', '_').slice(0, 15);
  const logFilename = `iteration_${state.iteration}_${timestamp}.log`;
  const logPath = path.join(archiveDir, logFilename);
  const logStream = fs.createWriteStream(logPath, { flags: 'a' });

  log(`Spawning ${cmd} for iteration ${state.iteration} (job ${jobId})`);
  log(`Archive log: ${logPath}`);

  // Spawn process
  const child = spawn(cmd, args, {
    cwd: REPO_PATH,
    env: { ...process.env },  // inherit full env (PATH, HOME, etc.)
    stdio: ['pipe', 'pipe', 'pipe'],
  });

  // Close stdin immediately (equivalent to </dev/null)
  child.stdin.end();

  // Stream stdout+stderr to archive log
  child.stdout.pipe(logStream);
  child.stderr.pipe(logStream);

  // Persist childPid
  state.childPid = child.pid;
  saveState(state);

  // Set up exit handler
  child.on('close', (exitCode, signal) => {
    log(`CLI exited: exitCode=${exitCode}, signal=${signal}, job=${jobId}`);

    logStream.end();

    const currentState = loadState();
    // Only handle if this is still our job (guard against stale handlers)
    if (currentState.jobId !== jobId) {
      log(`Ignoring exit for stale job ${jobId} (current: ${currentState.jobId})`);
      return;
    }

    currentState.running = false;
    currentState.childPid = null;
    saveState(currentState);

    const success = exitCode === 0;
    const relativeLogFile = path.join(RALPH_ARCHIVE_DIR, logFilename);

    sendCallback(currentState, {
      success,
      exitCode: exitCode ?? null,
      timedOut: false,
      aborted: false,
      logFile: relativeLogFile,
    });
  });

  child.on('error', (err) => {
    logError(`Failed to spawn ${cmd}: ${err.message}`);
    logStream.end();

    const currentState = loadState();
    if (currentState.jobId !== jobId) return;

    currentState.running = false;
    currentState.childPid = null;
    saveState(currentState);

    sendCallback(currentState, {
      success: false,
      exitCode: null,
      timedOut: false,
      aborted: false,
    });
  });
}
```

______________________________________________________________________

## Callback POST

When the CLI exits, the bridge POSTs to the `callbackUrl` stored in state.

### Callback payload

```json
{
  "action": "done",
  "jobId": "uuid",
  "iteration": 3,
  "tool": "claude",
  "success": true,
  "callbackUrl": "https://n8n.labs.lair.nntin.xyz/webhook/ralph",
  "maxIterations": 20,
  "exitCode": 0,
  "timedOut": false,
  "aborted": false,
  "logFile": "scripts/ralph/archive/iteration_3_20260221_100000.log"
}
```

**Critical:** The payload echoes `tool`, `callbackUrl`, and `maxIterations` so n8n can
thread these across the async gap to the next `/run-ralph` call.

### Implementation

```js
import https from 'node:https';
import http from 'node:http';

function sendCallback(state, result) {
  const payload = {
    action: 'done',
    jobId: state.jobId,
    iteration: state.iteration,
    tool: state.tool,
    success: result.success,
    callbackUrl: state.callbackUrl,
    maxIterations: state.maxIterations,
    exitCode: result.exitCode ?? null,
    timedOut: result.timedOut ?? false,
    aborted: result.aborted ?? false,
    logFile: result.logFile ?? null,
  };

  const body = JSON.stringify(payload);
  const url = new URL(state.callbackUrl);
  const transport = url.protocol === 'https:' ? https : http;

  const options = {
    hostname: url.hostname,
    port: url.port || (url.protocol === 'https:' ? 443 : 80),
    path: url.pathname + url.search,
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Content-Length': Buffer.byteLength(body),
      'Authorization': `Bearer ${RALPH_WEBHOOK_TOKEN}`,
    },
  };

  log(`Sending callback to ${state.callbackUrl}: success=${result.success}`);

  const req = transport.request(options, (res) => {
    let responseBody = '';
    res.on('data', chunk => responseBody += chunk);
    res.on('end', () => {
      if (res.statusCode >= 200 && res.statusCode < 300) {
        log(`Callback accepted (${res.statusCode})`);
      } else {
        logError(`Callback rejected (${res.statusCode}): ${responseBody}`);
      }
    });
  });

  req.on('error', (err) => {
    logError(`Callback failed: ${err.message}`);
    // Do NOT retry — n8n operator can check bridge status manually
    // The state file shows running=false, so the system is consistent
  });

  req.write(body);
  req.end();
}
```

______________________________________________________________________

## In-memory process tracking

Keep a reference to the active child process in a module-level variable:

```js
let activeChild = null;  // Set in spawnCli, cleared on exit
```

This is needed for Phase 4 (abort sends SIGTERM to this reference) and avoids reading
PID from state file. Store it alongside spawnCli:

```js
// In spawnCli, after spawn:
activeChild = child;

// In close handler:
activeChild = null;
```

Export or make accessible for the future `/abort` handler.

______________________________________________________________________

## Verification

### Test 1: /run-ralph with real CLI (quick test)

For a quick smoke test, temporarily override the tool command to just echo:

```bash
# Create a minimal test: spawn 'echo' instead of 'claude'
# Option A: test with a mock prompt
echo "Just echo hello" > /tmp/test-prompt.md
export RALPH_CLAUDE_MD=/tmp/test-prompt.md  # won't work — needs relative path

# Option B: use the real claude CLI with a trivial prompt
# Ensure scripts/ralph/CLAUDE.md exists and claude is installed
```

Better approach — use a callback catcher:

```bash
# Terminal 1: Start a callback listener
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
http.server.HTTPServer(('', 9000), H).serve_forever()
" &
LISTENER_PID=$!

# Terminal 2: Start bridge (with all env vars)
node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 1

# Send /run-ralph
curl -s -X POST http://localhost:8765/run-ralph \
  -H 'Content-Type: application/json' \
  -d '{
    "tool": "claude",
    "callbackUrl": "http://localhost:9000/callback",
    "maxIterations": 5
  }' | jq .
# Expected: { "status": "started", "jobId": "..." }
```

Wait for the CLI to finish (may take a while with real claude). Check:

- Archive log file created in `scripts/ralph/archive/`
- Callback received at the listener with `success: true` (if claude exits 0)
- State shows `running: false` after completion

### Test 2: /run-ralph while already running

```bash
# Immediately after starting one:
curl -s -X POST http://localhost:8765/run-ralph \
  -H 'Content-Type: application/json' \
  -d '{"tool":"claude","callbackUrl":"http://localhost:9000/cb","maxIterations":5}' | jq .
# Expected: { "status": "already_running", "jobId": "..." }
```

### Test 3: max_iterations_reached

```bash
# Set state to iteration=5, maxIterations=5 via /reset then manual edit,
# or just run enough iterations. Simpler: manually set state:
kill $BRIDGE_PID; sleep 1
cat > $RALPH_N8N_PATH/$RALPH_STATE_FILE << 'EOF'
{"running":false,"jobId":null,"iteration":5,"maxIterations":5,"startedAt":null,"tool":null,"callbackUrl":null,"childPid":null}
EOF
node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js &
BRIDGE_PID=$!
sleep 1

curl -s -X POST http://localhost:8765/run-ralph \
  -H 'Content-Type: application/json' \
  -d '{"tool":"claude","callbackUrl":"http://localhost:9000/cb","maxIterations":5}' | jq .
# Expected: { "status": "max_iterations_reached", "iteration": 5, "maxIterations": 5 }
```

### Test 4: Callback includes threaded fields

From the callback listener output in Test 1, verify the payload includes:

```json
{
  "action": "done",
  "tool": "claude",
  "callbackUrl": "http://localhost:9000/callback",
  "maxIterations": 5,
  "success": true,
  "exitCode": 0,
  "timedOut": false,
  "aborted": false,
  "logFile": "scripts/ralph/archive/iteration_1_..."
}
```

All of `tool`, `callbackUrl`, `maxIterations` must be present.

### Test 5: Archive log exists and has content

```bash
ls -la scripts/ralph/archive/
# Expected: iteration_1_*.log file exists

wc -l scripts/ralph/archive/iteration_1_*.log
# Expected: non-zero (contains CLI stdout+stderr)
```

### Test 6: Callback includes Authorization header

Check the callback listener for the `Authorization` header:

```bash
# Update the Python listener to also print headers:
# print(dict(self.headers))
# Verify: Authorization: Bearer <token value>
```

### Test 7: Invalid request body

```bash
curl -s -X POST http://localhost:8765/run-ralph \
  -H 'Content-Type: application/json' \
  -d '{"tool":"invalid"}' -w "\nHTTP %{http_code}\n"
# Expected: 400 with error message about missing fields or invalid tool
```

______________________________________________________________________

## Cleanup

After all tests:

```bash
kill $BRIDGE_PID $LISTENER_PID 2>/dev/null
rm -f $RALPH_N8N_PATH/$RALPH_STATE_FILE
```
