# Decisions — Issue #75: Tamboui REPL

## D1: REPL Architecture

**Choice:** Standalone Quarkus CLI + Tamboui app in `repl/` module at trellis root
**Alternatives:**
- Sidecar-embedded, remotely rendered — full access to sidecar services but fights Tamboui's direct terminal rendering model
- Separate repo — fully independent but fragments the project unnecessarily
**Rationale:** Clean process boundary, testable with Tamboui Pilot harness, matches incus-spawn prior art. Can be run standalone outside Trellis for debugging.
**Trade-offs:** No direct access to sidecar state — must use pages transport (REST/SSE/WebSocket) when sidecar data is needed.
**Sources:** incus-spawn project structure, Tamboui framework docs
**Exploration:** quick
**Status:** captured

## D2: REPL Hosting

**Choice:** Runs inside tmux terminal managed by Trellis
**Alternatives:**
- Standalone sidecar process — separate process with own communication channel, more complex
**Rationale:** Reuses existing terminal infrastructure (TerminalRegistry, WebSocket relay, xterm rendering). Zero new transport needed for display.
**Trade-offs:** Tied to tmux — can't run without it.
**Sources:** TerminalRegistry.java, TerminalWebSocket.java, terminal-tab-group.ts
**Exploration:** quick
**Status:** captured

## D3: Command Layer

**Choice:** Shell out to soredium Python commands
**Alternatives:**
- Java from scratch — clean but duplicates proven logic
- Sidecar REST API — adds indirection layer
**Rationale:** Reuses proven command implementations. Matches "Python underneath for now" intent. Java equivalents can replace individual commands incrementally later.
**Trade-offs:** Python runtime dependency. Subprocess overhead per command.
**Sources:** soredium commands/ module (start.py, end.py, pause.py, resume.py, etc.)
**Exploration:** quick
**Status:** captured

## D4: LLM Interaction Model

**Choice:** REPL dispatches to LLM by typing into paired terminal via tmux sendKeys. LLM spawned on demand, not persistent.
**Alternatives:**
- Persistent wait agent — constant memory cost (~200-400MB) per idle agent
- API-based — REPL calls LLM API directly, no terminal
- Hybrid — simple calls via API, complex via terminal
**Rationale:** Mechanical-first principle — LLM augments but isn't critical. Spawn-on-demand avoids memory cost of idle agents. sendKeys uses existing infrastructure and preserves LLM session chat history naturally.
**Trade-offs:** No persistent LLM context across commands. Each spawn starts fresh (but can reconstruct context from artifacts).
**Sources:** AgentProcessManager.java, TerminalRegistry.sendKeys()
**Exploration:** deep-analysis
**Status:** captured

## D5: Modal View Integration

**Choice:** Split view default (REPL + LLM terminal both visible), tab mode available. Integrates into existing repo/slot modal views.
**Alternatives:**
- Standalone modal — own modal view, but adds UI complexity and doesn't leverage existing infrastructure
- Tab-only — saves space but user can't see both simultaneously
**Rationale:** Split view is the natural setup when REPL drives commands to the LLM terminal. User clicks to focus. Tab mode available for full-width single-terminal work.
**Trade-offs:** Uses more screen space than tab mode.
**Sources:** terminal-tab-group.ts, existing repo/slot modal views
**Exploration:** quick
**Status:** captured

## D6: Command Structure

**Choice:** Namespace-based command tree (work, git, project, llm) with tab completion
**Alternatives:**
- Flat with aliases — simpler but namespace collisions (git status vs work status)
- Hybrid — namespaced default with top-level aliases for common commands
**Rationale:** Natural grouping for three command domains plus LLM dispatch. Tab completion walks the tree. Clean, no ambiguity.
**Trade-offs:** More typing for common commands (work start vs just start). Acceptable given autocomplete.
**Sources:** soredium commands/registry.py action derivation table
**Exploration:** quick
**Status:** captured

## D7: Input Model

**Choice:** Unified text field with suggestion dropdown — serves both keyboard and mouse users. Typing filters suggestions, tab/arrow navigates, mouse clicks to select.
**Alternatives:**
- Separate keyboard and mouse modes — more complexity, less cohesive
- Pure text REPL — excludes mouse users
**Rationale:** Single widget satisfies both input modes. Same pattern for commands, branches, issues, file paths — any known option set.
**Trade-offs:** Requires Tamboui widget that supports both text entry and clickable suggestions simultaneously.
**Sources:** User requirement for "keyboardless" operation
**Exploration:** quick
**Status:** captured

## D8: Command Configuration

**Choice:** YAML-defined command tree and autocomplete sources, loaded at runtime. No recompile to add/change commands.
**Alternatives:**
- Groovy DSL — more powerful for conditional logic but heavier (requires script engine)
- JSON — structured but verbose for defining flows with branching logic
- Hard-coded Java — requires recompile for every change
**Rationale:** Human-readable, easy to edit. Separates command definitions (data) from execution engine (code). Allows augmenting commands without touching Java.
**Trade-offs:** Less expressive than a full DSL for complex conditional flows.
**Sources:** User requirement for runtime configuration
**Exploration:** quick
**Status:** captured

## D9: Interaction Modes

**Choice:** Two modes — commands (atomic, immediate) and goals (Q&A guided, multi-step, summary before execute). Both YAML-configured.
**Alternatives:**
- Commands only — simpler but forces experienced-user UX on everyone
- Goals only — wizard-style everything, tedious for power users
**Rationale:** Commands for experienced users who know what they want. Goals for guided workflows that compose multiple commands with user confirmation. Goals show summary before executing.
**Trade-offs:** Two YAML schemas to maintain (commands + goals sections).
**Sources:** User requirement for Q&A driven selection with summary
**Exploration:** quick
**Status:** captured

## D10: Sidecar Communication

**Choice:** Pages transport — use REST, SSE, or WebSocket per capability depending on interaction requirements
**Alternatives:**
- Pure REST — simple but no push, requires polling
- Pure WebSocket — single channel but overkill for simple queries
**Rationale:** Pages already provides a unified transport layer. Match transport to capability: push events via SSE, on-demand queries via REST, bidirectional via WebSocket.
**Trade-offs:** Slightly more complex client — must handle multiple transport types.
**Sources:** Pages transport layer, existing sidecar SSE infrastructure (agent:state, workspace:* topics)
**Exploration:** quick
**Status:** captured

## D11: Mechanical vs LLM Work-End

**Choice:** Two paths through work-end — mechanical (close/merge/push, no sweep/squash) and LLM-assisted (full work-end with sweeps). Git squash optionally available mechanically.
**Alternatives:**
- LLM-only work-end — requires agent for every close
- Mechanical-only — loses sweep/squash/review value
**Rationale:** Mechanical-first principle. The work lifecycle must function without an LLM. LLM adds value but isn't required. Users choose their path.
**Trade-offs:** Mechanical path skips sweeps — knowledge capture opportunities may be missed.
**Sources:** soredium work-end/work_end_context.py, forage SWEEP, protocol SWEEP
**Exploration:** deep-analysis
**Status:** captured
