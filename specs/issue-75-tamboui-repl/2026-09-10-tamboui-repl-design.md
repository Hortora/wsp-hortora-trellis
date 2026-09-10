# Tamboui REPL for Mechanical Work Lifecycle

**Issue:** Hortora/trellis#75
**Date:** 2026-09-10
**Status:** Draft

## Problem

Trellis has terminals and agents but no mechanical command interface. Users must either use the LLM for everything (expensive, slow for routine operations) or leave Trellis entirely to run CLI commands. The soredium TUI (Python/Textual) exists but is on hold and separate from Trellis. The work lifecycle should be drivable mechanically, with LLM as an optional augmentation.

## Solution

A Tamboui-based Java REPL that runs inside a tmux terminal, paired with an LLM terminal, shown in the existing repo/slot modal view. The REPL handles the work lifecycle mechanically and dispatches to the LLM when needed.

## Architecture

### Process Model

```
Trellis Sidecar
  ├── creates tmux session: REPL terminal
  ├── creates tmux session: LLM terminal (existing agent terminal)
  └── serves pages transport (REST/SSE/WS)

REPL Process (lightweight Java CLI + Tamboui)
  ├── spawned into REPL tmux session via sidecar sendKeys
  ├── receives: --repo, --slot, --issue, --sidecar-port, --paired-terminal
  ├── connects to sidecar via pages transport (java.net.http + SSE client)
  ├── lifecycle commands via soredium CLI wrapper (JSON Lines protocol)
  ├── LLM dispatch via sidecar REST: POST /api/terminals/{name}/input
  └── agent lifecycle via sidecar REST: POST /api/terminals/{name}/agent/*

Crash recovery: sidecar's TerminalRegistry.bootstrap() recovers the tmux
session on restart. The REPL process is re-spawned by sending the launch
command to the recovered session.
```

**Not Quarkus.** The REPL is a lightweight Java CLI — Tamboui + java.net.http. No CDI container, no Quarkus bootstrap. The REPL is a presentation layer (TUI, autocomplete, goals) and a dispatch layer (calling sidecar REST API). All application logic stays in the sidecar.

### Module Structure

New Maven module at `repl/` in trellis root, sibling to `sidecar/` and `shell/`.

```
repl/
  pom.xml                          # Tamboui + java.net.http (no Quarkus)
  src/main/java/io/hortora/trellis/repl/
    ReplMain.java                  # main() entry point
    ReplApp.java                   # Tamboui app, status bar, input widget
    command/
      CommandRegistry.java         # Namespace tree, tab completion, dispatch
      CommandDefinition.java       # YAML-loaded command model
      GoalDefinition.java          # YAML-loaded goal model (Q&A flows)
      GoalRunner.java              # Executes goal steps, collects answers, shows summary
    input/
      SuggestionInput.java         # Unified text+click input widget
      SuggestionSource.java        # Interface for autocomplete data providers
    sidecar/
      SidecarClient.java           # HTTP client + SSE subscription to sidecar
    soredium/
      SorediumBridge.java          # Invokes soredium CLI wrapper, parses JSON Lines events
  src/main/resources/
    commands.yaml                  # Command tree + goal definitions
```

**Command dispatch is via sidecar REST API, not direct method calls.** The REPL never touches `TerminalRegistry`, `AgentProcessManager`, or `TmuxManager` directly. It calls:
- `POST /api/terminals/{name}/input` — send text to paired LLM terminal
- `POST /api/terminals/{name}/agent/start` — spawn agent on demand
- `GET /api/terminals` — check agent state
- Plus existing worklog, dependency, intelligence endpoints

### Soredium Bridge

Soredium commands (`commands/start.py`, `commands/end.py`, etc.) are Python modules with `execute()` functions, not CLI entry points. The bridge requires a CLI wrapper:

```
soredium/cli/__main__.py          # JSON Lines CLI wrapper
  ├── accepts: command name + kwargs as JSON
  ├── calls: commands/<name>.execute(**kwargs)
  ├── emits: JSON Lines events (StepProgress, BranchCreated, WorkEnded, CommandFailed)
  └── exit code: 0 success, 1 failure

REPL invokes: python3 -m soredium.cli start '{"issue": 42, "repo": "..."}'
REPL reads: JSON Lines from stdout, updates TUI accordingly
```

This CLI wrapper is a new addition to soredium — a thin layer that serialises the existing event stream.

### Mechanical-First Principle

The work lifecycle runs without an LLM. The LLM augments with higher-value operations.

| Operation | Mechanical (REPL) | LLM-assisted |
|-----------|-------------------|--------------|
| work start | Yes | — |
| work pause/resume | Yes | — |
| work continue | Yes | — |
| work next | Yes | — |
| work end (close/merge/push) | Yes | — |
| work status | Yes | — |
| git operations | Yes | — |
| build/test | Yes | — |
| sweep (forage, protocol, doc-sync, blog) | — | Yes |
| squash | Optional (mechanical rebase) | Yes (semantic squash) |
| code review | — | Yes |
| blog writing | — | Yes |
| full work-end (sweep + squash + review + close) | — | Yes |

### LLM Dispatch

The REPL dispatches to the LLM by sending text to the paired terminal via the sidecar REST API. From the LLM's perspective, a human typed the command. The session chat history accumulates naturally.

