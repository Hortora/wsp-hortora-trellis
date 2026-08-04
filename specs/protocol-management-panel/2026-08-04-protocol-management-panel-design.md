# Protocol Management Panel — Design Spec

**Date:** 2026-08-04
**Issue:** TBD
**Status:** Draft

## Overview

A read/write dock-bar panel in trellis for browsing, curating, and managing
protocol indexes across repos. Protocols are convention rules stored as
markdown files in `docs/protocols/` directories. INDEX.md files are curated
lists that link to protocol entries. The panel makes these discoverable and
editable.

## Data Model

Two layers:

### 1. Index Files (INDEX.md)

Curated lists of protocol entries. Markdown tables with columns:

```markdown
| File | Rule | Applies To |
|------|------|------------|
| [slug.md](slug.md) | One-line directive | Which modules / when |
```

Column headers vary across repos (`Rule` vs `Summary` vs `Rule Summary`).
The parser matches on pipe-delimited structure, not header names.

Two shapes:
- **Direct listing** — INDEX.md contains section-grouped tables of protocol entries (soredium pattern)
- **Router** — INDEX.md points to sub-tier INDEX.md files which contain the actual entry tables (garden pattern)

Sub-indexes are detected by rows where the linked file contains `INDEX` in
its name (e.g., `INDEX.md`, `FOUNDATION-INDEX.md`, `HARNESS-INDEX.md`).

### 2. Protocol Entries (.md files)

Individual protocol rules with YAML frontmatter:

```yaml
---
id: PP-YYYYMMDD-xxxxxx
title: "Short directive title"
type: principle | rule
scope: platform | repo | universal | application
severity: critical | important | guidance
applies_to: "which modules / contexts"
garden_ref: "GE-YYYYMMDD-xxxxxx"  # optional, tracks garden origin
---

One paragraph. The directive.
```

## Scanning & Discovery

### Convention-based sniffing

A repo has protocols if and only if `docs/protocols/INDEX.md` exists.
No configuration needed — the path is the convention.

### Discovery

Two complementary paths:

1. **Local scanning** — reuse `WorkspaceScanner`'s existing repo discovery
   (finds child directories with `.git/`). For each discovered repo, check
   if `docs/protocols/INDEX.md` exists. No duplicate repo-sniffing logic.

2. **Curated registry** (future) — a `PROTOCOL-LISTS.md` file in hortora's
   garden cataloging notable reusable protocol lists with label, description,
   and path metadata. Deferred to a follow-up — local scanning covers the
   primary use case. When added, registry entries supplement locally-discovered
   lists with curated metadata.

### INDEX.md chain walking

When a curated list is selected:
1. Read the INDEX.md file
2. Parse markdown table rows (pipe-delimited, regex extraction)
3. For each row: if the linked file contains `INDEX` in its name, follow it
   recursively. Track visited paths in a `Set<Path>` to detect cycles.
4. Collect all terminal protocol entries with: file path, summary, applies-to,
   resolved absolute path
5. Section headers (`## Heading`) above tables are preserved as grouping metadata

### File watching

Integrate with existing `FileWatcherService`. Add `docs/protocols/` to the
watched paths. Emit a new SSE topic (`workspace:protocols`) on changes.
Debounce write-triggered events — when `ProtocolService` writes a file, set
a flag to suppress the immediate watcher callback and avoid feedback loops.

## REST API

New `ProtocolResource` at `/api/protocols`:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/protocols/repos?root=<path>` | GET | List repos with `docs/protocols/INDEX.md` under the given root. Reuses `WorkspaceScanner` repo discovery. |
| `/api/protocols/indexes?repo=<path>` | GET | Discover all INDEX.md files under a repo's `docs/protocols/` |
| `/api/protocols/entries?index=<path>` | GET | Parse an INDEX.md (chain-walk), return all protocol entries |
| `/api/protocols/entries` | POST | Add entry: row to INDEX.md + optional .md creation + git commit |
| `/api/protocols/entries?index=<path>&file=<slug>` | DELETE | Remove entry: row from INDEX.md + git commit (file stays on disk) |

**Content serving:** reuse the existing `ArtifactResource` endpoint
(`GET /api/artifacts/content?path=...&root=...`) for reading protocol `.md`
files. No duplicate content endpoint.

**Path security:** all path parameters are validated against a whitelist of
known repo roots (from `WorkspaceScanner`). Paths must resolve within a known
repo's `docs/protocols/` directory. Reuse `ArtifactResource`'s existing path
traversal guards (canonical path comparison).

**Write serialization:** `ProtocolService` synchronizes write operations on
a per-INDEX.md lock (`ConcurrentHashMap<Path, ReentrantLock>`) to prevent
concurrent modifications to the same file.

### Backend classes

- **`ProtocolScanner`** — parses INDEX.md tables, follows sub-index links
  with cycle detection. Returns structured data. Does NOT duplicate repo
  discovery — receives repo paths from `WorkspaceScanner`.
- **`ProtocolResource`** — JAX-RS resource exposing the REST API.
- **`ProtocolService`** — orchestrates write operations: INDEX.md editing,
  file creation, git commit. Shells out to `git` for commits (new capability
  for trellis sidecar — first write operation). Serializes writes per-file.

## Frontend Panel

### Registration

New dock-bar panel `trellis-protocol-view` added to `PANELS` and `DOCK_PANELS`
in `workbench.ts`. Hash route: `#protocols`.

