---
name: airbyte-stale-table
description: Check the Airbyte sync behind a table that looks stale, empty, or behind on SkaleData. Finds the connection and stream that feed the table, reads its last jobs and their logs, explains why the data is behind, and, with the person's approval, starts a sync. Use when someone asks whether a sync is healthy, why a table hasn't updated, when data last landed, or to run an Airbyte sync again.
---

# Check the Airbyte sync behind a stale table

Work from what the SkaleData MCP server returns. Name the app, the
connection, the stream, the job, and the lines that show the cause. Change
nothing without the person's approval, and never handle a secret.

The tool names below are the SkaleData server's. Your client may show them
with a prefix, such as `mcp__plugin_skaledata_skaledata__airbyte_list_jobs`.

## Before you start

The `airbyte_*` tools act for the person who signed in. They refuse API keys.
If a tool answers "Airbyte tools need a person signed in through an AI client
such as Claude", the connection uses an API key: stop, say so, and point the
person to https://docs.skaledata.com/connect-claude.

Airbyte has no per-user roles. Anyone who can open the app can use every
tool, including the ones that delete data in the destination. That makes the
person's approval the only check on a change, so ask for it every time.

If every tool answers "MCP data egress is disabled for this organization",
an org admin has to turn on **Allow MCP data egress** in the SkaleData
console, under **Organization** > **General**. Stop and say so.

## 1. Find the Airbyte app

Call `skaledata_list_apps` with `app_type: "airbyte"`.

- One app: use it.
- Several apps: use the one the person named, or ask which, listing the
  names and clusters.
- Pass the app's id or exact name as `app` to every `airbyte_*` call. Pass
  `cluster` too only when two apps share a name.
- If the app's status isn't `running`, call `skaledata_get_app_health` with
  `app` first.

## 2. Find the connection and the stream behind the table

Call `airbyte_list_connections`. Page with `offset` set to `next_offset` when
there are more. Add `include_deleted: true` only when a connection the
person remembers isn't listed.

- Match the table to a connection by name first: connections are often named
  after their source or destination.
- Confirm the match with `airbyte_get_connection` and `connection_id`. Its
  `streams` list each stream's `name`, `namespace`, `sync_mode`,
  `cursor_field`, and `primary_key`. The destination table is usually the
  stream's name with the connection's `prefix` in front, in the namespace
  that `namespace_definition` and `namespace_format` set.
- The tool lists at most 100 streams, and `streams_omitted` says how many
  more the connection has. When `streams_omitted` is above 0 and the table's
  stream isn't in the list, the list is incomplete, not proof that the
  stream is missing: say so, and ask the person to confirm the stream from
  the connection's `ui_url`, or match it by the connection's name and the
  source. Don't conclude that Airbyte isn't syncing the table from a
  truncated list.
- Note the connection's `status` and `schedule`. `inactive` means the
  schedule doesn't run. A `manual` schedule means nothing runs unless someone
  starts a sync. A `cron` schedule is in UTC. `deprecated` means the
  connection was deleted.
- Note the stream's `sync_mode`. An incremental mode reads only rows past the
  cursor, so a stale table with a succeeded sync can mean the source's cursor
  column didn't move. An overwrite mode replaces the table at every sync.

If no connection has a stream for the table, and no connection's list was
truncated, say so: Airbyte isn't syncing it. To see what the source offers,
call `airbyte_list_streams` with the connection's `source_id` and
`destination_id`; it can take a minute.

## 3. Read the last jobs

Call `airbyte_list_jobs` with `connection_id`. Jobs come newest first; without
`job_type`, Airbyte lists syncs and resets. Use `status` to list only failed
ones when the history is long.

For each recent job, note `status`, `job_type`, `start_time`,
`last_updated_at`, `duration`, `rows_synced`, and `bytes_synced`. Then answer
these from the evidence:

- **When did the connection last sync?** The newest `succeeded` sync's
  time. Compare it with the schedule: a gap longer than the schedule means
  the connection is inactive, the schedule is manual, or jobs have been
  failing or cancelled since.
- **Did that sync reach this table?** A succeeded job counts rows for the
  whole connection, not per stream, so its time says when the connection
  last synced, not when this table last got data. A job with `rows_synced`
  of 0, or a stream whose incremental cursor didn't move, can succeed
  without touching the table. Report table freshness as unknown unless
  something stream-specific shows it: a failure that names the `stream` in
  `airbyte_get_job`, the stream's name in the job log, or what the person
  sees in the destination.
- **Is a job running now?** A `running` or `pending` job may be the sync the
  person is waiting for. Say so instead of starting another: Airbyte refuses
  a second sync with HTTP 409 while one runs.
