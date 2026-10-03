# wayfinder-sync — design

Date: 2026-10-02
Status: approved in conversation, pending spec review

## Problem

Wayfinder maps are charted and worked by Claude Code using Matt Pocock's `wayfinder`
skill in its GitHub mode, so GitHub Issues are the source of truth. chartr's Wayfinder
panel can only draw maps from local Markdown at `.plan/maps/<slug>/`. Nothing connects
the two.

## Goal

A one-way sync that copies each project's GitHub wayfinder maps into `.plan/maps/` so
chartr can display them, kept fresh automatically.

### Success criteria

- A change Claude makes to a map's issues appears in chartr within about 5 seconds.
- A change made anywhere else (github.com, phone, collaborators) appears within about
  3 minutes.
- A failed sync never damages the existing copy, and the failure is visible in chartr.
- Every file the tool creates outside its repo is documented, checkable with `doctor`,
  and removable with `uninstall.sh`.

### Non-goals

- Writing anything to GitHub. Agents write to GitHub directly through the wayfinder skill.
- Launching tickets from chartr (**Review & launch**) or using its Agent plugin. Tickets
  are worked by prompting Claude Code directly.
- Committing plan files. `.plan/` is gitignored in every project.
- Nested sub-issues deeper than one level.

## Usage model

1. In a project, run `wayfinder-sync add` once. The project is registered, `.plan/` is
   added to its `.gitignore`, and an initial sync runs.
2. Open the project as a chartr folder space and open the Wayfinder panel.
3. Run Claude Code (in chartr or another terminal) with the wayfinder skill to chart and
   work maps on GitHub. The panel follows along.

## Footprint

The tool creates exactly these things outside its own repo:

| What | Path |
|---|---|
| Command symlink | `~/.local/bin/wayfinder-sync` → `<repo>/wayfinder_sync.py` |
| Claude Code hook | One `PostToolUse` entry in `~/.claude/settings.json`, identified by the marker `wayfinder-sync hook` in its command |
| Timer | `~/Library/LaunchAgents/com.amierld.wayfinder-sync.plist` |
| Config | `~/.config/wayfinder-sync/projects` (one absolute path per line) and `~/.config/wayfinder-sync/paused` (present only while paused) |
| State | `~/.config/wayfinder-sync/state/` (locks, dirty flags, last-sync times, rate-limit and auth-warning timestamps) |
| Log | `~/Library/Logs/wayfinder-sync.log`, plus one rotated `wayfinder-sync.log.1` |
| Per project | `.plan/maps/<marked map folders>/` and a `.plan/` line in `.gitignore` |

The README repeats this table under the heading **Footprint**.

## Commands

| Command | Effect |
|---|---|
| `wayfinder-sync` | Sync the project containing the current directory |
| `wayfinder-sync all` | Sync every registered project (used by the timer) |
| `wayfinder-sync add` / `remove` | Register or unregister the current project. `remove` also deletes its marked map folders. It leaves the `.gitignore` line alone. |
| `wayfinder-sync hook` | Hook entry point; reads hook JSON on stdin |
| `wayfinder-sync doctor` | ✓/✗ for each footprint row, `gh` auth, each project's remote, and last sync time |
| `wayfinder-sync log` | Show the tail of the log |
| `wayfinder-sync pause` / `resume` | Create or remove the `paused` flag; while paused, the hook and timer do nothing |
| `--dry-run` | Print the files that would be written, write nothing |
| `--verbose` | Print GitHub responses and each decision |

`install.sh` creates the footprint and can be run repeatedly. `uninstall.sh` lists what
it will remove, asks for confirmation, then removes every footprint row, including
marked map folders in registered projects. It removes only the hook entry carrying the
marker and leaves the rest of `settings.json` byte-for-byte unchanged. It leaves the
`.gitignore` lines in place.

## Architecture

A single Python 3 file using only the standard library (`wayfinder_sync.py`), organised
as independent units:

- **fetch** — the only code that talks to GitHub. It runs `gh api graphql` (gh handles
  auth; no token is stored) and pages through map issues, sub-issues and blockers.
  It returns plain dicts. It contains no logic.
- **translate** — a pure function: GitHub data for one map → `{relative path: file
  content}`. No I/O.
- **write** — applies a translated map to disk atomically and enforces folder ownership.
- **hook filter** — a pure function: hook JSON + registered projects → sync or skip.
- **runner** — locking, dirty-flag coalescing, logging, pause handling, error capture.
- **cli** — argument parsing and the commands above.

The project's GitHub repo is the `origin` remote, as resolved by
`gh repo view --json nameWithOwner` run in the project folder.

## Translation: GitHub → files

**Maps.** Every open issue labelled `wayfinder:map` becomes a folder
`.plan/maps/<issue number>-<title slug>/`. When a map issue closes or disappears, its
folder is removed on the next successful sync.

**Slugs.** Lowercase the title, replace each run of non-alphanumeric characters with `-`,
trim leading and trailing `-`, and truncate to 60 characters. An empty result becomes
`untitled`. When a title changes, the file or folder is renamed on the next sync.

**`map.md`.** The map issue's title as `# H1`, a blank line, then the issue body verbatim.
GitHub links are left as-is; chartr opens `https` links in the browser.

**Tickets.** Each direct sub-issue of the map becomes
`tickets/<issue number>-<title slug>.md`:

