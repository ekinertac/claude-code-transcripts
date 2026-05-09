# Folder-grouped picker for the `local` command

**Date:** 2026-05-10
**Status:** Approved, pending implementation
**Scope:** `claude-code-transcripts local` only. Other commands (`web`, `json`, `all`) are untouched.

## Problem

`claude-code-transcripts local` shows a flat, mtime-sorted list of recent sessions across every project under `~/.claude/projects/`. With ~40 project folders, identifying which session belongs to which project from its first-prompt summary alone is hard. The list is also capped at `--limit` (default 10), hiding older sessions even within a single project.

## Goal

Make it easy to pick a session by first identifying the project it came from, and remove the limit so every session in a chosen project is visible.

## User-facing flow

Two-step picker:

1. **Project picker** — sorted by most recent activity. Each row: `<display-name>   <latest-mtime>   <N sessions>`. Colliding display names get a trailing `(<raw-folder-name>)` disambiguator.
2. **Session picker** — every session in the chosen project, sorted newest first. Row format unchanged from today: `<date>  <kb>  <summary>`.

Cancellation (Ctrl-C / Esc) at either step exits cleanly with a one-line message.

## Code shape

All changes live in `src/claude_code_transcripts/__init__.py`. The project keeps everything in one module; this design preserves that.

### New helper: `find_local_projects(folder)`

Placed near the existing `find_local_sessions`.

- Iterates top-level entries of `folder` via `iterdir()` — does **not** recurse into project folders for sessions, only globs `*.jsonl` one level deep.
- Filters out `agent-*.jsonl` files.
- Skips folders with zero matching sessions.
- For each remaining folder, computes `latest_mtime = max(f.stat().st_mtime for f in jsonls)` — folder mtime is unreliable on some filesystems.
- Returns a list of dicts: `{name, raw_name, path, session_count, latest_mtime}`, sorted by `latest_mtime` descending.
- **Does not read session summaries.** This is the speed win: the project picker is built from `stat()` calls only.
- Display name is produced by the existing `get_project_display_name()` helper.

### Collision handling

After building the list, post-process: group by `name`, and for any name appearing more than once, append `  ({raw_name})` to those rows' display strings only. Non-colliding rows stay clean.

### Modified `local_cmd`

- Replace the single `questionary.select` with two:
  1. Build choices from `find_local_projects(projects_folder)`, prompt `"Select a project:"`.
  2. Call `find_local_sessions(selected_project["path"], limit=None)` and prompt `"Select a session:"`.
- Change `find_local_sessions` so `limit=None` means "no cap"; existing default behaviour for callers is preserved by not changing the default value.
- Remove the `--limit` CLI option from `local_cmd`. The user explicitly asked to "see all sessions"; keeping a dead flag would rot. README gets a one-line update.

## Data flow

```
local_cmd
  └─ find_local_projects(~/.claude/projects)        # cheap: iterdir + stat
  └─ questionary.select(projects)                    → selected_project
  └─ find_local_sessions(selected_project["path"],
                         limit=None)                  # reads summaries for one project
  └─ questionary.select(sessions)                    → selected_session
  └─ generate_html(...)                               # unchanged
```

## Error handling and edge cases

- `~/.claude/projects` missing → existing message, return. (unchanged)
- `find_local_projects` returns `[]` → `"No local projects found."` and return.
- Ctrl-C / Esc on project picker → `selected is None` → `"No project selected."` and return.
- Ctrl-C / Esc on session picker → existing `"No session selected."` and return.
- Project shows in list but `find_local_sessions` returns `[]` because every session is warmup/agent → print `"No sessions in <project>."` and return. (No loop-back; user re-runs.)
- Permission errors during folder `stat()` → skip that folder silently, matching `find_local_sessions`'s existing resilience.

The "project listed but session picker empty" case can occur because `find_local_projects` does not read summaries, so it cannot detect warmup-only projects. This is a deliberate trade-off: avoiding the summary read is the whole point. The case is rare and the failure mode is benign.

## What is not changing

- `find_local_sessions` signature — only its `limit` default semantics (`None` means unlimited). The only existing caller is `local_cmd`.
- `find_all_sessions`, the `all` command, `web`, `json` — untouched.
- HTML generation and all output flags (`--gist`, `--repo`, `--open`, `-o`, `-a`, `--json`) — untouched.
- The existing `get_project_display_name()` helper is reused as-is.

## Testing

Per project convention (TDD: failing test → watch fail → make pass), tests are added to `tests/test_generate_html.py`, using the existing `tmp_path` patterns.

### New: `TestFindLocalProjects`

1. `test_returns_empty_for_missing_folder` — non-existent path returns `[]`.
2. `test_returns_empty_for_empty_folder` — folder exists, no children, returns `[]`.
3. `test_skips_folders_with_no_jsonl` — folder with only non-jsonl files is excluded.
4. `test_skips_folders_with_only_agent_files` — folder with only `agent-*.jsonl` is excluded.
5. `test_counts_sessions_correctly` — 3 jsonls + 1 agent file → `session_count == 3`.
6. `test_latest_mtime_is_max_session_mtime` — set known mtimes via `os.utime`, assert `latest_mtime` is the newest.
7. `test_sorted_by_latest_mtime_desc` — multiple projects with staggered mtimes, assert order.
8. `test_display_name_uses_helper` — folder named `-Users-x-Code-foo` → `name == "foo"`.
9. `test_collision_appends_disambiguator` — two folders that collapse to the same display name → both carry a raw-folder suffix in their display string; non-colliding rows do not.
10. `test_does_not_read_session_summaries` — monkeypatch `get_session_summary` to raise; `find_local_projects` still succeeds. Proves laziness.

### Updated: `TestFindLocalSessions`

- `test_limit_none_returns_all` — passing `limit=None` returns every session, not the default cap.

### Not tested

`local_cmd` end-to-end — the existing codebase has no CLI-flow tests for it, and exercising two `questionary.select` calls via `CliRunner` adds harness complexity without much signal. Logic lives in the helpers; that is where coverage goes.

### Manual smoke test before commit

`uv run claude-code-transcripts` → project picker shows real folders → pick one → session picker → render HTML → browser opens correctly.

## Documentation

Update `README.md`:
- Replace the `--limit` line in the "Local sessions" section with a one-line description of the new two-step flow.
- Remove the `claude-code-transcripts local --limit 20` example.

## Out of scope

- A `--project` flag to skip the project picker. Possible follow-up; not needed now.
- Search/filter inside the picker. Questionary's built-in arrow navigation is sufficient at ~40 projects.
- Changes to `find_all_sessions`, `web`, `json`, `all`.