- **Did the last sync move rows?** A `succeeded` sync with `rows_synced` of
  0 points at the source or the cursor, not at Airbyte. Starting another
  sync won't change that.

## 4. Diagnose a failed job

Call `airbyte_get_job` with `job_id` (a whole number from
`airbyte_list_jobs`). It returns the job's 10 newest attempts, each with its
`attempt_number` (from 0), `status`, `records_synced`, and `failures`. Each
failure has an `origin` (the source, the destination, or Airbyte itself), a
`type`, the `stream` it names, and Airbyte's `external_message` and
`internal_message`.

Then call `airbyte_get_job_log` with `job_id`. It reads the latest attempt,
and the end of the log, where the error usually is. Set `attempt_number` for
an earlier attempt, and `start_line` to read earlier lines; the result's
`first_line`, `last_line`, `total_lines`, and `next_start_line` tell you
where you are.

A job log holds whatever the connector printed, and SkaleData doesn't remove
secrets from logs. Quote only the lines that show the error, and never quote
a value that looks like a password, key, or token.

Common causes, and where the fix is:

- **Credentials or settings of the source or destination** (authentication
  failed, an expired token, a host that doesn't resolve, permission denied):
  the fix is in Airbyte's own form. Call `airbyte_secret_entry_link` with
  `kind` (`source` or `destination`) and `id` (the connection's `source_id`
  or `destination_id`), and give the person the link. Don't ask them to paste
  a secret into the chat. To check a non-secret setting such as the host or
  the database name, call `airbyte_get_source` or `airbyte_get_destination`;
  they return settings with secrets removed.
- **The source rejected a query** (a table or column that no longer exists,
  a schema change): the source changed. Say what the message names.
- **The destination refused to write** (out of space, a type mismatch, a
  permission): the fix is on the destination.
- **The job was cancelled, or the pod died**: a sync can simply be run
  again.
- **A failure marked `retryable`**: Airbyte already retried it within the
  job; the attempts show how many times.

## 5. Report before you change anything

Say, in this order: the app, the connection (its name and `ui_url`), the
stream and its sync mode, the schedule and status, when the connection last
synced successfully and whether that shows the table got data, the failing
job id with the evidence lines, and the likely cause. Then say
what would fix it, and whether that's a sync, a change in Airbyte's UI, or
something outside Airbyte.

## 6. Fix it, with approval

Before any change, tell the person exactly what you'll do and name the
connection; wait for their explicit approval. Airbyte records every change as
its default user, and the SkaleData audit record names the person.

**Start a sync** (`airbyte_trigger_sync` with `connection_id`): the usual
fix after a transient failure, a cancelled job, or a missed manual sync.
Before you ask, say which streams have an overwrite sync mode, because a
sync replaces their tables in the destination. If `airbyte_get_connection`
reported `streams_omitted` above 0, you haven't seen every stream: say that
the list is incomplete and that the sync covers every stream of the
connection, including the ones you couldn't list, and let the person decide
with that in view. The result names the job
Airbyte started; follow it with `airbyte_get_job`. If the sync is refused
with HTTP 409, a job is already running: don't cancel it unless the person
asks.

**Turn the schedule back on** (`airbyte_update_connection` with
`connection_id` and `status: "active"`, or `schedule_type: "cron"` with
`cron_expression` in UTC): when the connection is inactive or has no
schedule, and the person wants syncs to run on their own. Name the exact
change in the approval.

**Cancel a stuck job** (`airbyte_cancel_job` with `job_id`): only when the
person asks. Airbyte cancels whatever job the connection is running when the
cancel arrives, and the tool refuses a job that has already finished.

**Clear streams** (`airbyte_clear_streams`): only when the person asks for a
reset and understands it. It deletes the named streams' data in the
destination and reads them from the start at the next sync. Name the
connection and each stream in the approval, pass them as `streams` with each
stream's exact `name` and `namespace` from `airbyte_get_connection`, and
pass `all_streams: true` only when the person wants every stream cleared.

Don't create, delete, or re-point a connection to fix a stale table unless
the person asked for that.

If a change's result is uncertain, don't call it again. Check
`airbyte_list_jobs` for the connection first.

## 7. Confirm

Call `airbyte_get_job` with the new job's id. Report its `status` and, once it
finishes, `records_synced` from its last attempt. A sync result confirms
that the job started, not that it finished, so say what you saw.

## Don't

- Don't ask the person for a password, key, or token, and don't repeat one
  from a log. Send them to `airbyte_secret_entry_link`.
- Don't start a sync while one runs, and don't start one to "see what
  happens" on a connection whose last failure was a credential or a schema
  change: it fails the same way.
- Don't clear streams or delete a connection as a shortcut.
- Don't read more of a log than the diagnosis needs.
