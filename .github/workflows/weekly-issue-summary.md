---
emoji: "🗓️"
description: Posts a weekly digest of open issues as a single summary issue.
on:
  schedule:
    - cron: "weekly on monday"   # fuzzy weekly schedule (distributed run time)
  workflow_dispatch:
permissions:
  contents: read          # read repository files
  issues: read            # read open issues (the digest is created via safe-outputs)
  pull-requests: read     # required by the default GitHub toolset
  copilot-requests: write # use the Actions GITHUB_TOKEN for Copilot inference (no personal PAT)
engine: copilot
network: defaults
tools:
  github:
    mode: remote
    toolsets: [default]
safe-outputs:
  create-issue:
    max: 1
    title-prefix: "[Weekly Digest] "
    labels: [automation]
timeout-minutes: 15
---

# Weekly Issue Summary

Produce a single digest issue summarizing the state of open issues in this repository.

## Steps

1. List the currently open issues (exclude pull requests).
2. Group them into a short, useful digest, for example:
   - **Needs attention** — issues with no recent activity or no assignee.
   - **In progress** — assigned issues with recent activity.
   - **Recently opened** — issues opened in the last 7 days.
3. Create **one** issue titled with the current date containing:
   - The total number of open issues.
   - The grouped lists above, each item linking to the issue and showing its number and title.
   - A short highlight of anything that looks stale or high priority.

Keep it concise. If there are no open issues, create a brief issue noting that the backlog is clear.