The LLM is spawned on demand — when a user invokes an `llm` command, the REPL:
1. Checks if an agent is running in the paired terminal: `GET /api/terminals/{paired-terminal}`
2. If not, starts one: `POST /api/terminals/{paired-terminal}/agent/start`
3. Sends the command text: `POST /api/terminals/{paired-terminal}/input`
4. The user sees the LLM working in the split view

The LLM is abstracted — not Claude-specific. The agent start mechanism and command format are configurable.

### Command Structure

Namespace-based command tree with tab completion:

```
work
  ├── start [issue]        # create branch, scaffold .plan
  ├── continue             # resume work on current branch
  ├── pause                # save state, switch away
  ├── resume               # restore from pause stack
  ├── next                 # advance to next issue in .plan queue
  ├── end                  # mechanical close (merge, push)
  ├── status               # current branch, state, plan position
  └── slot
      └── create [--isx]   # create slot, optionally with ISX isolation

git
  ├── status
  ├── log
  ├── diff
  ├── commit [message]     # commit with issue linkage
  ├── branch               # branch listing/switching
  └── stash                # stash management

project
  ├── build                # compile (e.g. mvn compile)
  ├── test [pattern]       # run tests
  └── run                  # start dev mode

llm
  ├── start                # spawn agent in paired terminal
  ├── stop                 # stop agent
  ├── sweep                # full sweep (forage, protocol, doc-sync, blog)
  ├── squash               # dispatch git squash
  ├── review               # dispatch code review
  ├── blog                 # dispatch blog entry writing
  ├── end                  # dispatch full LLM-assisted work-end
  └── <freetext>           # send arbitrary prompt to the agent
```

### Two Interaction Modes

**Commands** — atomic operations. Type the command with autocomplete, it executes immediately.

**Goals** — multi-step guided flows. The REPL walks through Q&A to gather all inputs, shows a summary, then executes. Goals compose one or more commands. Both modes fully keyboard or mouse driven via the unified input widget.

### YAML Command Configuration

Commands, autocomplete flows, and goals are defined in YAML, loaded at startup. No recompile to add or change commands.

```yaml
commands:
  work:
    description: "Work lifecycle"
    children:
      pause:
        description: "Save state and switch away"
        handler: soredium:pause
      status:
        description: "Current branch, state, plan position"
        handler: soredium:status

goals:
  start-work:
    description: "Start working on an issue"
    steps:
      - prompt: "Select repo"
        source: sidecar:repos
        type: select
      - prompt: "Which issue?"
        source: github:open-issues
        type: select
      - prompt: "ISX isolation?"
        type: confirm
        default: false
      - type: summary
      - type: confirm
        prompt: "Proceed?"
    execute:
      - soredium:start --repo {repo} --issue {issue}
```

**Autocomplete sources** are pluggable: `sidecar:repos` queries the sidecar, `github:open-issues` calls `gh`, `shell:git-branches` runs git. New sources registered in YAML or discovered via SPI.

### Unified Input Widget

A single Tamboui input widget that serves both keyboard and mouse users:

- Text field with suggestion dropdown
- Typing filters suggestions in real-time (fuzzy match)
- Tab/arrow keys navigate suggestions, Enter selects
- Mouse click on any suggestion selects directly
- Context-aware completions: commands, branches, issues, test patterns, file paths

### Sidecar Integration

The REPL connects to the sidecar at startup using the port from spawn args. Uses pages transport (REST/SSE/WebSocket) per capability.

**Real-time state (SSE push):** Agent state changes, file watcher events, terminal lifecycle events.

**On-demand queries (REST pull):** Terminal list, worklog work-items, dependency graph, intelligence findings, backlog.

**Status bar:** Persistent Tamboui status line showing current branch, work state, agent status, repo/slot context, last command result. Updates reactively from SSE events.

### Modal View Integration

The REPL integrates into existing repo/slot modal views in Trellis. Two display modes:

- **Split view (default):** REPL and LLM terminal side by side. User clicks to focus.
- **Tab view:** Switch between REPL and LLM tabs for full-width single terminal.

This is more constrained than the workspace view — fixed to two terminals, no drag-and-drop.

## Testing

- Tamboui Pilot harness for headless REPL widget testing
- Command registry unit tests (namespace resolution, tab completion)
- YAML parsing tests (command definitions, goal flows)
- SorediumBridge integration tests (subprocess invocation, output parsing)
- Sidecar client tests (REST/SSE connection, event handling)

## Future Work

- Standalone `isx` command namespace (isx list, isx ssh, etc.)
- Java replacements for soredium Python commands (incremental migration)
- Additional goal definitions as usage patterns emerge
- REPL history and command recall (check if Tamboui provides this natively)
- Coordination with claudony extraction (#48) — terminal/agent code may move to shared platform layer

## References

- TerminalRegistry.java — tmux session management
- AgentProcessManager.java — agent lifecycle (start/stop/pause/resume)
- terminal-tab-group.ts — simpler tab-based terminal component
- soredium commands/ module — Python lifecycle command implementations
- soredium work-slot/slot_isx.py — ISX isolation for slots
- incus-spawn project — prior art for Quarkus + Tamboui CLI
- Tamboui framework (dev.tamboui) — Java TUI widgets, CSS, toolkit, Pilot test harness