### Layout

Two-column layout matching the garden view pattern (CSS grid, `1fr 1fr`):

**Left column:**
- **Repo selector** — dropdown or list of repos that have protocols (from
  `/api/protocols/repos`). Click to select.
- **Index list** — for the selected repo, shows all INDEX.md files found
  under `docs/protocols/`. Click to select an index.
- **Protocol entries table** — for the selected index, shows entries parsed
  from the INDEX.md chain. Each row: file name, summary, applies-to.
  Remove button per row.
- **Add action** — button that opens garden search for finding entries
  to promote to protocols.

**Right column:**
- **Protocol detail** — selected entry's `.md` content rendered via `marked`.
  YAML frontmatter displayed as metadata badges (scope, severity, type) above
  the body text. Read-only.

### Component structure

- `trellis-protocol-view` — top-level view component (dock-bar panel)
- `trellis-protocol-list` — curated lists browser + entries table
- Reuses `garden-entry-detail` rendering pattern for protocol detail

### Interaction flows

**Browse:**
1. Panel loads → fetches repos with protocols → shows repo list
2. Click a repo → fetches its indexes → shows index list
3. Click an index → fetches entries → shows protocol table
4. Click an entry → loads `.md` content via `/api/artifacts/content` → shows detail

**Add from garden:**
1. Click "Add" → garden search bar appears (inline or overlay)
2. Search returns garden entries via existing `/api/garden/search`
3. Select an entry → prompted for target INDEX.md
4. Backend creates protocol `.md` from garden entry, adds row to INDEX.md, git commits

**Remove:**
1. Click remove on an entry row
2. Backend removes row from INDEX.md, git commits
3. Protocol `.md` file stays on disk

## Write Operations

### Add entry to INDEX.md

POST body:
```json
{
  "indexPath": "/abs/path/to/INDEX.md",
  "section": "## Section Header",
  "file": "new-protocol-slug.md",
  "summary": "One-line directive",
  "appliesTo": "Which modules / when",
  "gardenEntryId": "GE-XXXXXXXX-XXXXXX",
  "content": "... protocol body if creating new file ..."
}
```

Steps:
1. If `gardenEntryId` is set: fetch garden entry, transform to protocol format
   (map frontmatter, add `garden_ref`, carry body text), write `.md` file
2. Parse INDEX.md, find the target section, append a new table row
3. `git add` both files, `git commit` with message
   `"protocol: add <slug> to <index-name>"`

### Remove entry from INDEX.md

DELETE `/api/protocols/entries?index=<path>&file=<slug>` (query params, no body).

Steps:
1. Parse INDEX.md, find and remove the row matching the file
2. `git add` INDEX.md, `git commit` with message
   `"protocol: remove <slug> from <index-name>"`
3. The `.md` file is NOT deleted

### Error handling

Write operations (add/remove) are atomic at the INDEX.md level:
- Read the file, apply the edit in memory, write back
- If the git commit fails, restore the original file content from the
  in-memory copy (no partial state on disk)
- Return 500 with error detail on failure

### Garden entry → protocol transformation

When promoting a garden entry:
- `id` → generated as `PP-YYYYMMDD-xxxxxx`
- `title` → from garden entry title
- `scope` → user selects (universal / platform / repo / application)
- `severity` → defaults to `guidance`, user can adjust
- `type` → defaults to `rule`
- `garden_ref` → set to the GE-ID for traceability
- `applies_to` → from garden entry domain or user-provided
- Body → garden entry body carried across

## Testing Strategy

- **ProtocolScanner** unit tests: parse known INDEX.md formats (direct listing,
  router, nested sub-indexes), handle malformed tables gracefully, cycle
  detection, varying column headers (`Rule` vs `Summary`)
- **ProtocolService** unit tests: add row to correct section, remove row,
  handle edge cases (empty section, last row in section), rollback on
  git commit failure
- **ProtocolResource** integration tests: CRUD operations, path traversal
  protection, write serialization
- **Frontend** manual testing: browse, select, detail view, add/remove flows

## Review Findings Incorporated

From light post-spec review (2026-08-04):

- Dropped `PROTOCOL-LISTS.md` registry as premature — local scanning covers
  the primary use case (STR R1-05, XC R1-04). Registry deferred to follow-up.
- Reuse `WorkspaceScanner` for repo discovery instead of duplicating (STR R1-02)
- Reuse `ArtifactResource` for content serving instead of new endpoint (STR R1-03)
- Added cycle detection to INDEX.md chain walking (ROB R1-04)
- Added path traversal security — whitelist of known repo roots (ROB R1-05)
- Added write serialization via per-file locks (ROB R1-01)
- Added rollback on partial write failure (ROB R1-03)
- Added file watcher debouncing to prevent feedback loops (ROB R1-02)
- Changed DELETE to use query params instead of request body (ROB R1-07)
- Fixed table column header to match real INDEX.md files (XC R1-12)
- Git writes via shelling out to `git` — first write operation in trellis
  sidecar, documented as architectural precedent (STR R1-04, ROB R1-15)
