# wayfinder-sync

One-way sync that copies a project's GitHub wayfinder maps into `.plan/maps/` so chartr's Wayfinder panel can display them.

Wayfinder maps are charted and worked on GitHub Issues by Claude Code (using the `wayfinder` skill). chartr can only draw maps from local Markdown. wayfinder-sync connects the two and keeps the local copy fresh:

- A change Claude makes appears in chartr within about 5 seconds (through a Claude Code `PostToolUse` hook).
- A change made anywhere else appears within about 3 minutes (through a LaunchAgent timer).
- A failed sync never damages the existing copy.

It never writes to GitHub.

## Status

In design. The full design, including commands, footprint and the GitHub-to-Markdown translation, is in [docs/specs/2026-10-02-wayfinder-sync-design.md](docs/specs/2026-10-02-wayfinder-sync-design.md).

## Planned usage

```sh
wayfinder-sync add      # register the current project and run a first sync
wayfinder-sync          # sync the current project
wayfinder-sync doctor   # check the install, gh auth and last sync times
```

Built as a single standard-library Python 3 script that talks to GitHub only through `gh`.
