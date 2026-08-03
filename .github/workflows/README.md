# Agentic workflows (gh-aw)

This directory contains [GitHub Agentic Workflows](https://github.github.com/gh-aw/)
authored as Markdown (`*.md`) and compiled into runnable GitHub Actions
(`*.lock.yml`) with the [`gh aw`](https://github.com/github/gh-aw) CLI extension.

All of these workflows are designed to run **without a personal access token (PAT)**
— they only use the built-in workflow token (`GITHUB_TOKEN`).

## Workflows

| Source | Purpose | Trigger |
| ------ | ------- | ------- |
| `ci-doctor.md` | Investigates failed **Release** runs and opens a diagnostic issue. | `workflow_run` (Release completed on `main`), `workflow_dispatch` |
| `pr-reviewer.md` | Reviews pull requests and posts one concise review comment. | `pull_request` |
| `weekly-issue-summary.md` | Posts a weekly digest issue of open issues. | weekly schedule, `workflow_dispatch` |

`agentics-maintenance.yml` is generated automatically by `gh aw compile` and also
runs on the workflow `GITHUB_TOKEN` only.

## No personal PAT required

Each workflow uses the Copilot engine and declares the `copilot-requests: write`
permission:

```yaml
engine: copilot
permissions:
  copilot-requests: write   # use the Actions GITHUB_TOKEN for Copilot inference
```

With this permission, `gh aw compile` wires Copilot inference to
`COPILOT_GITHUB_TOKEN: ${{ github.token }}` in the compiled `*.lock.yml` files,
so **no `COPILOT_GITHUB_TOKEN` (or any other PAT) secret needs to be created**.
All write operations go through gh-aw `safe-outputs`, which also use the workflow
`GITHUB_TOKEN`.

> Note: token-based Copilot inference requires that the repository's organization
> has centralized Copilot billing enabled. See the
> [gh-aw billing docs](https://github.github.com/gh-aw/reference/billing/).

## Compiling

Install the extension (once):

```sh
gh extension install github/gh-aw
```

Recompile every workflow in this directory after editing any `*.md` file:

```sh
gh aw compile
```

Compile a single workflow:

```sh
gh aw compile ci-doctor
```

The committed `*.lock.yml` files were generated pinned to gh-aw `v0.84.3`:

```sh
gh aw compile --action-mode release --action-tag 53258938b59e0797fefeed05ec0c681514b2a827
```

Always commit the regenerated `*.lock.yml` files together with the `*.md` changes.
