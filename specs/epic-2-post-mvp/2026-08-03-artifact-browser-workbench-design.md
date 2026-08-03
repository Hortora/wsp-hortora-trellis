# Artifact Browser and Workbench Shell

**Issue:** Hortora/trellis#15
**Date:** 2026-08-03
**Status:** Approved

## Problem

Trellis generates methodology artifacts (specs, ADRs, plans, blogs, handovers,
journals) across workspace and project directories. There is no way to browse
or read them within trellis — users must open files in an external editor.

The frontend is built as five hand-coded Lit views with a hash router, bypassing
the platform's workbench primitives (split panes, dock bars, sidebar navigation,
markdown rendering, data binding). Adding the artifact browser as a sixth
hand-coded view deepens this debt.

## Solution

Two changes in one:

1. **Workbench shell** — replace the hash router with a dock-bar workbench.
   A Lit component (`trellis-workbench`) renders a vertical dock bar and
   manages panel activation. Existing views become panels — created as
   custom elements with properties set by the shell, exactly as the current
   router does. No `hostPanel` DSL call — the shell is the host, not a
   pages-rendered page.

2. **Artifact panel** — the first panel built using platform rendering
   primitives. A thin Lit wrapper (`trellis-artifact-panel`) that uses the
   platform's `renderMarkdown()` from `channel-activity` for content display
   and a grouped sidebar for type-based navigation. The wrapper handles the
   fetch-on-select cycle; the rendering uses platform primitives throughout.

## §1 Workbench Shell

### Layout

```
┌──┬────────────────────────────────────────┐
│📁│                                        │
│📋│          Active Panel                  │
│📄│     (full height, full width)          │
│🌿│                                        │
│🤖│                                        │
│⚡│                                        │
└──┴────────────────────────────────────────┘
 ↑ dock bar (vertical, left edge)
```

### Dock Items

| Icon | Panel ID | Label | Component |
|------|----------|-------|-----------|
| 📁 | workspace | Workspace | `trellis-org-dashboard` (existing) |
| 📋 | slot | Slot | `trellis-slot-detail` (existing) |
| 📄 | artifacts | Artifacts | `trellis-artifact-panel` (new) |
| 🌿 | garden | Garden | `trellis-garden-view` (existing) |
| 🤖 | coordinator | Coordinator | `trellis-coordinator-panel` (existing) |
| ⚡ | epic | Epic | `trellis-epic-dashboard` (existing) |

### Shell Component — `trellis-workbench`

`trellis-workbench` is a Lit component that owns the dock bar and panel lifecycle.
It replaces the `route()` function in `app.ts`. It is not a pages-rendered page —
it is the application shell that hosts panels, some of which may use pages
primitives internally.

```typescript
const PANELS = {
  workspace:   { icon: '📁', label: 'Workspace',   tag: 'trellis-org-dashboard' },
  slot:        { icon: '📋', label: 'Slot',         tag: 'trellis-slot-detail' },
  artifacts:   { icon: '📄', label: 'Artifacts',    tag: 'trellis-artifact-panel' },
  garden:      { icon: '🌿', label: 'Garden',       tag: 'trellis-garden-view' },
  coordinator: { icon: '🤖', label: 'Coordinator',  tag: 'trellis-coordinator-panel' },
  epic:        { icon: '⚡', label: 'Epic',          tag: 'trellis-epic-dashboard' },
};
```

Responsibilities:
- Render vertical dock bar with icon buttons
- Manage active panel state — one panel visible at a time
- Create panel elements lazily on first activation
- Pass workspace root and panel-specific context (slot number, epic ref) to panels
- Sync active panel with URL hash for deep linking

### Panel Lifecycle

Panels are created lazily on first activation and cached. When the workspace root
changes (user switches to a different project), all cached panels are destroyed
and recreated on next activation — this prevents stale state from a previous
workspace leaking into the new one.

Context properties (`workspaceRoot`, `slotNumber`, `epicRef`, etc.) are set on
panel elements whenever the active panel changes or the hash updates. Panels are
responsible for reacting to property changes (standard Lit reactive properties).

### Hash Routing

The hash remains the deep-link mechanism. The workbench reads it on load and
listens for `hashchange`.

