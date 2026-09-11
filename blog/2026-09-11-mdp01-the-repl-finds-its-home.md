---
layout: post
title: "The REPL Finds Its Home"
date: 2026-09-11
entry_type: note
subtype: diary
projects: [Hortora/trellis]
tags: [repl, tamboui, terminal-pairing, split-view, sidecar]
series: issue-75-tamboui-repl
---

# The REPL Finds Its Home

Yesterday I designed the REPL and built the widget. Today's question was simpler but more satisfying: where does it live, and how does it get there?

## Two Terminals, One View

The REPL and LLM terminal are a pair. They share a repo context, they share a purpose, and they should share a screen. The existing modal views in Trellis — repo-detail and slot-detail — each show a single terminal or a tab group. Neither supports showing two terminals simultaneously.

I built `trellis-terminal-pair-view`, a Lit component with two modes. Split mode renders both terminals side by side in a flex row. Tab mode shows one at a time with a tab bar. A toggle button switches between them. The split is the default — the whole point is seeing both the mechanical REPL and the LLM working in parallel, so hiding one behind a tab defeats the purpose.

The component is deliberately constrained. No drag-and-drop, no resize handles, no floating frames. Two terminals, fixed layout, mode toggle. The workspace view already handles the complex case with its floating frame engine and Dockview backend. The pair-view is for the modal context where you want to look at one repo's REPL+agent and nothing else.

Integration into the detail views was straightforward. When the sidecar reports two terminals with a pairing relationship, the view renders the pair component. When there's one terminal, it falls back to the existing single-terminal rendering. When there are three or more (a slot with multiple repos), it uses the tab group.

## Pairing as Data, Not Heuristic

The first version detected pairs by counting — two terminals for a repo meant a pair. That's fragile. A repo could have two agent terminals for different branches, or a terminal left over from a crashed session. Count isn't semantics.

I added a `pairedTerminal` field to `TerminalInfo` on the sidecar side. When you create a REPL terminal, you name the terminal it pairs with. The sidecar persists this as a tmux user option (`@trellis_paired`) so the relationship survives sidecar restarts. The frontend reads it from the REST response and uses it for explicit pair detection instead of guessing.

The same change added a `command` field to the terminal creation request. When present, the sidecar sends that command into the tmux session via `sendKeys` instead of starting a Claude agent. This is how the REPL process gets spawned — the sidecar doesn't need to know it's a REPL. It just creates a terminal and types a command into it. The mutual exclusion between `command` and `agent` is enforced at the API level.

## History

Tamboui has no built-in command history. I checked the 0.4.0 source — `TextInputState` does text editing and nothing else. So I built `CommandHistory`: a list of commands with a cursor. `add()` appends and resets position to the end. `previous()` walks back. `next()` walks forward, returning empty when past the end (which clears the input — the same behavior as bash). Consecutive duplicates are suppressed.

Wiring it into `ReplApp` was three changes: a field, a `history.add(input)` call in `processCommand`, and UP/DOWN key handling when the suggestion dropdown isn't visible. When the dropdown IS visible, UP/DOWN still navigate suggestions — the context determines the behavior.

## What Opens Up

The REPL has a home now. It can be spawned into a tmux session, paired with an LLM terminal, and rendered side by side in the Trellis UI. The mechanical-first principle from yesterday's design is no longer theoretical — the infrastructure exists to create a REPL terminal and type commands into it.

What's missing is the actual deployment path. The `repl/` module builds a JAR but nobody calls `java -jar` yet. That's the next connection: a button in the UI or a lifecycle hook that creates both terminals as a pair and launches the REPL process with the right args.
