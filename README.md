# SkaleData plugin for Claude Code

Connects Claude Code to your SkaleData organization's MCP gateway at
`https://mcp.skaledata.run/mcp`, and teaches Claude two SkaleData workflows:

- `/skaledata:airflow-failed-run`: diagnose a failed Airflow DAG run, and clear
  or re-trigger it with your approval.
- `/skaledata:airbyte-stale-table`: check the Airbyte sync behind a stale
  table, and start a sync with your approval.

Read [Install the Claude Code plugin](https://docs.skaledata.com/claude-code-plugin)
for the full guide, and [What data leaves your cloud](https://docs.skaledata.com/mcp-data-flow)
before an org admin turns on MCP data egress.

## Install

At the Claude Code prompt:

```text
/plugin marketplace add skaledata/claude-plugin
/plugin install skaledata@skaledata
```

Then sign in: run `/mcp`, pick the `skaledata` server, and choose
**Authenticate**. Your browser opens SkaleData's sign-in; sign in, choose the
organization Claude Code acts in, and allow the connection. `/mcp` then lists
the `skaledata` server as connected, and every tool runs as you, with your own
roles. No API key is needed.

## What's in it

| Path                                  | What it is                                                                                 |
| ------------------------------------- | ------------------------------------------------------------------------------------------ |
| `.mcp.json`                           | The gateway as a streamable HTTP MCP server; Claude Code signs you in with OAuth.          |
| `skills/airflow-failed-run/SKILL.md`  | The Airflow workflow.                                                                      |
| `skills/airbyte-stale-table/SKILL.md` | The Airbyte workflow.                                                                      |
| `.claude-plugin/plugin.json`          | The plugin manifest.                                                                       |
| `.claude-plugin/marketplace.json`     | Makes this folder a one-plugin marketplace.                                                |

The Airflow, Airbyte and DataHub tools act as the signed-in person and refuse
API keys, which is why the plugin signs you in instead of taking a key. For CI
and other headless use, add the gateway with an org-wide API key as
[Connect other MCP clients with an API key](https://docs.skaledata.com/mcp-api-keys)
describes; those tools then stay off.

## Develop

The public repo [skaledata/claude-plugin](https://github.com/skaledata/claude-plugin)
is a read-only mirror of this folder, pushed by the `Mirror Claude Code plugin`
workflow after every merge to `main` that touches it. Change the plugin here,
not there.

From a checkout, add the folder as a marketplace and install from it. Claude
Code then reads the plugin from the folder, so an edit takes effect after
`/reload-plugins`:

```bash
claude plugin marketplace add /path/to/skaledata/apps/claude-plugin
claude plugin install skaledata@skaledata
```

Validate the manifests, the MCP configuration, and the skills:

```bash
claude plugin validate apps/claude-plugin --strict
claude plugin validate apps/claude-plugin/skills --strict
```
