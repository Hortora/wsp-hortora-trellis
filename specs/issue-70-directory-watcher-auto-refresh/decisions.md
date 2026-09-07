## D1: Notification architecture

**Choice:** Domain-routed SSE topics with partial rescan (Approach A)
**Alternatives:**
- Single `workspace:changed` event with domain tags — simpler backend but every panel receives every event, defeating the efficiency goal
- Per-domain generation counters with client polling — adds latency, generates constant traffic, steps backwards from existing push infrastructure
**Rationale:** Leverages the existing `EventBroadcaster` topic system exactly as designed. Each panel subscribes only to the topics whose data it consumes, so a `.git/HEAD` change doesn't wake the protocol panel. Partial rescan means the backend does less work too — `WorkspaceScanner` already has separate methods per domain. Most testable (each domain independently).
**Trade-offs:** Requires maintaining a path→domain classification map on the backend. More wiring than a single broadcast, but the classification is just a handful of path prefix checks.
**Sources:** `FileWatcherService.java:79-98` (existing rescan + broadcast), `PushResource.java` (SSE topic delivery), `WorkspaceScanner.java:42-65` (separate scanRepos/scanSlots)
**Exploration:** quick
**Status:** captured

## D2: Debounce strategy

**Choice:** Idle-wait with max cap — trailing-edge debounce with a hard ceiling
**Alternatives:**
- Fixed-interval polling (e.g., drain changeset every N seconds) — simpler but adds up to N seconds latency and burns cycles when nothing changed
- No debounce, process every event — overwhelms the system during git operations or batch writes
**Rationale:** Balances low latency for isolated file changes (3s default idle) against burst protection during sustained writes (20s max cap). First change is reflected within 3s of quiet; during a `git rebase` touching hundreds of files, the system waits for the storm to pass (up to 20s max). `DirectoryChangeEvent.path()` provides the changed path — accumulated into a `Set<Path>` during the window, classified in bulk at drain time.
**Trade-offs:** Giving up sub-second reactivity for stability. 3s idle window means the UI is never perfectly real-time, but for workspace state (branch changes, slot transitions) this is imperceptible.
**Sources:** `io.methvin.watcher.DirectoryChangeEvent` (provides `path()` and `rootPath()`), `io.methvin.watcher.changeset.ChangeSetListener` (accumulation model, though we build our own debounce timer)
**Exploration:** quick
**Status:** captured

## D3: Configuration via PreferencesService

**Choice:** Debounce timing and fallback rescan interval read from `PreferencesService` with sensible defaults
**Alternatives:**
- Hardcoded constants — simpler but inflexible; the current `RESCAN_INTERVAL_SECONDS = 60` is already hardcoded
- Quarkus `@ConfigMapping` — appropriate for deployment-time config but these values may need tuning per-user at runtime
**Rationale:** `PreferencesService` already exists, reads `~/.trellis/preferences.json`, supports typed accessors with defaults. Values live under a `watcher` section: `debounceIdleSeconds` (default 3), `debounceMaxWaitSeconds` (default 20), `fallbackRescanSeconds` (default 60). Works out of the box without any config file. The existing hardcoded `RESCAN_INTERVAL_SECONDS` also moves to preferences.
**Trade-offs:** Runtime JSON reads on each preference access (mitigated by the volatile snapshot pattern already in `PreferencesService`). If the user edits preferences.json while trellis is running, they'd need to trigger a reload — but that's the existing pattern.
**Sources:** `PreferencesService.java:14-89` (existing pattern with `getInt()` + defaults), `FileWatcherService.java:22` (hardcoded constant to migrate)
**Exploration:** quick
**Status:** captured
