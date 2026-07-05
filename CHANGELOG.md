# Changelog

All notable changes to the `railway-operator` plugin are documented here.

## 0.1.1

- **Model policy**: removed the pinned `model: opus` from the agent frontmatter. The agent now inherits the session model — always the strongest available Claude — instead of a specific hardcoded model. All "Opus 4.6" references in prose (plugin.json, marketplace.json, README, agent body) were replaced with session-model language.
- **Escalation-ladder decision tree**: added explicit rung-selection logic to Step 5 of the agent — when to use the CLI, when a failure is operator error vs. a genuine capability gap, and when GraphQL or Playwright is the correct next rung.
- **Worked examples**: added a full mutate-then-verify walkthrough for both the CLI rung (attaching a volume) and the GraphQL rung (renaming a service), showing what "verified" actually requires.
- **Pre-report gate**: the agent now runs a 7-point checklist before writing its final report (every mutation read back, no service newly broken, no unintended redeploys, all `TodoWrite` items resolved or logged as follow-up, every destructive action pre-authorized, Playwright tabs closed, no secrets echoed).
- **Symptom index**: added a table routing common reported failures (502s, data loss, `ECONNREFUSED`, etc.) directly to the matching gotcha, to be checked first before forming a new hypothesis.
- **18th gotcha documented**: `railway volume add` / `variable set` / config patches triggering unwanted redeploys — use `--skip-deploys` for bulk updates, then one explicit `railway redeploy`.
- **Skill dispatch**: `railway-op` now dispatches the `railway-operator:railway-operator` agent type directly (previously routed through `general-purpose` as a workaround); the agent inherits the session model rather than a pinned `model: "opus"` in the dispatch call.
- **Docs**: README rewritten with a quickstart, a worked walkthrough (custom-domain example with full report shape), and a troubleshooting table. `plugin.json` and `marketplace.json` author/owner set to `TheMizeGuy` (email `ben@meipath.com`).

## 0.1.0

- Initial release: `railway-op` skill + `railway-operator` agent. CLI-first execution with GraphQL and Playwright fallbacks, 17 built-in Railway gotchas, verification after every mutation, destructive-action guardrails.
