---
name: airflow-failed-run
description: Diagnose a failed Apache Airflow DAG run on SkaleData, from the app and the run to the failed task and its log, and, with the person's approval, clear the failed tasks or trigger the DAG again. Use when someone asks why a DAG or task failed, what went wrong with a run, whether a DAG is healthy, or to retry, clear, or re-run an Airflow DAG.
---

# Diagnose a failed Airflow run

Work from evidence the SkaleData MCP server returns, never from guesses about
what a DAG does. Name the app, the DAG, the run, the task, and the log lines
that show the cause. Change nothing without the person's approval.

The tool names below are the SkaleData server's. Your client may show them
with a prefix, such as `mcp__plugin_skaledata_skaledata__airflow_list_dags`.

## Before you start

The `airflow_*` tools act as the person who signed in, with that person's own
Airflow role for the app. They refuse API keys. If a tool answers "Airflow
tools need a person signed in through an AI client such as Claude", the
connection uses an API key: stop, say so, and point the person to
https://docs.skaledata.com/connect-claude. Don't look for another way in.

If every tool answers "MCP data egress is disabled for this organization",
an org admin has to turn on **Allow MCP data egress** in the SkaleData
console, under **Organization** > **General**. Stop and say so.

## 1. Find the Airflow app

Call `skaledata_list_apps` with `app_type: "airflow"`.

- One app: use it.
- Several apps: use the one the person named. If they didn't name one, ask
  which, listing the names and clusters. Don't pick for them.
- Pass the app's id or exact name as `app` to every `airflow_*` call. Pass
  `cluster` too only when two apps share a name.
- If the app's status isn't `running`, call `skaledata_get_app_health` with
  `app` before going on: a failed scheduler heartbeat or database signal can
  explain a run that never started, and a stopped app has nothing to query.

## 2. Find the DAG

- If the person named the DAG, call `airflow_get_dag` with `dag_id`.
- Otherwise call `airflow_list_dags` with `dag_id_pattern` set to the words
  they used. Airflow treats `%` and `_` in the pattern as wildcards. Page with
  `offset` set to `next_offset` when there are more.
- Note `is_paused`, `has_import_errors`, and the schedule.
- If `has_import_errors` is true, call `airflow_list_import_errors`. The
  stack trace names the file and the line. Report it, and say when it
  happened (its timestamp). Then keep going: an import error stops Airflow
  from creating new runs, and it can start after the run the person asked
  about already failed, so it isn't the cause of a run that exists. Only
  when the person asks about a run that never appeared, and the import error
  explains it, is the import error the diagnosis; the fix is in the DAG
  file, not in Airflow.

## 3. Find the run

Call `airflow_list_dag_runs` with `dag_id`. Runs come newest first.

- Use `state: "failed"` to list only failed runs, and call once more without
  the filter when you need to know whether a later run succeeded.
- Match the run to the person's words by `logical_date`, `run_after`,
  `start_date`, and `end_date`. Times are UTC; "this morning" is in the
  person's time zone. If two runs fit, name both and ask.
- Note `run_type` and `triggered_by` or `triggering_user_name`: a manual run
  can carry parameters a scheduled run doesn't.
- Call `airflow_get_dag_run` with `dag_id` and `dag_run_id` when you need the
  run's `conf` or data interval.

A run can be `failed` with no failed task, for example when it exceeded the
DAG's run timeout. Then list the task instances without a state filter and
look for tasks that never finished.

## 4. Find the failed task

Call `airflow_list_task_instances` with `dag_id`, `dag_run_id`, and
`state: "failed"`.

- A `failed` task is a cause. An `upstream_failed` task only waited on one;
  don't diagnose it on its own.
- Note each task's `try_number`, `max_tries`, `operator`, timing, and
  `map_index`. A mapped task has one instance per index; a map index of -1
  means the task isn't mapped.
- Several failed tasks: start with the earliest `start_date`. The rest are
  often the same cause.

## 5. Read the log

