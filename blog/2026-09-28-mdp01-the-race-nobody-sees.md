---
title: "The Race Nobody Sees"
date: 2026-09-28
author: mdp
entry_type: note
subtype: diary
series: issue-89-terminal-navigation
projects: [Hortora/trellis]
tags: [lit, web-components, async, race-condition, terminal, repo-detail]
---

The bug was intermittent — terminal content vanishing after arrow navigation in the repo detail modal. Sometimes it rendered, sometimes blank. The kind of thing that makes you question whether you actually saw it.

We traced the data flow through `repo-detail.ts` and found two async fetches racing each other. When `repoName` changes, both `_loadRepo()` and `_loadTerminal()` fire in parallel. The first one sets `_loading = true`, which tears the terminal element out of the DOM (replaced by a loading spinner). If the second fetch resolves while the spinner is still showing, the configure callback runs `querySelector('#repo-terminal')` and gets null — the element doesn't exist yet.

The configure is skipped. But the guard variable (`_lastTerminalName`) doesn't update either, because it's inside the `if (el)` block. When loading finally completes and the terminal element appears in the DOM, nothing re-triggers configure. The `updated()` lifecycle only fires on property changes, and `_terminalName` didn't change that render cycle.

A second variant: navigating from a repo with a terminal to one without, then back. The guard retains the terminal name from the first visit and blocks configure on return — same name means "already configured," even though the DOM element is brand new.

The fix is two lines of logic: reset the guard when the repo changes, and check for pending configuration when `_loading` transitions to false. The extracted `_needsTerminalConfigure()` method captures both triggers cleanly.

While fixing the modal, I noticed the repo detail sidebar was sparse compared to slot detail — just path and remote link, while slots showed status, issues, plan progress, and covers badges. Extended `RepoInfo` on the backend to parse `.plan` files during workspace scanning, so standalone repos now carry the same plan/issue metadata that slots do. The sidebar now shows work state, clickable issue links, and plan progress with issue refs that open GitHub in a new tab.

The Lit async race is interesting beyond this specific fix. Any component with two parallel async fetches where one controls conditional rendering creates the same window: the configuring fetch can resolve while the rendering fetch still has the element absent. `updateComplete.then()` guarantees the current render is complete, but it can't guarantee the element exists if a sibling state removed it. The pattern — watch for loading-complete as a secondary configure trigger — is general.
