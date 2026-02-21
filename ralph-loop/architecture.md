# Ralph Loop Architecture

Visual specification for the ralph-loop n8n implementation.
See `gsnake-n8n/workflows/n8n-workflow/ralph-loop.md` for the full SOP.

______________________________________________________________________

## 1. System Overview

High-level view of all components and their relationships.

```mermaid
graph TB
    subgraph Host["Host Machine"]
        direction TB
        CLI_C["claude CLI"]
        CLI_X["codex CLI"]
        Bridge["ralph-bridge.js
        :8765"]
        Repo["gSnake repo
        prd.json · CLAUDE.md
        archive logs"]
        SD["systemd
        ralph-bridge.service"]

        SD -->|"manages"| Bridge
        Bridge -->|"spawns"| CLI_C
        Bridge -->|"spawns"| CLI_X
        Bridge -->|"reads/writes"| Repo
        CLI_C -->|"git commit + push"| Repo
        CLI_X -->|"git commit + push"| Repo
    end

    subgraph DockerNet["Docker Bridge Network"]
        direction TB
        N8N["n8n container
        n8n.labs.lair.nntin.xyz"]
    end

    subgraph External["External Services"]
        GH["GitHub
        git push · CI"]
        Notification["e.g. Discord notifications"]
    end

    Human["👤 Human
    or automation"] -->|"POST /webhook/ralph
    Authorization: Bearer token
    {action:start}"| N8N
    N8N -->|"GET /status
    GET /prd.json
    POST /reset
    POST /run-ralph
    (host-gateway:8765)"| Bridge
    Bridge -->|"POST /webhook/ralph
    Authorization: Bearer token
    {action:done}"| N8N
    N8N -->|"workflow execute:<br>n8n notification flow"| Notification
    Repo -->|"git push"| GH
```

______________________________________________________________________

## 2. Bridge HTTP API

The bridge service acts as a thin execution proxy between n8n and the host CLIs.

```mermaid
graph LR
    N8N["n8n"]

    subgraph Bridge["ralph-bridge.js :8765"]
        S["/status
        GET"]
        P["/prd.json
        GET"]
        R["/run-ralph
        POST"]
        RST["/reset
        POST"]
        AB["/abort
        POST"]
        State["state.json
        {running, iteration,
        jobId, tool,
        callbackUrl,
        childPid, ...}"]
        EH["exit handler
        (spawned process 'close' event)"]
    end

    subgraph HostProcess["Host Process"]
        Claude["claude CLI"]
        Codex["codex CLI"]
        Log["archive/iteration_N_*.log"]
    end

    N8N -->|"check if busy"| S
    N8N -->|"read PRD"| P
    N8N -->|"reset counter (start path)"| RST
    N8N -->|"start iteration"| R
    N8N -->|"abort in-flight (manual)"| AB

    S --> State
    R --> State
    RST --> State
    AB --> State

    R -->|"async spawn"| Claude
    R -->|"async spawn"| Codex
    Claude -->|"stdout+stderr"| Log
    Codex -->|"stdout+stderr"| Log

    Claude -->|"process exit"| EH
    Codex -->|"process exit"| EH
    AB -->|"SIGTERM → SIGKILL"| Claude
    AB -->|"SIGTERM → SIGKILL"| Codex
    EH --> State
    EH -->|"POST /webhook/ralph
    Authorization: Bearer token
    {action:done, success, jobId}"| N8N

    P -->|"reads"| PRD["scripts/ralph/prd.json"]
```

______________________________________________________________________

## 3. n8n Webhook State Machine

The n8n workflow is driven entirely by Execute Workflow callbacks. There is no polling or sleep.

