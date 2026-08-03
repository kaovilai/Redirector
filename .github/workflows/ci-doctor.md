---
emoji: "🏥"
description: Investigates failed Release workflow runs and opens a diagnostic issue with the likely root cause.
on:
  workflow_run:
    workflows: ["Release"]
    types: [completed]
    branches: [main]
  workflow_dispatch:
permissions:
  actions: read          # read workflow runs, jobs, and logs
  contents: read         # read repository files
  issues: read           # search existing issues (writes go through safe-outputs)
  pull-requests: read    # required by the default GitHub toolset
  copilot-requests: write # use the Actions GITHUB_TOKEN for Copilot inference (no personal PAT)
engine: copilot
network: defaults
tools:
  github:
    mode: remote
    toolsets: [default, actions]
safe-outputs:
  create-issue:
    max: 1
    title-prefix: "[CI Doctor] "
    labels: [ci-failure, automation]
timeout-minutes: 15
---

# CI Doctor

The **Release** workflow just finished. If it **failed**, investigate and file a single, actionable diagnostic issue. If it succeeded (or was cancelled/skipped), do nothing.

## Steps

1. Confirm the triggering run's conclusion. Only continue when it is `failure`.
2. Use the GitHub `actions` tools to list the jobs for the failed run and read the logs of the failed job(s).
3. Identify the most likely root cause. Focus on the first real error, not downstream noise. Common areas for this WXT browser-extension project:
   - `pnpm install` / lockfile issues
   - `wxt build`, `pnpm zip`, or `pnpm zip:firefox` failures
   - store-publishing steps (Chrome, Edge, Firefox, Safari) failing due to missing/expired secrets
4. Create **one** issue summarizing:
   - A short, specific title describing the failure.
   - The failing job and step.
   - The key error lines (quoted, trimmed).
   - Your best hypothesis for the root cause and a concrete suggested fix.
   - A link to the failed run: `${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.event.workflow_run.id }}`.

Keep the issue concise and skimmable. Do not create duplicate issues for the same failure if one already exists.
