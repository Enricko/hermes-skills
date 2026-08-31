# Recovering an Overwritten File from Claude Code Session Logs

When Claude Code edited a file (between your turns) and you overwrote it with your own
version, recover the last Claude Code version from its session JSONL. Validated 2026-08-29
on `/root/hematcuy/backend/app/api/v1/redirect.py` — all 28 routing tests passed after recovery.

## Where the logs live
`/root/.claude/projects/-root-<project-dir-slug>/*.jsonl` — one file per Claude session,
newest usually largest. Frontend-only runs live under `-root-<project>-frontend/`. Pick the
JSONL that was being written at the time of the user's "saya sudah ubah lewat claude code".

## What each line contains
Every line is one JSON event with a `message` object:
- `message.role == "assistant"`, `content[].type == "tool_use"`:
  - `name: "Write"` → `input.content` holds the FULL file (easiest recovery).
  - `name: "Edit"` / `"MultiEdit"` → `input.old_string` / `input.new_string` (incremental patches).
- `message.role == "user"`, `content[].type == "tool_result"` with content starting `1\tfrom ...`
  → a `read_file` snapshot of the file at that point (strip the `^\d+\t` line-number prefixes).

Claude Code usually works in small Edits, not full Writes — so reconstruct:

1. Find the newest Read snapshot BEFORE the batch of edits (clean base).
2. Apply every later Edit in log order with `content.replace(old_string, new_string, 1)`.
3. If an `old_string` no longer matches, the reconstruction diverged — find the edit's
   `parentUuid` chain or use the next Read snapshot instead.
4. Save, then run the project test suite — the tests often import the recovered module's
   functions directly (e.g. `tests/test_grocery_routing.py` imports
   `_get_clean_redirect_url` and `build_gomart_search_url`), so green tests = correct recovery.

## Why recover instead of reimplement
The Claude Code version is usually smarter than a naive rewrite: per-item detail-page
priority, affiliate_url precedence, optional `target=` query param, helper functions that
tests import. A rewrite that "simplifies" these silently breaks the routing tests.
