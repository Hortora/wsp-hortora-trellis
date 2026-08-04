# Handoff — trellis

## Last Session

Built repo terminal integration (#24) — repos are now first-class work targets with Claude agent terminals. Replaced custom terminal-panel.ts with shared PagesTerminal from casehub-packages. Fixed scanner to use `slots/` instead of `worktrees/`. Fixed terminal rendering (xterm CSS in shadow DOM, cursor repositioning, focus through nested shadow roots, tmux working directory).

## Immediate Next Step

Branch `issue-24-repo-terminal-integration` is still open. Terminal rendering has a residual garbling issue on reconnect when browser width differs from tmux pane width. Need Playwright visual tests (#25) before further fixes. Run `/work` to continue.

## What's Left

- Terminal garbling on reconnect at different widths · S · Med
- Playwright visual test suite (#25) · M · Med
- Lifecycle buttons (end/next) on repo detail toolbar · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #25 | Playwright visual tests for terminal rendering | M | Med | Created this session |
| #19 | Velocity Tracking + Projections | M | Med | Ready |
| #16 | Drafthouse Integration (B6c) | M | Med | Ready |

## References

- Spec: `specs/repo-terminal-integration/`
- Issue: #24 (open), #25 (open)
- Commits: 009c568..83b7973 (9 commits)
