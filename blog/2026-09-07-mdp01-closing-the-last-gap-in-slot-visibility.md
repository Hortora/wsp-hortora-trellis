---
layout: post
title: "Closing the Last Gap in Slot Visibility"
date: 2026-09-07
entry_type: note
subtype: diary
projects: [Hortora/trellis]
tags: [trellis, slots, plan-parsing, scanner, frontend]
---

Trellis already knows what a slot is doing — it tracks status, repos, terminals, agent state. But until now, it couldn't tell you where a slot was in its work. If a slot had a `.plan` with batches and a queue of issues, the only way to see progress was to open the file.

That's the kind of gap that's invisible until someone asks. The slot detail modal shows everything about a slot except the thing you most want to know: which issue is active, how many are done, what's next.

The fix is a straight extension of the scanner pattern that already reads `.slot` files. `WorkspaceScanner` now reads `.plan` files alongside them, parsing the queue into three records — `PlanItem`, `PlanBatch`, `PlanProgress` — and attaching the result to `SlotInfo`. The parser handles the plan_manager format: batch headers, checked/unchecked items, the `← active` marker, and epic parent lines (which get skipped since they're containers, not work items).

On the frontend, the slot detail sidebar grows a Plan section between Issues and Repos. Batches render as uppercase headers, items show done/active/pending states with check marks and dots, and a progress summary anchors the bottom: "2/4 done · Batch 2 of 3". The data arrives through the existing `/api/workspace` endpoint — no new fetch, no new SSE topic. The file watcher already classifies `slots/` changes into the `SLOTS` domain, so `.plan` edits trigger re-scans automatically.

What makes this satisfying is that nothing new had to be invented. The scanner already knew how to read per-slot files. The sidebar already had the visual language. The file watcher already covered the directory. The whole thing slots in — three records, one parser method, one render method, four tests.
