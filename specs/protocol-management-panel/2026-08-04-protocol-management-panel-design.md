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

Three layers:

### 1. Curated Lists Registry

A file in hortora's garden (`PROTOCOL-LISTS.md`) that catalogs notable
reusable protocol lists with metadata:

```markdown
| Label | Description | Path |
|-------|-------------|------|
| Universal Java/Quarkus | Reusable conventions for any Java/Quarkus project | casehub/garden/docs/protocols/universal/INDEX.md |
| CaseHub Foundation | Platform building protocols | casehub/garden/docs/protocols/casehub/FOUNDATION-INDEX.md |
| CaseHub Harness | App building protocols | casehub/garden/docs/protocols/casehub/HARNESS-INDEX.md |
```

Each entry has:
- **Label** — display name
- **Description** — what the list covers and who it's for
- **Path** — INDEX.md location (relative to org root or absolute)

### 2. Index Files (INDEX.md)

Curated lists of protocol entries. Markdown tables with columns:

```markdown
| File | Summary | Applies To |
|------|---------|------------|
| [slug.md](slug.md) | One-line directive | Which modules / when |
```

Two shapes:
- **Direct listing** — INDEX.md contains section-grouped tables of protocol entries (soredium pattern)
- **Router** — INDEX.md points to sub-tier INDEX.md files which contain the actual entry tables (garden pattern)

Sub-indexes are detected by rows where the linked file is itself an INDEX.md.

### 3. Protocol Entries (.md files)

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

### Two discovery paths

1. **Registry path** — read `PROTOCOL-LISTS.md` from the garden to get the
   catalog of notable curated lists with labels and descriptions. This is the
   primary browse experience.

2. **Local path** — scan repos on disk for `docs/protocols/INDEX.md`. From
   the project root's parent (the org directory), find all child directories
   with `.git/` and check each for the convention path. Supplements the
   registry with locally-discovered lists not in the catalog.

### INDEX.md chain walking

When a curated list is selected:
1. Read the INDEX.md file
2. Parse markdown table rows (pipe-delimited, regex extraction)
3. For each row: if the linked file is another INDEX.md, follow it recursively
4. Collect all terminal protocol entries with: file path, summary, applies-to, resolved absolute path
5. Section headers (`## Heading`) above tables are preserved as grouping metadata

### File watching

Existing `DirectoryWatcher` pattern triggers rescan when any file under
`docs/protocols/` changes in a watched repo.

## REST API

New `ProtocolResource` at `/api/protocols`:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/protocols/registry` | GET | Curated lists catalog from garden's `PROTOCOL-LISTS.md` |
| `/api/protocols/indexes?repo=<path>` | GET | Discover all INDEX.md files under a repo's `docs/protocols/` |
| `/api/protocols/entries?index=<path>` | GET | Parse an INDEX.md (chain-walk), return all protocol entries |
| `/api/protocols/content?path=<path>` | GET | Serve a protocol `.md` file's raw content |
| `/api/protocols/entries` | POST | Add entry: row to INDEX.md + optional .md creation + git commit |
| `/api/protocols/entries` | DELETE | Remove entry: row from INDEX.md + git commit (file stays on disk) |

### Backend classes

- **`ProtocolScanner`** — discovers repos with protocols, parses INDEX.md tables,
  follows sub-index links. Returns structured data.
- **`ProtocolResource`** — JAX-RS resource exposing the REST API.
- **`ProtocolService`** — orchestrates write operations: INDEX.md editing,
  file creation, git commit.
- **`ProtocolRegistry`** — reads and caches the garden's `PROTOCOL-LISTS.md`.

## Frontend Panel

### Registration

New dock-bar panel `trellis-protocol-view` added to `PANELS` and `DOCK_PANELS`
in `workbench.ts`. Hash route: `#protocols`.

### Layout

Two-column layout matching the garden view pattern (CSS grid, `1fr 1fr`):

**Left column:**
- **Curated lists browser** — cards/rows from the registry, each showing
  label and description. Click to select a list.
- **Protocol entries table** — for the selected list, shows entries parsed
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
1. Panel loads → fetches registry → shows curated lists
2. Click a list → fetches entries → shows protocol table
3. Click an entry → loads `.md` content → shows detail

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

DELETE body:
```json
{
  "indexPath": "/abs/path/to/INDEX.md",
  "file": "protocol-slug.md"
}
```

Steps:
1. Parse INDEX.md, find and remove the row matching the file
2. `git add` INDEX.md, `git commit` with message
   `"protocol: remove <slug> from <index-name>"`
3. The `.md` file is NOT deleted

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

### Registry management

The `PROTOCOL-LISTS.md` file in the garden is itself editable through the
panel — add/remove curated list entries. Same auto-commit pattern, committed
to the garden repo.

## Testing Strategy

- **ProtocolScanner** unit tests: parse known INDEX.md formats (direct listing,
  router, nested sub-indexes), handle malformed tables gracefully
- **ProtocolResource** integration tests: CRUD operations, path traversal
  protection
- **INDEX.md editing** unit tests: add row to correct section, remove row,
  handle edge cases (empty section, last row in section)
- **Frontend** manual testing: browse, select, detail view, add/remove flows
