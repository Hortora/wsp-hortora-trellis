# trellis Workspace

**Name:** trellis
**Project repo:** proj/
**Workspace type:** public

## Session Start

Run `# add-dir removed — use proj/ symlink instead and `# add-dir removed — use proj/ symlink instead before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| update-design | `design/JOURNAL.md` (created by `work-start`) |
| adr | `adr/` |
| write-content | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md
- `design/` — work-start creates JOURNAL.md and .meta here per branch

## Git Discipline

Two git repositories are active in every session:
- **Workspace** (wksp/) — plans, blog, snapshots, handover
- **Project repo** (proj/) — source code, ADRs, specs

Never rely on CWD for git operations. Always use explicit paths:
```bash
git -C proj/ add <file>   # workspace
git -C proj/ add <file>          # project
```

## Rules

- All methodology artifacts go here, not in the project repo
- Workspace branches mirror project branches — switch both together

## Routing

| Artifact   | Destination |
|------------|-------------|
| adr        | project     |
| specs      | project     |
| blog       | workspace   |
| plans      | workspace   |
| design     | workspace   |
| snapshots  | workspace   |
| handover   | workspace   |

**Blog directory:** `wksp/`

@/Users/mdproctor/claude/hortora/trellis/CLAUDE.md