```mermaid
stateDiagram-v2
    [*] --> Idle

    Idle --> CheckStatus : Execute Workflow trigger<br>action "start"
    CheckStatus --> AlreadyRunning : running == true
    CheckStatus --> CheckPRD : running == false

    AlreadyRunning --> [*] : Notification "already running" (info)

    CheckPRD --> Done_NothingToDo : all passes == true
    CheckPRD --> ResetCounter : remaining stories > 0

    Done_NothingToDo --> [*] : Notification "nothing to do" (success)

    ResetCounter --> StartIteration : POST /reset → ok
    ResetCounter --> Done_Error : POST /reset → error

    StartIteration --> InFlight : POST /run-ralph → status "started"
    StartIteration --> Done_MaxHit : POST /run-ralph → status "max_iterations_reached"
    StartIteration --> Done_AlreadyRunning : POST /run-ralph → status "already_running"
    StartIteration --> Done_Error : POST /run-ralph → network/bridge error

    Done_MaxHit --> [*] : Notification "max iterations reached" (warning)
    Done_AlreadyRunning --> [*] : Notification "already running" (info)
    Done_Error --> [*] : Notification "error" (error)

    InFlight --> CheckStatus2 : Execute Workflow trigger<br>action "done", success true
    InFlight --> Done_IterationFailed : Execute Workflow trigger<br>action "done", success false

    Done_IterationFailed --> [*] : Notification "iteration failed/aborted" (error/warning)

    CheckStatus2 --> CheckPRD2 : running == false
    CheckPRD2 --> Done_AllComplete : all passes == true
    CheckPRD2 --> NextIteration : remaining stories > 0

    Done_AllComplete --> [*] : Notification "all stories complete!" (success)

    NextIteration --> InFlight : POST /run-ralph → status "started"
    NextIteration --> Done_MaxHit : POST /run-ralph → status "max_iterations_reached"
    NextIteration --> Done_AlreadyRunning : POST /run-ralph → status "already_running"
    NextIteration --> Done_Error : POST /run-ralph → error

    note right of InFlight
        n8n execution ends here.
        Bridge holds state (callbackUrl + childPid persisted).
        Bridge POSTs callback when
        claude/codex exits.
    end note

    note right of ResetCounter
        Only on the "start" path.
        Resets iteration counter to 0.
        Not called on "done" continuation path.
    end note
```

______________________________________________________________________

## 4. Iteration Sequence

One complete iteration from n8n's perspective, showing all HTTP hops and auth headers.

```mermaid
sequenceDiagram
    participant H as 👤 Human
    participant A as ralph-loop-auth<br/>/webhook/ralph
    participant N as ralph-loop<br/>(Execute Workflow)
    participant B as ralph-bridge<br/>:8765
    participant C as claude CLI
    participant G as GitHub

    H->>A: POST {action:"start", tool:"claude", maxIterations:20}<br/>Authorization: Bearer token
    A->>A: Validate token → 202 Accepted
    A->>N: Execute Workflow (async, fire-and-forget)
    N->>B: GET /status
    B-->>N: {running:false, iteration:0}
    N->>B: GET /prd.json
    B-->>N: {userStories:[...3 with passes:false...]}
    N->>B: POST /reset
    B-->>N: {status:"reset"}
    N->>B: POST /run-ralph {tool:"claude", callbackUrl, maxIterations:20}
    B-->>N: {status:"started", jobId:"uuid"}
    Note over B,C: Async — n8n execution ends here

    B->>C: spawn claude --dangerously-skip-permissions<br>--no-session-persistence --print "$PROMPT"
    C->>C: Read prd.json, pick story US-005
    C->>C: Implement story, run tests
    C->>G: git commit + push
    C->>C: Set passes:true in prd.json
    C->>G: git push (prd.json update)
    C->>B: exit 0
    B->>B: state.running=false, iteration++, save state

    B->>A: POST /webhook/ralph {action:"done", jobId, success:true, iteration:1}<br/>Authorization: Bearer token
    A->>A: Validate token → 202 Accepted
    A->>N: Execute Workflow (async)
    N->>B: GET /status
    B-->>N: {running:false, iteration:1}
    N->>B: GET /prd.json
    B-->>N: {userStories:[...2 with passes:false...]}
    N->>B: POST /run-ralph {tool:"claude", callbackUrl, maxIterations:20}
    B-->>N: {status:"started", jobId:"uuid2"}
    Note over B,C: Next iteration begins...
```

______________________________________________________________________

## 5. Bridge State Machine

Internal state transitions inside `ralph-bridge.js`.

