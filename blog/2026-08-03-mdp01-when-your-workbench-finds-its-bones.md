---
title: "When Your Workbench Finds Its Bones"
author: mdp
date: 2026-08-03
tags: [trellis, architecture, workbench, platform]
---

Trellis had five views, a hash router, and zero awareness that its own platform
provides split panes, dock bars, sidebar navigation, and markdown rendering out
of the box.

The artifact browser was supposed to be the sixth hand-coded view. Instead it
became the forcing function for a question we'd been avoiding: why aren't we
using the platform we're building on?

The answer is now a dock bar on the left edge. Six icons, one active panel. The
existing views slot in as panels with zero rewrite — the workbench creates them
as custom elements and sets properties, exactly as the old router did. The
artifact browser is the first panel built natively with platform primitives:
`marked` for markdown rendering, a grouped sidebar for type-based navigation,
lazy content loading with a client-side cache.

The interesting design decision was what NOT to use. The platform has
`blocks-split-workbench`, `blocks-list-pane`, and `blocks-detail-pane` — full
workbench components with DataSourceMixin and table-based lists. The artifact
browser didn't need any of that. A thin Lit wrapper with a fetch cycle and
`marked.parse()` was the right weight. The platform primitives are there for
when you need them; using them because they exist is the wrong reason.

What the workbench shell actually changed: the hash router's 60-line `if/else`
chain became a data-driven panel registry. Adding a new panel is now one line in
a `PANELS` record, not another branch in a growing conditional. The dock bar
syncs with the hash for deep linking. Panels are lazily created and cached.
When the workspace root changes, all panels are destroyed and recreated —
preventing stale state from a previous workspace leaking into the new one.

The backend is unremarkable by design. `ArtifactScanner` walks known directories
(specs, ADRs, plans, blog, handovers, design docs, journals) in both the
workspace and project repos. `ArtifactResource` serves the list and content with
path validation and a 1 MB size limit. Nine unit tests, seven REST tests. The
scanner handles a broken `proj/` symlink gracefully — returns workspace artifacts
only, logs a warning, doesn't throw.

Next: each existing panel can be incrementally migrated to use platform primitives.
The artifact panel is the reference implementation. One issue per panel.
