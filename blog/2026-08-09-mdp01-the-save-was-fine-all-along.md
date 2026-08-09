---
layout: post
title: "The Save Was Fine All Along"
date: 2026-08-09
entry_type: note
subtype: diary
projects: [trellis]
tags: [dockview, persistence, debugging, testing]
series: issue-43-frame-tab-management
---

*Continues from [When Your Agent Gets Eyes and Hands](2026-08-06-mdp01-when-your-agent-gets-eyes-and-hands.md).*

I spent two days convinced the position persistence bug was in the save path. Eight different fixes — event-driven tracking, beforeunload guards, panel-content matching, race condition suppression. Each one passed the unit tests. Each one failed in the browser.

The unit tests were the problem. The mock `DockviewComponent` was 80 lines of array operations that didn't have a layout engine, didn't clamp positions, didn't switch CSS anchoring modes. Every test that said "position persisted correctly" was testing against a fantasy. The mock agreed with every fix I wrote because it had no opinions about how overlays actually work.

The real bug was in the restore path. Dockview's `addPanel({floating: {x, y}})` clamps the position to the container's known dimensions. The container's known dimensions are zero when `DockviewComponent` is initialized in Lit's `firstUpdated` — the browser hasn't done a layout pass yet, and Dockview's `ResizeObserver` hasn't fired. Every frame restored at (0, 0). Every save afterward faithfully recorded (0, 0).

The fix was one line: `requestAnimationFrame(() => requestAnimationFrame(() => this._restoreLayout()))`. Two animation frames to let the browser lay out the container and let the ResizeObserver update Dockview's internal state. That's it. Two days for one line, because I was looking at the wrong end of the pipeline.

What finally broke the loop was a `console.log` in the save path. The log showed `pos=(201,198)` — the save was writing correct coordinates. The frame appeared at (0, 0). Save correct, display wrong. That pointed straight at the restore path. Thirty seconds of observation beat two days of source analysis.

The other outcome from this session: drop zone indicators are gone. Dockview has a smooth tab reorder mode (`tabAnimation: 'smooth'`) that works like iTerm2 — drag a tab and the others shift in real time to make room. No blue lines, no position markers, no MutationObserver hackery. The old drop zone code was 80 lines of fragile DOM observation that never quite worked for 3+ tab frames. Replaced by a single theme property.

There's also a second Dockview gotcha worth knowing: after a user drags a floating group, the overlay's CSS switches from top/left anchoring to bottom/left anchoring. `overlay.toJSON()` returns `{bottom, left}` with `top` literally absent from the object. Code that checks `bounds.top != null` passes fine at creation (top is 0, which is not null) but fails after drag (top is undefined, which is also not null — wait, no. `undefined != null` is `false`). The fix is to read `getBoundingClientRect()` from the overlay's DOM element instead of trusting `toJSON()`.

The Dockview workspace code is moving to pages next. Everything we built — floating groups, position persistence, smooth reorder, overlay event subscriptions — is generic. Nothing about it requires trellis. A `pages-floating-workspace` component would give any casehub app the same capability, tested properly from the start with Playwright against real Dockview instead of happy-dom mocks.

The mock tests are gone now. 22 fake DnD/position tests removed, replaced by 4 Playwright e2e tests that run against the real running app with real Dockview. The remaining 193 unit tests cover pure logic — math, serialization, command dispatch. Things that don't need a browser to verify.
