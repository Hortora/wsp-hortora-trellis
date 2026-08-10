# Decisions — workspace-view engine refactor

## D1: Replace inline Dockview with FloatingFrameEngine + DockviewBackend

**Choice:** Import `createFloatingFrameEngine` and `createDockviewBackend` from `@casehubio/pages-runtime` via portal resolution. Delegate all frame/tab state management to the engine.
**Alternatives:**
- Keep inline Dockview code — duplicates 1,800+ lines already extracted to pages-runtime
- Import only the backend, manage state inline — partial extraction, two sources of truth
**Rationale:** The engine and backend were extracted specifically from this file in slot 5. Using them eliminates duplication and makes trellis a consumer of the tested, general-purpose component.
**Trade-offs:** Trellis loses direct Dockview access for edge cases. Mitigated by `backend.unwrap()`.
**Exploration:** quick
**Status:** captured

## D2: Type bridging — TabRef ↔ FrameTabConfig

**Choice:** Keep trellis's `TabRef { terminalName, type }` as the domain type. Convert to/from pages `FrameTabConfig { key, label, content }` at the boundary: `key = terminalName`, `label = display name`, `content` unused (ContentFactory provides elements).
**Alternatives:**
- Adopt FrameTabConfig directly throughout trellis — invasive change to MCP command handler, getUIState, groups, persistence
- Create an adapter type — unnecessary indirection
**Rationale:** TabRef is used by MCP commands, persistence, groups, and getUIState. Changing it ripples through the backend Java code. Boundary conversion is cheap and isolated.
**Trade-offs:** Two tab representations exist. Conversion is mechanical and localised to 2-3 methods.
**Exploration:** quick
**Status:** captured

## D3: ContentFactory provides terminal elements

**Choice:** Implement `ContentFactory` in workspace-view that creates `pages-component-terminal` elements, connects WebSocket, and returns `{ element, dispose }`. Terminal lifecycle (connect, retry, renderer tiers) stays in workspace-view.
**Alternatives:**
- Push terminal lifecycle into a separate component — over-abstraction for a single consumer
**Rationale:** The factory pattern matches exactly how workspace-view already creates terminal panels. The engine calls `factory(tab)` and workspace-view handles everything terminal-specific.
**Trade-offs:** None significant.
**Exploration:** quick
**Status:** captured

## D4: Delete trellis utility modules replaced by pages-runtime

**Choice:** Delete `workspace-zorder.ts`, `workspace-spatial-nav.ts`, `workspace-organisers.ts` and their tests. These are now in pages-runtime as `frame-zorder.ts`, `frame-spatial-nav.ts`, `frame-organisers.ts`.
**Alternatives:**
- Keep both — redundant code, drift risk
**Rationale:** The pages modules were extracted from these exact files. The engine uses them internally. No reason to keep the duplicates.
**Trade-offs:** Trellis can no longer customise these algorithms independently. Acceptable — they should evolve in pages where they're tested.
**Exploration:** quick
**Status:** captured

## D5: Keep workspace-renderer-tiers.ts

**Choice:** Retain `workspace-renderer-tiers.ts` — it's trellis-specific (WebGL/Canvas/None tier logic tied to Electron IPC). No equivalent in pages-runtime.
**Alternatives:** None — this is trellis-specific by nature.
**Rationale:** Renderer tier management depends on Electron's WebGL context budget IPC (`trellis.webglAcquire/Release`). Pages has no concept of this.
**Trade-offs:** None.
**Exploration:** quick
**Status:** captured

## D6: Test strategy — mock FloatingFrameBackend instead of DockviewComponent

**Choice:** Replace the 100-line `DockviewComponent` mock in workspace-view.test.ts with a mock `FloatingFrameBackend`. Tests interact with the engine's public API instead of Dockview internals.
**Alternatives:**
- Keep the Dockview mock — defeats the purpose of the extraction
- Test through Playwright only — too slow for unit-level assertions
**Rationale:** The engine is tested in pages-runtime. Trellis tests should verify trellis-specific behavior (terminal lifecycle, MCP commands, persistence format, groups) through the engine's public interface.
**Trade-offs:** Tests that verified Dockview-specific behavior (overlay.setBounds, panel order) become integration tests. The engine's own tests cover that layer.
**Exploration:** quick
**Depends on:** D1
**Status:** captured
