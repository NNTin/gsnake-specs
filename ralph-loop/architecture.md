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
        Discord["Discord
        notifications"]
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
    N8N -->|"Discord webhook"| Discord
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
    end

    subgraph Host["Host Process"]
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

    Claude -->|"on exit
    POST callback"| N8N
    Codex -->|"on exit
    POST callback"| N8N

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

    Done_NothingToDo --> [*] : Discord "nothing to do"

    StartIteration --> InFlight : POST /run-ralph → status "started"
    StartIteration --> Done_MaxHit : POST /run-ralph → status "max_iterations_reached"
    StartIteration --> Done_Error : POST /run-ralph → network/bridge error

    Done_MaxHit --> [*] : Discord "max iterations reached"
    Done_Error --> [*] : Discord "error"

    InFlight --> CheckStatus : POST /webhook/ralph-loop<br>action "done", success true
    InFlight --> Done_IterationFailed : POST /webhook/ralph-loop<br>action "done", success false

    Done_IterationFailed --> [*] : Discord "iteration failed"

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
    [*] --> Idle : startup<br>(load state.json)

    Idle --> Running : POST /run-ralph<br>(not at maxIterations)
    Idle --> Idle : POST /run-ralph<br>{status "already_running"} — impossible from Idle
    Idle --> Idle : POST /run-ralph<br>{status "max_iterations_reached"}

    Running --> Running : POST /run-ralph → {status "already_running"}
    Running --> Idle : CLI exits (success)<br>→ POST callbackUrl {success true}
    Running --> Idle : CLI exits (error)<br>→ POST callbackUrl {success false}
    Running --> Idle : RALPH_ITERATION_TIMEOUT exceeded<br>→ POST callbackUrl {timedOut true}

    note right of Running
        state persisted to state.json
        on every transition
    end note

    Idle --> Idle : GET /status (read-only)
    Running --> Running : GET /status (read-only)
    Idle --> Idle : GET /prd.json (read-only)
    Running --> Running : GET /prd.json (read-only)
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
        DC["Discord webhook"]
    end

    BridgeSvc -->|"reads"| CLAUDE_MD
    BridgeSvc -->|"reads/writes"| PRD
    BridgeSvc -->|"writes"| Archive
    BridgeSvc -->|"spawns"| Claude_Bin
    BridgeSvc -->|"spawns"| Codex_Bin

    N8N_Container -->|"HTTP :8765<br>via host-gateway"| BridgeSvc
    BridgeSvc -->|"POST callback<br>via public URL"| N8N_URL
    N8N_Container -->|"Discord"| DC
    Claude_Bin -->|"git push"| GH
```