Call `airflow_get_task_log` with `dag_id`, `dag_run_id`, and `task_id`. Add
`map_index` for a mapped task.

- Without `start_line`, the tool returns the end of the log, where the error
  usually is. If the end only shows cleanup, read earlier lines by calling
  again with `start_line` set to a smaller line number; the result's
  `first_line`, `last_line`, `total_lines`, and `next_start_line` tell you
  where you are, and its note suggests a value.
- The tool reads the latest try. To compare with an earlier try, set
  `try_number`.
- A log can contain whatever the task printed, including data. Quote only
  the lines that show the error, with their line numbers.

## 6. Report before you change anything

Say, in this order: the app, the DAG, the run (`dag_run_id` and its logical
date), the failed task, the error lines, and the likely cause. Then say which
of these it looks like, and why:

- **Transient**: a connection reset, a timeout, a pod that was evicted or ran
  out of memory, a rate limit, or a dependency that was briefly down. Running
  the task again is likely to succeed.
- **Code or configuration**: an exception in the task's own code, a missing
  variable or connection, a wrong path or table name. Running it again fails
  the same way until the DAG or its configuration is fixed.
- **Upstream data**: the task ran, but its input wasn't there yet. Running it
  again helps only once the input exists.
- **Import error**: the DAG file doesn't load now (step 2), so no new run
  will start until it's fixed. It explains a missing run, not a run that
  ran and failed, unless the run's own log shows the same error.

If you can't tell, say what you'd need to see.

## 7. Fix it, with approval

Before any change, tell the person exactly what you'll do, name the DAG, the
run, and the tasks, and wait for their explicit approval. Every change is
audited under the person's email address.

**Clear the failed tasks** (the usual fix for a transient failure, and the
one that keeps the run's logical date and data interval):

1. Call `airflow_clear_task_instances` with `dag_id`, `dag_run_id`, and
   `dry_run: true`. By default it selects failed and upstream-failed tasks
   (`only_failed`). Narrow it with `task_ids` (at most 20), or widen it with
   `include_downstream`.
2. Show the person the tasks the dry run listed, and get their approval.
3. Call it again with the same arguments and `dry_run: false`.

**Trigger a new run** (when the person wants a fresh run rather than a
repaired one, or the DAG has no run to repair): call `airflow_trigger_dag_run`
with `dag_id`, and `conf` or `logical_date` only if the person asked for
them. The new run's id is `mcp__` followed by the audit request id; find it
with `airflow_get_dag_run`. A paused DAG's run stays queued until the DAG is
unpaused: say so, and call `airflow_unpause_dag` only if the person approves
that too.

**Refusals**, which you don't retry:

- "Your Airflow role doesn't allow you to ...": Airflow answered 403. The
  person's role for this app doesn't allow the change. An org admin can change
  it in the app's **User Management** tab in the SkaleData console.
- "Airflow app ... doesn't enforce per-user Airflow roles": SkaleData refuses
  every change tool on this app before anything reaches Airflow, because the
  app runs an external image or has identity turned off. The read tools still
  work. The person makes the change in the Airflow UI, or an org admin
  switches the app to the SkaleData Airflow image with identity on.

If a change's result is uncertain, don't call it again. Check the state with
`airflow_list_task_instances` or `airflow_get_dag_run` first.

## 8. Confirm

- After a clear: `airflow_list_task_instances` for the run shows the cleared
  tasks back in a waiting state, and later `running`, `success`, or `failed`.
- After a trigger: `airflow_get_dag_run` with the new run id shows `queued`,
  then `running`.

Say what you saw. Don't promise that the task will succeed.

## Don't

- Don't ask the person to paste logs or screenshots: read the log yourself.
- Don't pause or unpause a DAG unless the person asked, or a trigger needs it
  and they approved it.
- Don't clear every task in a run when only one failed, unless the person
  wants a full rerun.
- Don't read more of a log than the diagnosis needs, and don't repeat data
  from it.