| Hash | Panel | Context |
|------|-------|---------|
| `#?root=...` | workspace | `workspaceRoot` |
| `#slot/N?root=...` | slot | `slotNumber`, `workspaceRoot` |
| `#artifacts?root=...` | artifacts | `workspaceRoot` |
| `#garden` | garden | — |
| `#coordinator?root=...&epic=owner/repo/N` | coordinator | `workspaceRoot`, `epicRef` |
| `#epic/owner/repo/N?root=...` | epic | `owner`, `repo`, `epicNumber`, `workspaceRoot` |

Clicking a dock button updates the hash. Hash changes activate the corresponding
panel. Old hash formats continue to work — the mapping is backward compatible.

Unknown or malformed hashes activate the workspace panel as a safe default (or
the launcher if no workspace root is available).

### Launcher Exception

When no workspace is configured (no `root` in hash, first launch), the workbench
is not rendered. The launcher view (`trellis-launcher`) shows instead. Once a
project is bootstrapped with a workspace root, the workbench activates with the
workspace panel.

### Existing Views — Zero Rewrite

Existing Lit components become panels by being created as custom elements within
the workbench's panel container. The workbench sets properties on them
(`workspaceRoot`, `slotNumber`, etc.) the same way the current router does.
No changes to existing view code.

Future migration: each panel can be incrementally rewritten to use pages
primitives. The artifact panel serves as the reference implementation. One issue
per panel migration.

## §2 Artifact Panel

### Layout

```
┌──────────┬─────────────────────────────┐
│ Specs    │                             │
│  spec-1  │  # Spec Title               │
│  spec-2  │                             │
│ ADRs     │  Content rendered as         │
│  adr-1   │  markdown with full GFM     │
│ Plans    │  support (tables, code       │
│ Blog     │  blocks, lists)             │
│ Handovers│                             │
│ Design   │                             │
│ Journals │                             │
└──────────┴─────────────────────────────┘
  sidebar        markdown viewer
```

### Component — `trellis-artifact-panel`

A thin Lit wrapper that composes platform rendering primitives and handles the
data fetch cycle. Uses `renderMarkdown()` from `@casehubio/channel-activity` for
markdown→HTML conversion with DOMPurify sanitization. The sidebar is a custom
grouped list (artifact types as collapsible headings, files as clickable items).

State:
- `artifacts: ArtifactEntry[]` — populated from list endpoint
- `selectedPath: string | null` — currently selected artifact
- `content: string` — markdown content of selected artifact
- `contentCache: Map<string, string>` — path → content cache

### Artifact Types and Source Paths

| Type | Workspace path | Project path | Notes |
|------|---------------|-------------|-------|
| Specs | `specs/` | `docs/specs/` | Both: workspace has branch-scoped, project has promoted |
| ADRs | `adr/` | `docs/adr/` | Architecture decision records |
| Plans | `plans/` | `docs/plans/` | Implementation plans |
| Blog | `blog/` | — | Workspace only |
| Handovers | `HANDOFF.md` | — | Single file at workspace root |
| Design | — | `docs/ARC42STORIES.MD` | Project only |
| Journals | `design/JOURNAL.md` | — | Current branch journal |

### Sidebar Navigation

The sidebar groups artifacts by type. Each type heading is collapsible. Individual
files are listed under their type with the filename (sans extension) as the label.
Clicking a file selects it and loads its content.

Sorted: types in the fixed order listed above, files alphabetically by name within
each type. (`modifiedAt` is unreliable after git operations like checkout and
rebase — alphabetical ordering is stable and predictable.)

### Markdown Rendering

Uses `renderMarkdown()` from `@casehubio/channel-activity` — the platform's
`marked` (GFM mode) + `DOMPurify` pipeline. This function:
- Parses GFM markdown (tables, strikethrough, task lists, fenced code blocks)
- Sanitizes output via DOMPurify with a strict tag/attribute whitelist
- Returns safe HTML string

Not in scope for #15:
- Syntax highlighting (add via `marked` highlight extension later)
- Mermaid diagram rendering

### Content Loading

1. Panel mounts → `GET /api/artifacts?root=<workspace>` → populate sidebar
2. User clicks artifact → check `contentCache` → if miss,
   `GET /api/artifacts/content?path=<path>` → render
3. Cache is per-session (Map), cleared when `workspaceRoot` property changes

### UI States