```mermaid
stateDiagram-v2
    direction TB

    [*] --> Idle : startup<br>(load state.json + PID liveness check)

    %% --- Core lifecycle ---
    Idle --> Running : POST /run-ralph<br>(not at maxIterations)

    Running --> Success : CLI exits (exit 0)
    Running --> Error : CLI exits (non-zero)
    Running --> Timeout : RALPH_ITERATION_TIMEOUT exceeded
    Running --> Aborted : POST /abort received

    Success --> Idle : POST callbackUrl<br>{success true}
    Error --> Idle : POST callbackUrl<br>{success false}
    Timeout --> Idle : POST callbackUrl<br>{timedOut true}
    Aborted --> Idle : POST callbackUrl<br>{aborted true}

    %% --- Guards / rejections ---
    Idle --> MaxReached : POST /run-ralph<br>{max_iterations_reached}
    MaxReached --> Idle

    Running --> AlreadyRunning : POST /run-ralph<br>{already_running}
    AlreadyRunning --> Running

    %% --- Reset ---
    Idle --> Idle : POST /reset<br>{status reset}
    Running --> RejectReset : POST /reset<br>{409 conflict}
    RejectReset --> Running

    %% --- Abort idempotent ---
    Idle --> Idle : POST /abort<br>{status idle}

    %% --- Read-only operations (grouped) ---
    state "Read-only APIs" as ReadOnly {
        [*] --> Status
        Status --> PRD
    }

    Idle --> ReadOnly : GET /status<br>GET /prd.json
    Running --> ReadOnly : GET /status<br>GET /prd.json
    ReadOnly --> Idle

    %% --- Persistence note ---
    note right of Running
        state persisted to state.json on
        every transition: running, jobId,
        iteration, maxIterations, startedAt,
        tool, callbackUrl, childPid
    end note

    note right of Idle
        On startup: if state.running=true,
        run kill -0 on stored childPid.
        If dead: reset + POST callback
        with {success:false} to callbackUrl.
    end note
```

______________________________________________________________________

## 6. Deployment View

Physical deployment topology and network paths.

```mermaid
graph TB
    subgraph Host["Ubuntu Host (WSL2 / Linux)"]
        subgraph SystemD["systemd"]
            BridgeSvc["ralph-bridge.service<br>→ node $RALPH_N8N_PATH/tools/scripts/ralph-bridge.js<br>  0.0.0.0:8765<br>  EnvironmentFile: ralph-bridge.env"]
        end

        subgraph Repo["~/git/gSnake"]
            CLAUDE_MD["scripts/ralph/CLAUDE.md<br>(agent prompt)"]
            PRD["scripts/ralph/prd.json<br>(story state)"]
            Archive["scripts/ralph/archive/<br>(iteration logs)"]
        end

        subgraph N8NSubmodule["~/git/gSnake/gsnake-n8n  (submodule)"]
            BridgeJS["tools/scripts/ralph-bridge.js"]
            StateFile["tools/scripts/ralph-bridge.state.json<br>(runtime — gitignored)"]
        end

        Claude_Bin["/usr/local/bin/claude"]
        Codex_Bin["/usr/local/bin/codex"]
    end

    subgraph DockerNetwork["nntin-labs-network (Docker bridge)"]
        N8N_Container["n8n container<br>host-gateway resolves<br>to host bridge IP<br>(extra_hosts config)"]
    end

    subgraph CloudServices["Cloud / External"]
        N8N_URL["https://n8n.labs.lair.nntin.xyz<br>(reverse proxy → n8n:5678)"]
        GH["github.com/NNTin/gSnake"]
        NS["Notification Service"]
    end

    BridgeSvc -->|"reads"| CLAUDE_MD
    BridgeSvc -->|"reads/writes"| PRD
    BridgeSvc -->|"writes"| Archive
    BridgeSvc -->|"reads/writes"| StateFile
    BridgeSvc -->|"spawns"| Claude_Bin
    BridgeSvc -->|"spawns"| Codex_Bin
    BridgeSvc -.->|"is"| BridgeJS

    N8N_Container -->|"HTTP :8765<br>via host-gateway"| BridgeSvc
    BridgeSvc -->|"POST callback<br>Bearer token<br>via public URL"| N8N_URL
    N8N_Container -->|"notifies"| NS
    Claude_Bin -->|"git push"| GH
```

______________________________________________________________________

## 7. n8n Workflow Node Graph

Concrete n8n node layout for both action paths. Entry point is the Execute Workflow Trigger
(fed by `ralph-loop-auth`). Full SOP in `gsnake-n8n/workflows/n8n-workflow/ralph-loop.md`.

