---
emoji: "🔍"
description: Reviews pull requests and posts a concise, actionable review comment.
on:
  pull_request:
    types: [opened, reopened, synchronize]
permissions:
  contents: read          # read the changed files
  pull-requests: read     # read PR metadata (the comment is posted via safe-outputs)
  issues: read            # required by the default GitHub toolset
  copilot-requests: write # use the Actions GITHUB_TOKEN for Copilot inference (no personal PAT)
engine: copilot
network: defaults
tools:
  github:
    mode: remote
    toolsets: [default]
safe-outputs:
  add-comment:
    max: 1
timeout-minutes: 15
---

# PR Reviewer

Review the pull request that triggered this workflow and post **one** concise review comment.

## Steps

1. Read the PR diff and the changed files. This is a WXT + Svelte + TypeScript browser extension (Redirector).
2. Evaluate the changes for:
   - Correctness and obvious bugs, especially in URL matching / redirect rule logic.
   - Cross-browser concerns (Chrome, Edge, Firefox, Safari via WXT).
   - TypeScript/Svelte type safety and adherence to existing patterns.
   - Missing or inadequate tests (`vitest` unit tests, `playwright` e2e tests).
3. Post a single comment with:
   - A one-line overall summary (e.g. looks good / needs changes).
   - A short bulleted list of the most important findings, each pointing at a specific file/line where possible.
   - Only high-signal feedback. Skip nits, formatting, and style unless they cause real problems.

If the change is trivial and looks correct, say so briefly rather than inventing issues.
