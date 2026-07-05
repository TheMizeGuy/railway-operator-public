# railway-operator

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-8A2BE2.svg)](https://claude.com/claude-code)
[![Model](https://img.shields.io/badge/model-session--model-blueviolet.svg)](https://www.anthropic.com/claude)

A [Claude Code](https://claude.com/claude-code) plugin that dispatches an autonomous agent, running on **the session model** (always the strongest available Claude — no hardcoded model dependency), to execute any Railway infrastructure task end-to-end. The agent follows a strict escalation ladder: **CLI first**, **GraphQL API** when the CLI lacks a mutation, **Playwright** against the Railway dashboard for UI-only actions. Every mutation is verified, every known gotcha is applied automatically.

Current version: **0.1.1**.

## What it does

When you invoke the `railway-op` skill (or ask Claude to do something on Railway), the plugin:

1. **Context gathering** — captures Railway auth state, project link, git state, project type hints
2. **Agent dispatch** — a fresh-context agent (running on the session model) with the full Railway operating knowledge baked in
3. **Execution** — the agent plans, executes, and verifies each step using the escalation ladder, ruling out operator error before escalating a rung
4. **Structured report** — task, actions taken, verification results, gotchas avoided, follow-up items

## Installation

```bash
# 1. Add this repo as a marketplace
claude plugin marketplace add https://github.com/TheMizeGuy/railway-operator-public.git

# 2. Install the plugin
claude plugin install railway-operator@railway-operator-public

# 3. Restart Claude Code for the plugin to load
```

After restart, verify with `claude plugin list` and look for `railway-operator@railway-operator-public`.

### Prerequisites

- **Railway CLI** ≥ 4.36.0, installed and authenticated (`railway login`, or let the agent trigger `railway login --browserless` on first use)
- **`jq`** and **`curl`** — used for JSON parsing and the GraphQL escalation path
- **Playwright MCP plugin** for Claude Code — needed for dashboard-only actions (webhooks, audit logs, billing, member management)

### Optional enhancements

- **GoodMem MCP** — if configured, the agent queries your Learnings space for Railway-specific gotchas and fixes from prior sessions, and writes new ones back
- **Local Railway knowledge base** — if you maintain Railway reference docs (an Obsidian vault, a markdown folder), the agent reads and cites them; a suggested layout is documented in the agent definition

Neither optional integration is required — the agent works from its built-in CLI reference and gotcha table regardless.

## Quickstart

1. Install the plugin (above) and confirm `railway` is authenticated: `railway whoami`.
2. From a project directory, say **"deploy this to Railway"** or run `/railway-op deploy this`.
3. The `railway-op` skill gathers context (git state, project type, Railway link/auth status) and dispatches the agent with a self-contained brief.
4. The agent plans the steps with `TodoWrite`, executes via the CLI (escalating to GraphQL or Playwright only when the CLI can't do it), and verifies every mutation by reading the resource back — never trusting an exit code alone.
5. You get a structured report: what was done, what was verified, results (URLs, IDs, DNS records), gotchas avoided, and follow-up items.

## Worked walkthrough

**Request:** `/railway-op add a custom domain for api.example.com`

What happens:

1. The skill gathers context: `railway whoami --json`, `railway status --json`, git state — confirms the project is linked and which service is likely the target.
2. The agent checks the CLI reference: `railway domain <hostname> --service <name>` is Rung 1 (CLI) — no GraphQL or Playwright needed.
3. The agent runs `railway domain api.example.com --service <name>`.
4. **Verification**: re-reads `railway domain --service <name> --json` and confirms the DNS records block was returned — a domain add that doesn't return DNS records isn't done yet.
5. The report:

```
## Task
Add a custom domain for api.example.com

## Actions taken
1. Added custom domain — `railway domain api.example.com --service backend` (CLI, Rung 1)

## Verification
- `railway domain --service backend --json` returned the domain with its DNS record requirements

## Results
- Domain: api.example.com
- DNS records to configure: <CNAME/A record block from the CLI output>

## Gotchas avoided / applied
- None triggered for this action

## Follow-up
- Add the DNS records above at your registrar/DNS provider
- DNS propagation can take up to 24-48h; the domain will show unverified until records resolve
```

A destructive request (e.g., `/railway-op delete the staging project`) without prior authorization stops at the plan stage instead — see [Destructive-action guardrails](#destructive-action-guardrails).

## Usage

| Invocation | What it does |
|---|---|
| `/railway-op deploy this` | Detect project type, create/link project, set variables, deploy, verify healthy, add domain |
| `/railway-op fix the 502 on backend` | Read logs, classify failure via the symptom index, apply fix, redeploy, verify |
| `/railway-op add a custom domain for api.example.com` | Add domain, return DNS records to configure |
| `/railway-op enable PR deploys` | GraphQL mutation (CLI doesn't expose this) |
| `/railway-op set up a webhook to Slack` | Playwright-driven — prompts you to sign in to the Railway dashboard |
| `/railway-op status` | Enumerate projects, services, environments, deployment states |
| `/railway-op add postgres with redis` | Add managed databases, wire connection variables |
| `/railway-op restore latest backup` | Identify backup, restore, verify |

Natural-language triggers also work: "deploy this to Railway", "my Railway service is down", "what's running on Railway", "check Railway logs".

## Escalation ladder

The agent always tries the simplest tool first and escalates only on a genuine capability gap — never on a usage error (a failed CLI attempt is retried once with corrected input before escalating):

| Level | When | Examples |
|---|---|---|
| **Railway CLI** | Always first | `railway up`, `railway variable set`, `railway logs`, `railway domain` |
| **GraphQL API** | CLI lacks the mutation, or a bulk mutation would trigger a redeploy storm | `tcpProxyCreate`, `projectUpdate` (rename, PR deploys), `serviceUpdate` (rename, icon), `metrics` |
| **Playwright** | Genuinely UI-only actions | Webhook setup, audit logs, billing, SSO, member management, template publishing |

Entering the Playwright rung requires being able to name the missing CLI subcommand and the missing GraphQL mutation — it's never a workaround for an auth failure or an unread doc.

## Built-in gotchas (18 known sharp edges)

The agent knows and routes around these automatically, and uses a symptom index to route a reported failure straight to the relevant gotcha. A few highlights:

| Gotcha | What happens if you don't know |
|---|---|
| `railway volume add` doesn't redeploy | Writes go to overlay filesystem, lost on restart |
| Postgres needs `PGDATA` subdirectory | `initdb` fails silently because `lost+found/` makes mount "non-empty" |
| Private DNS is IPv6-only (older envs) | Connection refused on apps that only bind IPv4 |
| `railway ssh` re-tokenizes args | Quoted SQL loses its boundaries at the remote shell |
| Cloudflare SSL "Flexible" + Railway | Infinite redirect loop (`ERR_TOO_MANY_REDIRECTS`) |
| `railway logs` without bounds | Streams forever, blocks script execution |

Full list of 18 gotchas, plus the symptom index, in the agent definition (`railway-operator/agents/railway-operator.md`).

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

Every mutation is verified by reading back the resource — a mutation without a read-back is not considered done:

| After | Verified by |
|---|---|
| Deploy | `service status --json` shows SUCCESS + clean log output |
| Variable set | `variable list --json` returns new value |
| Volume add | `ssh df -hT` shows ext4 (not overlay) |
| Domain add | `domain --json` + DNS records block |
| GraphQL mutation | Independent read-back query — never trust the mutation's own echo |
| Playwright action | Screenshot + snapshot |

Before any report is written, the agent runs a pre-report gate: every mutation has a captured read-back, no service that was healthy at task start ended up crashed, no unintended redeploys fired, every `TodoWrite` entry is resolved or listed as follow-up, and no secret values are echoed beyond what the user needs.

## Components

| Type | Name | Purpose |
|---|---|---|
| Skill | `railway-op` | User-invoked entry point; gathers Railway + project context, dispatches the agent |
| Agent | `railway-operator` | Executor with CLI + GraphQL + Playwright tools, 18 gotchas, verification loop, running on the session model |

## Tool access

The agent has broad tool access for the escalation ladder:

| Tool | Purpose |
|---|---|
| `Bash` | Railway CLI commands, GraphQL via curl, git operations |
| `Read`, `Grep`, `Glob` | Read project files, Railway config, optional local knowledge base |
| `Edit`, `Write` | Modify project config files (railway.toml, Dockerfile, etc.) when needed |
| `TodoWrite` | Track multi-step task progress |
| `WebSearch`, `WebFetch` | Check Railway docs, status page |
| Playwright MCP tools | Drive the Railway dashboard for UI-only actions |
| GoodMem MCP tools (optional) | Query/write Railway learnings |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Skill doesn't trigger on a Railway request | Plugin not installed, or Claude Code needs a restart after install | Run `claude plugin list` to confirm `railway-operator` is active; restart Claude Code |
| Agent stops immediately asking about auth | Railway CLI isn't authenticated | Run `railway login` yourself, or let the agent run `railway login --browserless` and paste the code it prints |
| Agent falls back to Playwright when you expected the CLI to work | The CLI genuinely doesn't expose that mutation (see the escalation ladder table), or the CLI attempt hit a usage error first | Check the agent's report for which rung it used and why; a usage error (bad flag, wrong service name) is retried automatically before any escalation |
| Playwright step fails with "not logged in" | The dashboard's browser session isn't authenticated | Sign in when prompted at the `railway.com/login` screenshot the agent shows you, then tell the agent you're ready |
| Task reported as "not done" | The agent's pre-report gate failed a check (e.g., a mutation without a verified read-back) | Read the report's stated failing evidence — it names the exact unverified step |
| `railway scale` command panics | Known CLI 4.36.0 bug (`UnauthorizedLogin`) | The agent uses `railway environment edit --service-config` instead; if you hit this directly, do the same |

## License

MIT. See [LICENSE](LICENSE).

## Credits

Built by [TheMizeGuy](https://github.com/TheMizeGuy). Backed by the [Claude Code](https://claude.com/claude-code) plugin system, running on the session model — always the strongest available Claude.
