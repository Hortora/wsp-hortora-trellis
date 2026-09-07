# Directory Watcher Auto-Refresh — Design Spec

## Problem

Panels that display workspace-derived state (repos, slots, epics, protocols,
backlog, intelligence, blockers) only fetch data on mount or manual refresh.
File changes in the workspace — new commits, plan updates, slot state
transitions, worklog writes — are not reflected until the user navigates away
and back.

The backend (`FileWatcherService`) already detects file changes and broadcasts
to SSE topics (`workspace:repos`, `workspace:slots`, `workspace:protocols`),
but no frontend panel subscribes to them. Additionally, every file change
triggers a full `WorkspaceScanner.scan()` — a complete filesystem walk —
regardless of which domain was affected.

## Design

### Data flow

```
File change (DirectoryWatcher / worklog.db mtime)
  → Debouncer (idle-wait with max cap)
  → PathDomainClassifier (path → Set<Domain>)
  → Partial rescan (only affected domains)
  → EventBroadcaster (domain-specific SSE topics)
  → SSE push to frontend
  → Panel re-fetches from existing REST endpoint
```

### Domain classification

`PathDomainClassifier` — a pure function mapping changed paths to affected
domains. Extracted as a standalone class for testability.

| Path pattern | Domain | SSE topic |
|---|---|---|
| `<repo>/.git/HEAD`, `<repo>/.git/refs/**` | `REPOS` | `workspace:repos` |
| `slots/**/.slot`, `slots/**/.phase-a-complete` | `SLOTS` | `workspace:slots` |
| `**/docs/protocols/INDEX.md` | `PROTOCOLS` | `workspace:protocols` |
| `**/.plan`, `**/.pause-stack`, `**/.epic` | `LIFECYCLE` | `workspace:lifecycle` |
| `worklog.db` (external path) | `WORKLOG` | `workspace:worklog` |

A single file change may match multiple domains (e.g., a `.plan` inside a
slot directory matches both `SLOTS` and `LIFECYCLE`). The classifier returns
a `Set<Domain>`. Unclassified paths are ignored — not every file change is
relevant (e.g., source code edits, build artifacts).

The classifier operates on paths relative to the watched root. For worklog.db,
the root is the `~/.hortora/` directory.

### Panel subscription table

Each panel subscribes to the SSE topics whose data it consumes and re-fetches
from its existing REST endpoint on notification.

| Panel | Subscribes to | Re-fetches |
|---|---|---|
| `trellis-org-dashboard` | `workspace:repos`, `workspace:slots`, `workspace:lifecycle` | `GET /api/workspace?root=...` |
| `trellis-repo-detail` | `workspace:repos` | `GET /api/workspace?root=...` (repo subset) |
| `trellis-slot-detail` | `workspace:slots` | slot load logic |
| `trellis-protocol-view` | `workspace:protocols` | `GET /api/protocols/repos?root=...` |
| `trellis-backlog-panel` | `workspace:lifecycle`, `workspace:worklog` | `GET /api/backlog?root=...` |
| `trellis-intelligence-panel` | `workspace:lifecycle`, `workspace:worklog` | `GET /api/intelligence?root=...` |
| `trellis-blockers-panel` | `workspace:worklog` | `GET /api/dependencies?root=...` |

### Debounce strategy

Idle-wait with max cap — balances low latency for isolated changes against
burst protection during sustained writes (git operations, batch updates).

```
State: IDLE
  first event → record firstEventTime, schedule drain in idleWait → ACCUMULATING

State: ACCUMULATING
  more events → reset drain timer to idleWait from now
  maxWait elapsed since firstEventTime → drain immediately
  idleWait timer fires → drain → IDLE
```

Events accumulate into a `Set<Path>` during the debounce window. At drain time,
all paths are classified in bulk, then partial rescans run for each affected
domain.

The debouncer is per-watched-root. Workspace roots and the worklog directory
each have their own debounce state — a burst of workspace file changes does
not delay worklog notifications and vice versa.

### Configuration

All timing values read from `PreferencesService` (`~/.trellis/preferences.json`)
with sensible defaults. No config file needed for out-of-the-box operation.

```json
{
  "watcher": {
    "debounceIdleSeconds": 3,
    "debounceMaxWaitSeconds": 20,
    "fallbackRescanSeconds": 60
  }
}
```

The existing hardcoded `RESCAN_INTERVAL_SECONDS = 60` in `FileWatcherService`
moves to `fallbackRescanSeconds` in preferences.

### Backend changes

#### `PathDomainClassifier` (new)

Package: `io.hortora.trellis.scanner`

```java
public class PathDomainClassifier {
    public enum Domain { REPOS, SLOTS, PROTOCOLS, LIFECYCLE, WORKLOG }

    public Set<Domain> classify(Path changedPath, Path watchRoot) { ... }
}
```

Pure function. Given a changed path and the watch root, returns the set of
affected domains. Stateless, no dependencies. Unit tested with path fixtures.

#### `FileWatcherDebouncer` (new)

Package: `io.hortora.trellis.scanner`

Manages the debounce state machine per watched root. Uses
`ScheduledExecutorService` for timer management.

```java
public class FileWatcherDebouncer {
    public FileWatcherDebouncer(ScheduledExecutorService scheduler,
                                int idleWaitSeconds,
                                int maxWaitSeconds,
                                Consumer<Set<Path>> drainCallback) { ... }

    public void onFileChanged(Path path) { ... }
}
```

