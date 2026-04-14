# railway-operator

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-8A2BE2.svg)](https://claude.com/claude-code)
[![Model](https://img.shields.io/badge/model-Opus%204.6-orange.svg)](https://www.anthropic.com/claude)

A [Claude Code](https://claude.com/claude-code) plugin that dispatches an **Opus 4.6** agent to execute any Railway infrastructure task end-to-end. The agent follows a strict escalation ladder: **CLI first**, **GraphQL API** when the CLI lacks a mutation, **Playwright** against the Railway dashboard for UI-only actions. Every mutation is verified, every known gotcha is applied automatically.

## What it does

When you invoke the `railway-op` skill (or ask Claude to do something on Railway), the plugin:

1. **Context gathering** -- captures Railway auth state, project link, git state, project type hints
2. **Agent dispatch** -- fresh-context Opus 4.6 agent with the full Railway knowledge baked in
3. **Execution** -- the agent plans, executes, and verifies each step using the escalation ladder
4. **Structured report** -- task, actions taken, verification results, gotchas avoided, follow-up items

## Installation

```bash
# 1. Add this repo as a marketplace
claude plugin marketplace add https://github.com/TheMizeGuy/railway-operator-public.git

# 2. Install the plugin
claude plugin install railway-operator@railway-operator

# 3. Restart Claude Code for the plugin to load
```

After restart, verify with `claude plugin list`.

### Prerequisites

- **Railway CLI** installed and authenticated (`railway login`)
- **Playwright MCP plugin** for Claude Code (needed for dashboard-only actions like webhooks, audit logs, billing)

### Optional enhancements

- **GoodMem MCP** -- if configured, the agent queries your Learnings space for Railway-specific gotchas and fixes from prior sessions
- **Local Railway knowledge base** -- if you maintain Railway reference docs (Obsidian vault, markdown folder), pass the path and the agent will read and cite them

## Usage

| Invocation | What it does |
|---|---|
| `/railway-op deploy this` | Detect project type, create/link project, set variables, deploy, verify healthy, add domain |
| `/railway-op fix the 502 on backend` | Read logs, classify failure, apply fix, redeploy, verify |
| `/railway-op add a custom domain for api.example.com` | Add domain, return DNS records to configure |
| `/railway-op enable PR deploys` | GraphQL mutation (CLI doesn't expose this) |
| `/railway-op set up a webhook to Slack` | Playwright-driven -- prompts you to sign in to the Railway dashboard |
| `/railway-op status` | Enumerate projects, services, environments, deployment states |
| `/railway-op add postgres with redis` | Add managed databases, wire connection variables |
| `/railway-op restore latest backup` | Identify backup, restore, verify |

Natural-language triggers also work: "deploy this to Railway", "my Railway service is down", "what's running on Railway", "check Railway logs".

## Escalation ladder

The agent always tries the simplest tool first and escalates only when needed:

| Level | When | Examples |
|---|---|---|
| **Railway CLI** | Always first | `railway up`, `railway variable set`, `railway logs`, `railway domain` |
| **GraphQL API** | CLI lacks the mutation | `tcpProxyCreate`, `projectUpdate` (rename, PR deploys), `serviceUpdate` (rename, icon), `metrics` |
| **Playwright** | UI-only actions | Webhook setup, audit logs, billing, SSO, member management, template publishing |

## Built-in gotchas (17 known sharp edges)

The agent knows and routes around these automatically. A few highlights:

| Gotcha | What happens if you don't know |
|---|---|
| `railway volume add` doesn't redeploy | Writes go to overlay filesystem, lost on restart |
| Postgres needs `PGDATA` subdirectory | `initdb` fails silently because `lost+found/` makes mount "non-empty" |
| Private DNS is IPv6-only (older envs) | Connection refused on apps that only bind IPv4 |
| `railway ssh` re-tokenizes args | Quoted SQL loses its boundaries at the remote shell |
| Cloudflare SSL "Flexible" + Railway | Infinite redirect loop (`ERR_TOO_MANY_REDIRECTS`) |
| `railway logs` without bounds | Streams forever, blocks script execution |

Full list of 17 gotchas in the agent definition.

## Destructive-action guardrails

The agent **never** performs destructive actions without explicit user confirmation:

- Delete project/service/domain
- Drop database / wipe volume
- Rename project (changes public URLs)
- Rotate API tokens
- Force-push to auto-deploy branches
- Modify shared production variables
- Restart production during business hours

For these, the agent presents the plan with specific blast radius and waits for "yes do it" or equivalent.

## Verification

Every mutation is verified by reading back the resource:

| After | Verified by |
|---|---|
| Deploy | `service status --json` shows SUCCESS + clean log output |
| Variable set | `variable list --json` returns new value |
| Volume add | `ssh df -hT` shows ext4 (not overlay) |
| Domain add | `domain --json` + DNS records block |
| GraphQL mutation | Read-back query |
| Playwright action | Screenshot + snapshot |

## Components

| Type | Name | Purpose |
|---|---|---|
| Skill | `railway-op` | User-invoked entry point; gathers Railway + project context, dispatches the agent |
| Agent | `railway-operator` | Opus 4.6 executor with CLI + GraphQL + Playwright tools, 17 gotchas, verification loop |

## Tool access

The agent has broad tool access for the escalation ladder:

| Tool | Purpose |
|---|---|
| `Bash` | Railway CLI commands, GraphQL via curl, git operations |
| `Read`, `Grep`, `Glob` | Read project files, Railway config, knowledge base |
| `Edit`, `Write` | Modify project config files (railway.toml, Dockerfile, etc.) when needed |
| `TodoWrite` | Track multi-step task progress |
| `WebSearch`, `WebFetch` | Check Railway docs, status page |
| Playwright MCP tools | Drive the Railway dashboard for UI-only actions |
| GoodMem MCP tools (optional) | Query/write Railway learnings |

## License

MIT. See [LICENSE](LICENSE).

## Credits

Built by [mize](https://github.com/TheMizeGuy). Backed by the [Claude Code](https://claude.com/claude-code) plugin system and Anthropic's Opus 4.6 model.
