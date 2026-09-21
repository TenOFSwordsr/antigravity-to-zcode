# Antigravity → ZCode Chat Importer

A one-file migration script that replays Google Antigravity conversation history into the ZCode
CLI's local session database, so past agent chats stay browsable after switching tools. It reads
Antigravity's per-chat SQLite files, its `transcript.jsonl` event logs and its protobuf
title index, rebuilds them as ZCode `session` / `message` / `part` rows, and writes them into
`~/.zcode/cli/db/db.sqlite`. `titles.json` is a dump of the decoded conversation-title map
(118 chats) kept for reference.

**Suggested repo name:** `antigravity-to-zcode`
**Stack:** Python 3, stdlib only (`sqlite3`, `json`, `uuid`) plus a hand-rolled protobuf varint parser
**Status:** finished
**Last modified:** 2026-09-05

## What it does

- `load_titles()` - parses `~/.gemini/antigravity/agyhub_summaries_proto.pb` with a
  field-number/wire-type walker (`read_varint`, `pb_parse`) to recover each conversation's title,
  avoiding a protobuf dependency.
- `load_events()` - reads `~/.gemini/antigravity/brain/<uuid>/.system_generated/logs/transcript.jsonl`.
- `convert_session()` - maps Antigravity event types onto chat messages: `USER_INPUT` → user
  prompt, `PLANNER_RESPONSE` → assistant text, `RUN_COMMAND` / `CODE_ACTION` / `VIEW_FILE` /
  `SEARCH_WEB` / `LIST_DIRECTORY` / `FIND` / `INVOKE_SUBAGENT` / `ASK_QUESTION` → labelled
  assistant tool messages (`TOOL_LABELS`). `EPHEMERAL_MESSAGE` and `CONVERSATION_HISTORY` are dropped.
- Content cleaners strip the `<USER_REQUEST>` / `<ADDITIONAL_METADATA>` wrappers and the
  `Created At:` / `Completed At:` header lines from tool output.
- `main()` - backs the target DB up to `db.sqlite.pre-import` once, then for each conversation
  deletes and re-inserts `sess_<conv_id>` and its rows. **Idempotent**: rerunning refreshes rather
  than duplicating. Commits every 20 sessions and prints an imported / skipped / error tally.
- Imported sessions are pinned to `project_id = proj_c-users-administrator-documents-projects`
  and `directory = C:\Users\Administrator\Documents\Projects`.

## Layout

```
convert.py    the whole importer (~350 lines)
titles.json   conversation-uuid -> title map, decoded from the Antigravity summaries
```

## Running it

Antigravity and ZCode must both have been used on this machine; the paths are resolved from
`os.path.expanduser("~")`.

```bash
python convert.py
```

Expected tail of the output: `imported: N, skipped(empty): N, errors: N` followed by
`custom sessions now in db: N / total sessions: N`.

## Notes

- Point-in-time utility, not a library. Three absolute paths are hardcoded near the top
  (`PROJECT_ID`, `PROJECT_DIR`, `ZCODE_VERSION = "0.16.5"`); the schema assumptions break if
  ZCode changes its session tables, so review `convert_session()` before running it against a
  newer version.
- It writes to a live application database. The only safety net is the one-time
  `db.sqlite.pre-import` backup, which is skipped if it already exists - delete it manually to
  force a fresh backup.
- Nothing here touches the pharmacology work that most of the imported chats are about
  (`titles.json` is full of "Persian Pharmacology PDF Transcription", "Extract Persian Drugs To
  JSON", "Enrich JSON Batch To Pages"); that folder is the transcript archive of those sessions,
  not part of the content pipeline itself.