| State | Display |
|-------|---------|
| Loading artifact list | Sidebar: "Loading..." spinner |
| Empty artifact list | Sidebar: "No artifacts found" message |
| Loading content | Content pane: "Loading..." spinner (sidebar remains interactive) |
| Content load error | Content pane: error message with retry link |
| No selection | Content pane: "Select an artifact to view" placeholder |

## §3 Backend

### Package: `io.hortora.trellis.artifact`

### ArtifactEntry

```java
record ArtifactEntry(
    String type,        // "spec", "adr", "plan", "blog", "handover", "design", "journal"
    String name,        // filename without extension
    String path,        // absolute path
    Instant modifiedAt  // file last modified (informational, not used for ordering)
)
```

### ArtifactScanner

Stateless utility. Scans workspace and project paths for markdown artifacts.

```java
class ArtifactScanner {
    List<ArtifactEntry> scan(Path workspaceRoot)
}
```

Scan logic:
- Resolve project root from the workspace's `proj/` symlink. If the symlink is
  missing or broken, log a warning and skip project artifacts — do not throw.
  Workspace artifacts are still returned.
- For each artifact type, walk its known directory in workspace and/or project
- Collect `*.md` files, classify by parent directory name
- Single-file artifacts (`HANDOFF.md`, `JOURNAL.md`, `ARC42STORIES.MD`) checked
  directly — if the file doesn't exist, skip silently
- Skip `INDEX.md` files (convention: index files are not browseable artifacts)
- Return sorted: by type order (fixed), then alphabetically by name within type

### ArtifactResource

```
GET /api/artifacts?root=<workspace-root>
    → List<ArtifactEntry>

GET /api/artifacts/content?path=<absolute-path>&root=<workspace-root>
    → raw markdown (text/plain, max 1 MB)
```

### Content Size Limit

The content endpoint enforces a 1 MB file size limit. Files larger than 1 MB
return 413 Payload Too Large. This prevents accidental loading of large generated
files or binary content mistakenly placed in artifact directories.

### Path Validation

The content endpoint validates that `path` resolves to a file within either:
- The workspace root
- The project root (resolved via `proj/` symlink)

Validation uses `Path.toRealPath()` to resolve symlinks before the prefix check,
ensuring symlinks within the workspace/project tree are followed correctly.
Any path outside these boundaries returns 403.

### Error Responses

| Condition | HTTP Status |
|-----------|------------|
| Missing `root` parameter | 400 |
| Workspace root not found | 404 |
| Artifact path outside allowed roots | 403 |
| Artifact file not found | 404 |
| Artifact file exceeds 1 MB | 413 |

## §4 Testing

### Unit Tests

**`ArtifactScannerTest`** — temp filesystem (`java.nio.file.Files.createTempDirectory`):
- Finds specs in both workspace and project directories
- Finds single-file artifacts (HANDOFF.md, ARC42STORIES.MD)
- Handles missing directories gracefully (empty list, no error)
- Handles broken `proj/` symlink gracefully (returns workspace artifacts only)
- Skips INDEX.md files
- Sorts by type order, then alphabetically by name

### REST Tests (`@QuarkusTest`)

**`ArtifactResourceTest`**:
- List endpoint returns entries for a workspace with artifacts
- Content endpoint serves raw markdown as text/plain
- Content endpoint returns 403 for path traversal attempt
- Content endpoint returns 404 for nonexistent file
- Content endpoint returns 413 for file exceeding 1 MB
- List endpoint returns empty array for workspace with no artifacts

### Frontend

Manual Playwright verification:
- Dock bar renders with all panel icons
- Clicking dock icons switches panels
- Artifact sidebar shows entries grouped by type
- Selecting an artifact loads and renders markdown content
- Hash deep linking works (`#artifacts?root=...`)
- Existing views (workspace, slot, garden) still work when accessed via dock
- Workspace change destroys and recreates panels

## §5 Scope Boundary

### In scope
- Workbench shell with dock bar replacing hash router
- Existing views wrapped as panels (zero rewrite)
- Artifact panel with sidebar navigation and markdown rendering
- Backend artifact scanning and content serving
- Path validation and size limit on content endpoint
- Hash-based deep linking
- Panel lifecycle on workspace change

### Out of scope
- Rewriting existing views to use pages primitives (future: one issue per panel)
- Syntax highlighting in code blocks
- Mermaid diagram rendering
- Full-text search across artifacts
- Artifact editing (read-only browser)
- File watching / live reload on artifact changes