On drain: passes accumulated `Set<Path>` to the callback. The callback
(in `FileWatcherService`) classifies and triggers partial rescans.

#### `WorkspaceScanner` changes

Expose existing private methods as package-private for partial rescan:

- `scanRepos(Path root)` → `List<RepoInfo>` (already exists as private)
- `scanSlots(Path root)` → `List<SlotInfo>` (already exists as private)
- `scanWorkspaces(Path root, ...)` → epics + pauses (already exists as private)

No protocol-specific scan method needed. The `PROTOCOLS` domain is
signal-only — when `PathDomainClassifier` detects an `INDEX.md` change,
`FileWatcherService` broadcasts `workspace:protocols` without rescanning.
The protocol panel re-fetches via `GET /api/protocols/repos?root=...`
which reads INDEX.md directly from disk.

No new scanning logic — just visibility changes to enable domain-scoped calls.

#### `FileWatcherService` changes

- Replace `rescan(Path root)` full-scan with domain-scoped partial rescans
- Create `FileWatcherDebouncer` per watched root instead of direct
  `event -> rescan(root)` listener
- Add `watchWorklog()` to watch `~/.hortora/` for worklog.db changes
- Read timing config from `PreferencesService` instead of hardcoded constants
- Add `workspace:lifecycle` and `workspace:worklog` topic constants
- Broadcast only for domains where the partial rescan produced a change
  (compare old vs new model segment, same as current full-model compare)

#### `PreferencesService` changes

Add watcher config accessors:

```java
public int watcherDebounceIdleSeconds() { ... }    // default 3
public int watcherDebounceMaxWaitSeconds() { ... }  // default 20
public int watcherFallbackRescanSeconds() { ... }   // default 60
```

These read from `root.getJsonObject("watcher")` using the existing `getInt()`
pattern.

### Frontend changes

#### Shared SSE utility

Create a workspace-scoped subscription helper that panels import.
Uses the sse-manager pattern (shared EventSource per URL) rather than
per-panel raw EventSource connections.

SSE events arrive as unnamed `message` events with JSON payload:
`{ topic: "workspace:repos", payload: ... }`. The helper filters by topic
and dispatches to the panel's callback.

```typescript
// workspace-sse.ts
export function subscribeWorkspace(
  topics: string[],
  onTopicMessage: (topic: string) => void,
): () => void { ... }  // returns unsubscribe function
```

**Connection strategy:** One shared EventSource subscribes to ALL
`workspace:*` topics up front (`/api/push?topics=workspace:repos&topics=
workspace:slots&topics=workspace:protocols&topics=workspace:lifecycle&
topics=workspace:worklog`). The `subscribeWorkspace()` helper filters
incoming messages client-side by the `topics` parameter, invoking the
callback only for matching topics. This avoids reconnecting the
EventSource when new panels mount — all topics are already subscribed.
The cost of receiving unused topics is negligible (workspace events are
infrequent).

The shared connection is created lazily on first `subscribeWorkspace()`
call and closed when the last subscriber unsubscribes.

Panels call `subscribeWorkspace()` in `connectedCallback()` and the
returned unsubscribe function in `disconnectedCallback()`.

#### Panel changes

Each panel adds:

1. `subscribeWorkspace([...topics], () => this._reload())` in `connectedCallback()`
2. Unsubscribe in `disconnectedCallback()`
3. The `_reload()` / `_scan()` / `_fetch()` method already exists — it's
   what runs on mount. No new data fetching logic needed.

This is mechanical — each panel gets ~5 lines of SSE wiring that calls
its existing fetch method.

### worklog.db watching

`FileWatcherService.watchWorklog()` watches `~/.hortora/` for changes to
`worklog.db`. Uses the same `DirectoryWatcher` + debouncer pattern as
workspace roots. The `PathDomainClassifier` maps `worklog.db` to `WORKLOG`
domain.

This runs once at startup (the path is well-known) alongside whatever
workspace roots are registered. The worklog watcher is independent of
workspace root lifecycle — it starts when the sidecar starts and stops
at shutdown.

### Testing

- **`PathDomainClassifierTest`** — unit tests for each path pattern: repo
  git changes, slot files, protocol INDEX.md, lifecycle files, worklog.db,
  unclassified paths, multi-domain matches
- **`FileWatcherDebouncerTest`** — unit tests for debounce state machine:
  single event drain, burst coalescing, max cap enforcement, idle reset,
  concurrent roots
- **`FileWatcherServiceTest`** — integration test verifying that a file
  write in a watched directory results in the correct SSE topic broadcast
  after debounce
- **Frontend** — verify each panel re-fetches on SSE message (unit test
  with mock EventSource)

## References

- `FileWatcherService.java` — existing watcher infrastructure
- `WorkspaceScanner.java` — existing scanner with per-domain private methods
- `PushResource.java` — SSE endpoint
- `EventBroadcaster` (pages-push) — topic-based SSE broadcasting
- `PreferencesService.java` — runtime preferences pattern
- `sse-manager.ts` (pages-data) — shared EventSource management
- `workbench.ts:360-374` — existing SSE consumption pattern (topic routing via msg.topic)
- `DirectoryChangeEvent` (io.methvin) — provides `path()` for changed file
- Issue #70 — problem statement and incremental refresh requirement