```mermaid
flowchart TD
    EWT(["Execute Workflow Trigger<br>(called by ralph-loop-auth)<br>receives original request body"])
    SW{{"Switch<br>$json.action"}}

    %% ── start path ──────────────────────────────────────────────────────
    GS["HTTP GET /status<br>host-gateway:8765"]
    IR{{"running == true?"}}
    BUSY(["● Notify(info)<br>already running"])
    GPS["HTTP GET /prd.json<br>host-gateway:8765"]
    CRS["Code: count remaining<br>passes == false"]
    ADS{{"remaining == 0?"}}
    DN(["● Notify(success)<br>nothing to do"])
    RST["HTTP POST /reset<br>host-gateway:8765"]
    PRS["HTTP POST /run-ralph<br>{ tool · maxIterations · callbackUrl }"]
    RJS{{"status?"}}
    DS(["● Notify(info)<br>Ralph started · N remaining"])
    DM_S(["● Notify(warning)<br>max iterations reached"])
    DB_S(["● Notify(info)<br>already running"])

    %% ── done path ───────────────────────────────────────────────────────
    SF{{"$json.success?"}}
    DF(["● Notify(error/warning)<br>iteration failed · aborted"])
    GPD["HTTP GET /prd.json<br>host-gateway:8765"]
    CRD["Code: count remaining"]
    ADD{{"remaining == 0?"}}
    DC(["● Notify(success)<br>all stories complete!"])
    PRD["HTTP POST /run-ralph<br>{ tool · maxIterations · callbackUrl }"]
    RJD{{"status?"}}
    DM(["● Notify(warning)<br>max iterations reached"])
    DI(["● Notify(info)<br>iteration N done · M remaining"])
    DB_D(["● Notify(info)<br>already running"])

    %% ── shared ──────────────────────────────────────────────────────────
    DE(["● Notify(error)<br>error"])
    WAIT(["⏳ n8n execution ends here<br>bridge runs CLI · holds state<br>(callbackUrl + childPid persisted)<br>bridge POSTs callback on exit"])

    %% ── routing ─────────────────────────────────────────────────────────
    EWT --> SW
    SW -- "start" --> GS
    SW -- "done"  --> SF
    SW -- "other" --> DE

    GS --> IR
    IR -- "true"  --> BUSY
    IR -- "false" --> GPS
    GPS --> CRS --> ADS
    ADS -- "yes" --> DN
    ADS -- "no"  --> RST
    RST --> PRS
    PRS --> RJS
    RJS -- "started"               --> DS
    RJS -- "max_iterations_reached" --> DM_S
    RJS -- "already_running"        --> DB_S
    RJS -- "error"                  --> DE

    SF -- "false" --> DF
    SF -- "true"  --> GPD
    GPD --> CRD --> ADD
    ADD -- "yes" --> DC
    ADD -- "no"  --> PRD
    PRD --> RJD
    RJD -- "max_iterations_reached" --> DM
    RJD -- "already_running"        --> DB_D
    RJD -- "error"                  --> DE
    RJD -- "started"                --> DI

    DS --> WAIT
    DI --> WAIT
    WAIT -. "POST {action:'done', success, iteration, tool, exitCode, aborted, ...}" .-> EWT
```

______________________________________________________________________

## 8. Notification Workflow

The `● Notify(...)` terminal nodes in the ralph-loop workflow each trigger a
separate n8n **notification workflow** via an internal Execute Workflow call.
That workflow owns the notification pipeline and can fan out to multiple
channels independently of the ralph-loop logic.

```mermaid
flowchart TD
    %% ── callers (ralph-loop terminal nodes) ─────────────────────────────
    subgraph RalphLoop["ralph-loop workflow (callers)"]
        DN(["● nothing to do"])
        DBUSY(["● already running"])
        DS(["● Ralph started"])
        DI(["● iteration N done"])
        DF(["● iteration failed/aborted"])
        DC(["● all stories complete!"])
        DM(["● max iterations reached"])
        DE(["● error"])
    end

    %% ── notification workflow ────────────────────────────────────────────
    subgraph NotifWorkflow["notification workflow"]
        NW(["Execute Workflow Trigger"])
        FMT["Code: format message<br>(title · body · color · emoji)"]
        SW2{{"Switch<br>channel routing"}}

        subgraph Discord["Discord (active)"]
            DC_NODE["HTTP POST<br>discord webhook URL<br>{embeds: [{title, description, color}]}"]
        end

        subgraph Future["Future channels (inactive)"]
            WA["WhatsApp"]
            TG["Telegram"]
            ETC["…"]
        end

        OK(["● done"])
    end

    %% ── routing ──────────────────────────────────────────────────────────
    DN & DBUSY & DS & DI & DF & DC & DM & DE -->|"Execute Workflow<br>{event, payload, level}"| NW
    NW --> FMT
    FMT --> SW2
    SW2 -- "discord" --> DC_NODE
    SW2 -. "whatsapp (future)" .-> WA
    SW2 -. "telegram (future)" .-> TG
    SW2 -. "other (future)" .-> ETC
    DC_NODE --> OK
```
