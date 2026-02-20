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
    or automation"] -->|"POST /webhook/ralph-loop
    {action:start}"| N8N
    N8N -->|"GET /status
    GET /prd.json
    POST /run-ralph
    (host-gateway:8765)"| Bridge
    Bridge -->|"POST /webhook/ralph-loop
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
        State["state.json
        {running, iteration,
        jobId, tool, ...}"]
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
    N8N -->|"start iteration"| R

    S --> State
    R --> State

    R -->|"async spawn"| Claude
    R -->|"async spawn"| Codex
    Claude -->|"stdout+stderr"| Log
    Codex -->|"stdout+stderr"| Log

    Claude -->|"process exit"| EH
    Codex -->|"process exit"| EH
    EH --> State
    EH -->|"POST /webhook/ralph-loop
    {action:done, success, jobId}"| N8N

    P -->|"reads"| PRD["scripts/ralph/prd.json"]
```

______________________________________________________________________

## 3. n8n Webhook State Machine

The n8n workflow is driven entirely by webhook callbacks. There is no polling or sleep.

```mermaid
stateDiagram-v2
    [*] --> Idle

    Idle --> CheckStatus : POST /webhook/ralph-loop<br>action "start"
    CheckStatus --> Idle : running == true<br>(already busy, no-op)
    CheckStatus --> CheckPRD : running == false

    CheckPRD --> Done_NothingToDo : all passes == true
    CheckPRD --> StartIteration : remaining stories > 0

    Done_NothingToDo --> [*] : Notification "nothing to do"

    StartIteration --> InFlight : POST /run-ralph → status "started"
    StartIteration --> Done_MaxHit : POST /run-ralph → status "max_iterations_reached"
    StartIteration --> Done_Error : POST /run-ralph → network/bridge error

    Done_MaxHit --> [*] : Notification "max iterations reached"
    Done_Error --> [*] : Notification "error"

    InFlight --> CheckStatus : POST /webhook/ralph-loop<br>action "done", success true
    InFlight --> Done_IterationFailed : POST /webhook/ralph-loop<br>action "done", success false

    Done_IterationFailed --> [*] : Notification "iteration failed"

    note right of InFlight
        n8n execution ends here.
        Bridge holds state.
        Bridge POSTs callback when
        claude/codex exits.
    end note
```

______________________________________________________________________

## 4. Iteration Sequence

One complete iteration from n8n's perspective, showing all HTTP hops.

```mermaid
sequenceDiagram
    participant H as 👤 Human
    participant N as n8n<br/>/webhook/ralph-loop
    participant B as ralph-bridge<br/>:8765
    participant C as claude CLI
    participant G as GitHub

    H->>N: POST {action:"start", tool:"claude", maxIterations:20}
    N->>N: Respond 200 immediately (async)
    N->>B: GET /status
    B-->>N: {running:false, iteration:2}
    N->>B: GET /prd.json
    B-->>N: {userStories:[...3 with passes:false...]}
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

    B->>N: POST /webhook/ralph-loop {action:"done", jobId, success:true, iteration:3}
    N->>N: Respond 200
    N->>B: GET /status
    B-->>N: {running:false, iteration:3}
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

    [*] --> Idle : startup<br>(load state.json)

    %% --- Core lifecycle ---
    Idle --> Running : POST /run-ralph<br>(not at maxIterations)

    Running --> Success : CLI exits (success)
    Running --> Error : CLI exits (error)
    Running --> Timeout : RALPH_ITERATION_TIMEOUT exceeded

    Success --> Idle : POST callbackUrl<br>{success  true}
    Error --> Idle : POST callbackUrl<br>{success  false}
    Timeout --> Idle : POST callbackUrl<br>{timedOut  true}

    %% --- Guards / rejections ---
    Idle --> MaxReached : POST /run-ralph<br>{max_iterations_reached}
    MaxReached --> Idle

    Running --> AlreadyRunning : POST /run-ralph<br>{already_running}
    AlreadyRunning --> Running

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
        state persisted to state.json
        on every transition
    end note
```

______________________________________________________________________

## 6. Deployment View

Physical deployment topology and network paths.

```mermaid
graph TB
    subgraph Host["Ubuntu Host (WSL2 / Linux)"]
        subgraph SystemD["systemd"]
            BridgeSvc["ralph-bridge.service<br>→ node ralph-bridge.js<br>  0.0.0.0:8765"]
        end

        subgraph Repo["~/git/gSnake"]
            CLAUDE_MD["scripts/ralph/CLAUDE.md<br>(agent prompt)"]
            PRD["scripts/ralph/prd.json<br>(story state)"]
            Archive["scripts/ralph/archive/<br>(iteration logs)"]
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
    BridgeSvc -->|"spawns"| Claude_Bin
    BridgeSvc -->|"spawns"| Codex_Bin

    N8N_Container -->|"HTTP :8765<br>via host-gateway"| BridgeSvc
    BridgeSvc -->|"POST callback<br>via public URL"| N8N_URL
    N8N_Container -->|"notifies"| NS
    Claude_Bin -->|"git push"| GH
```

______________________________________________________________________

## 7. n8n Workflow Node Graph

Concrete n8n node layout for both action paths. Full SOP in
`gsnake-n8n/workflows/n8n-workflow/ralph-loop.md`.

```mermaid
flowchart TD
    WH(["Webhook<br>POST /webhook/ralph-loop<br>responds 200 immediately"])
    SW{"Switch<br>$json.body.action"}

    %% ── start path ──────────────────────────────────────────────────────
    GS["HTTP GET /status<br>host-gateway:8765"]
    IR{"running == true?"}
    NOOP(["● stop — already busy"])
    GPS["HTTP GET /prd.json<br>host-gateway:8765"]
    CRS["Code: count remaining<br>passes == false"]
    ADS{"remaining == 0?"}
    DN(["● Notification<br>nothing to do"])
    PRS["HTTP POST /run-ralph<br>{ tool · maxIterations · callbackUrl }"]
    RJS{"status == 'started'?"}
    DS(["● Notification<br>Ralph started · N remaining"])

    %% ── done path ───────────────────────────────────────────────────────
    SF{"$json.body.success?"}
    DF(["● Notification<br>iteration failed<br>exitCode · timedOut"])
    GPD["HTTP GET /prd.json<br>host-gateway:8765"]
    CRD["Code: count remaining"]
    ADD{"remaining == 0?"}
    DC(["● Notification<br>all stories complete!"])
    PRD["HTTP POST /run-ralph<br>{ tool · maxIterations · callbackUrl }"]
    RJD{"status?"}
    DM(["● Notification<br>max iterations reached"])
    DI(["● Notification<br>iteration N done · M remaining"])

    %% ── shared ──────────────────────────────────────────────────────────
    DE(["● Notification<br>error"])
    WAIT(["⏳ n8n execution ends here<br>bridge runs CLI · holds state<br>bridge POSTs callback on exit"])

    %% ── routing ─────────────────────────────────────────────────────────
    WH --> SW
    SW -- "start" --> GS
    SW -- "done"  --> SF
    SW -- "other" --> DE

    GS --> IR
    IR -- "true"  --> NOOP
    IR -- "false" --> GPS
    GPS --> CRS --> ADS
    ADS -- "yes" --> DN
    ADS -- "no"  --> PRS
    PRS --> RJS
    RJS -- "yes" --> DS
    RJS -- "no"  --> DE

    SF -- "false<br>(timedOut or exitCode ≠ 0)" --> DF
    SF -- "true" --> GPD
    GPD --> CRD --> ADD
    ADD -- "yes" --> DC
    ADD -- "no"  --> PRD
    PRD --> RJD
    RJD -- "max_iterations_reached" --> DM
    RJD -- "error / already_running" --> DE
    RJD -- "started" --> DI

    DS --> WAIT
    DI --> WAIT
    WAIT -. "POST {action:'done', success, iteration, tool, exitCode, ...}" .-> WH
```

______________________________________________________________________

## 8. Notification Workflow

The `● Notification` terminal nodes in the ralph-loop workflow each trigger a
separate n8n **notification workflow** via an internal Execute Workflow call.
That workflow owns the notification pipeline and can fan out to multiple
channels independently of the ralph-loop logic.

```mermaid
flowchart TD
    %% ── callers (ralph-loop terminal nodes) ─────────────────────────────
    subgraph RalphLoop["ralph-loop workflow (callers)"]
        DN(["● nothing to do"])
        DS(["● Ralph started"])
        DI(["● iteration N done"])
        DF(["● iteration failed"])
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
    DN & DS & DI & DF & DC & DM & DE -->|"Execute Workflow<br>{event, payload}"| NW
    NW --> FMT
    FMT --> SW2
    SW2 -- "discord" --> DC_NODE
    SW2 -. "whatsapp (future)" .-> WA
    SW2 -. "telegram (future)" .-> TG
    SW2 -. "other (future)" .-> ETC
    DC_NODE --> OK
```
