---
layout: post
title: "Dependency Graphs and the Blackboard Coordinator"
date: 2026-08-01
type: phase-update
entry_type: note
subtype: diary
projects: [trellis]
tags: [algorithms, critical-path, llm, dependency-graph, java, quarkus]
series: trellis-build
---

The core question Trellis answers is: "what should I work on next?" Getting from GitHub issues to a useful recommendation turns out to require three distinct layers, and the design choice that matters most is keeping them separate.

## Parsing dependencies from the wild

GitHub issues don't have a dependency field. People express blocked-by relationships in at least three ways: labels (`blocked-by:#42`), body checklists (`- [ ] #42 — implement the parser`), and inline text (`**Blocked by:** #42, owner/repo#15`). DependencyParser handles all three, including cross-repo references in `owner/repo#N` format. Checked checklist items (`- [x]`) are marked as resolved dependencies — they stay in the graph but don't block.

The parser feeds DependencyGraph, which builds a forward and reverse adjacency map. From there, three algorithms do the real work.

## Critical path and bottleneck detection

Critical path is a longest-path computation on the DAG. Topological sort first (Kahn's algorithm — if the sort doesn't consume all nodes, the remainder form cycles, which get excluded from path computation but shown in the UI as warnings). Then dynamic programming: each open node has weight 1, each closed node has weight 0. Walk the sorted order, propagate distances, track predecessors. The node with maximum distance is the path endpoint; backtrack through predecessors for the full chain.

The bottleneck detector is the piece I find most interesting. For each open issue, it simulates completing that issue and then counts how many downstream issues become unblocked as a result — recursively, through the full transitive chain. This "cascade unlock count" is the real measure of an issue's impact. An issue that sits on the critical path and unblocks three others transitively is more valuable to complete than one that unblocks nothing.

```java
// RecommendationEngine scoring — simple and principled
if (onCritPath) {
    score = 100 + cascade;    // critical path issues dominate
    reason = "On critical path, unblocks " + cascade + " issues transitively";
} else if (cascade > 1) {
    score = 50 + cascade;     // bottlenecks that aren't on critical path
    reason = "Bottleneck — completing this unblocks " + cascade + " issues";
}
```

Critical path issues get base score 100 plus cascade count. Bottlenecks (cascade > 1 but not on critical path) get 50 plus cascade. Everything else is filtered out — only actionable, high-value recommendations survive. The scoring is intentionally transparent: a developer looking at "score 103, critical path, unblocks 3" can verify the reasoning immediately.

## The LLM layer: a blackboard, not a brain

The key design choice: algorithms produce rankings, the LLM adds reasoning. The two layers are intentionally separate. The algorithmic layer works and is testable without an LLM. The LLM adds value on top — it doesn't replace the algorithm.

The coordinator uses a blackboard architecture. Events flow through the system as a sealed interface with three record types — workspace changes, analysis recomputed, issue state changes. They accumulate in a ring buffer (EventRing, capacity 256). A SignificanceFilter examines each batch: the current implementation triggers on analysis events with newly unblocked issues. Only significant batches invoke the LLM.

CoordinatorContextAssembler structures the prompt from live system state. It appends epic KPIs (total, open, closed, critical path length, bottleneck count), algorithmic recommendations with scores and reasoning, recent events from the ring buffer, and conversation history for interactive mode. Three prompt modes serve different triggers: proactive (significant events happened, LLM offers advice), interactive (user asks a question), and enhancement (algorithmic recommendations get natural language reasoning layered on top).

The separation matters for a practical reason. LLM calls are expensive — in latency, tokens, and money. The algorithmic layer handles the common case (ranked recommendations with transparent scoring) at zero marginal cost. The LLM is reserved for the cases where natural language reasoning adds genuine value: explaining *why* a recommendation matters in the context of the current epic's progress, or answering freeform questions about delivery state that don't reduce to a graph query.

The spec survived contact with implementation largely intact. That's what ten rounds of adversarial review buys you — fewer surprises when the code materialises. The failure modes documented during review (WatchService event loss, process timeouts, concurrent slot operations) mapped directly to the concurrency guards and fallback paths in the production code.

Next: point it at a real multi-repo epic and see if the recommendations match what a developer would choose by intuition. That's the real test — not whether the algorithm is correct, but whether it's useful.