```markdown
---
type: <from the wayfinder:<type> label, or "unknown">
blocked_by: [<blocking issue numbers that are also tickets in this map>]
undermined_by: []
assets: []
claimed_by: github:<assignee login>      # only while open and assigned
claimed_at: <RFC 3339 time of that assignment>   # only with claimed_by
---
# <issue title>

## Question
<issue body, minus any "## Done when" section>

<if any blockers are outside this map: "Also blocked by (outside this map): <links>">

## Done when
<the body's own "## Done when" section, or "_Not specified on GitHub._">

## Answer            ← closed and resolved
<the issue's last comment, or "_Closed on GitHub without a resolution comment._">

## Ruled out         ← instead of Answer, when ruled out (see below)
<the issue's last comment, or the same placeholder>

[View on GitHub](<issue url>)
```

- A closed ticket is **ruled out** when its state reason is "not planned", or when the
  map body's **Out of scope** section links to it. Otherwise it is **resolved**.
- If the issue body already uses `## Question` as its first heading, that heading is not
  duplicated.
- Multiple assignees: the first assignee is used.
- No `status` field is ever written.

**Known divergence.** GitHub counts any closed blocker as cleared. chartr clears only
resolved blockers, so a ticket blocked by a ruled-out ticket can look blocked in chartr
while it is unblocked on GitHub. This is accepted.

## Writing

- Each marked map folder contains a `.wayfinder-sync` marker file. The tool only rewrites
  or deletes folders that carry the marker. Unmarked folders in `.plan/maps/` are never
  touched.
- A map is written by building the complete folder under
  `.plan/maps/.wayfinder-sync-tmp/`, then replacing the old folder by rename.
- If the rendered content for a map is identical to what is on disk, nothing is written,
  so chartr does not redraw needlessly.

## Triggers

**Hook.** A global `PostToolUse` hook with matcher `Bash` runs `wayfinder-sync hook`.
- It syncs only if the command contains `gh issue` or `gh api`, and the hook's `cwd` is
  inside a registered project.
- It detaches the sync into a background process and exits immediately with status 0
  and no output, in every case, including internal errors.
- Before fetching, the background sync waits 2 seconds to let GitHub's sub-issue and
  dependency links settle.

**Timer.** The LaunchAgent runs `wayfinder-sync all` every 180 seconds (`StartInterval`)
and at login.

**Concurrency.** One lock per project, shared by every trigger. If a sync is already
running when another trigger arrives, a dirty flag is set instead, and the running sync
does one more pass when it finishes.

**Pause.** While `~/.config/wayfinder-sync/paused` exists, the hook and timer exit
immediately. Manual `wayfinder-sync` still runs and says that syncing is paused.

## Error handling

- **Atomicity.** Any failure while fetching or translating a map leaves its existing
  folder unchanged. A GraphQL response containing `errors` counts as a failure.
- **Visible staleness.** On failure, a single line is inserted directly under the H1 of
  the existing `map.md`:
  `> ⚠ Last sync from GitHub failed at HH:MM (<reason>). Showing the copy from HH:MM.`
  The next successful sync rewrites `map.md` without it.
- **Logged out.** Logged at most once per hour, with the hint `gh auth login`.
- **Rate limited.** Syncs are skipped until the reset time GitHub reports.
- **No remote, or issues disabled.** The project is skipped and logged, and `doctor`
  shows ✗.
- **Registered folder missing.** Skipped; `doctor` suggests `wayfinder-sync remove`.
- **Malformed data is copied, not fixed.** A missing type label becomes `type: unknown`.
  Missing map sections and blocker cycles are copied as they are, so chartr's own
  warnings surface them.
- **Crashes.** Caught at the top level and logged with a traceback. The hook still exits 0.
- **Log size.** Rotated at 1 MB, keeping one older file.

## Testing

- Python `unittest` with no third-party packages, run by `./test.sh`. Each unit is built
  test-first.
- **translate** is tested against GitHub responses saved in `tests/fixtures/`. They are
  recorded from real queries against the private test repo, not hand-written. Cases:
  claims, blockers outside the map, closed without a comment, ruled out (by state reason
  and by Out of scope link), renames, missing labels, empty titles, paging.
- **write** is tested in temporary folders: failure leaves the old copy, unmarked folders
  are untouched, unchanged maps are not rewritten, the warning line is added and removed.
- **hook filter** is tested on sample hook inputs.
- **install / uninstall** run against temporary `HOME` folders. Tests check that other
  `settings.json` content survives both directions byte-for-byte.
- **Convention validator.** Every generated map is checked against chartr's tracker
  convention: filename pattern, five map sections in order, required ticket sections,
  no `status` field, every blocker present.

### Acceptance (manual, end-to-end)

Using a private throwaway repo `wayfinder-sync-test`, opened as a chartr space:

1. `wayfinder-sync add`. Claude Code charts a three-ticket map with the wayfinder skill;
   it appears in chartr within seconds.
2. Work one ticket. The claim ring appears, then the star turns resolved.
3. Edit an issue on github.com. It appears within 3 minutes.
4. Go offline. The warning line appears. Reconnect; it clears.
5. `wayfinder-sync remove`, then delete the repo (`gh auth refresh -s delete_repo`).

The repo creation and deletion each need the user's confirmation at the time.

## Prerequisites outside this tool

- `gh auth login`.
- The wayfinder skill's GitHub tracker configured for each project. The skill tells the
  user to run `/setup-matt-pocock-skills`, which is not installed, and otherwise falls
  back to its local-Markdown tracker. Configuring it is part of the implementation plan.
