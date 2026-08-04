# Community Marketplace Repository (Rough Design)

Status: Proposal — **Rev 2**, reworked per maintainer review
([PR #3 review](https://github.com/kaovilai/redirector/pull/3#issuecomment-5181559627)).
Describes a **separate repository** (e.g. `redirector-marketplace`). Nothing in
this document is implemented in the extension repo; it exists so the
extension-side design in [one-click-rule-install.md](./one-click-rule-install.md)
has a concrete counterpart.

The marketplace site is hosted on a **single official origin under the
extension maintainer's control** — the origin ships hardcoded in the extension
to all users, so it cannot live under an arbitrary account. Hosting details are
settled when this moves forward.

## Purpose

A community-curated catalog of Redirector rules, published as a **static
GitHub Pages site**. Each published **entry** (identified by a slug, containing
1..N rules) has an Install button that the extension's content script wires up
when present. Contribution happens entirely through GitHub's built-in features
— no backend, no database, no auth code:

- **"Login with GitHub"** is simply GitHub itself: contributors submit via
  GitHub **Issue Forms**, which require being signed in.
- Moderation is automated by default with human review reserved for risk
  (see Moderation below).
- Hosting is GitHub Pages built by GitHub Actions.

## Entry Model

The unit of sharing is the **catalog entry**, not a payload:

- One entry = one slug = one file `rules/<slug>.json` containing 1..N rules.
  (This replaces Rev 1's payload bundles; curated multi-rule sets are just
  entries with several rules.)
- The published file is the **canonical artifact**: the extension fetches
  `rules/<slug>.json` from the official origin at install time and installs
  only that. Deleting the file (takedown) immediately 404s all new installs.
- Install / uninstall / reinstall are keyed by slug on the extension side;
  reinstalling replaces the entry's rules, which doubles as the manual update
  path.

### Entry file format (`rules/<slug>.json`)

```json
{
  "v": 1,
  "meta": {
    "name": "Old Reddit",
    "slug": "old-reddit",
    "description": "Always use old.reddit.com instead of the redesign",
    "author": "Alice (github: alice-gh)",
    "tags": ["reddit", "ui"],
    "createdFromIssue": 123,
    "rev": 2,
    "additionalTestUrls": ["https://www.reddit.com/r/aww/top"]
  },
  "rules": [
    {
      "from": "^https://www\\.reddit\\.com/(.*)",
      "to": "https://old.reddit.com/$1",
      "exclude": "",
      "mode": "regex",
      "testUrl": "https://www.reddit.com/r/cats"
    }
  ]
}
```

- `rules[*]` maps onto the extension's `MatchRule` (`from`, `to`, `exclude?`,
  `mode?`, `testUrl?`). `enabled` is never published and never honored.
- `meta.author` — display name the submitter provided in the issue form
  (falls back to their GitHub username); shown as a credit on the detail page.
  Display-only: the extension does not persist it (sync-quota reasons — see
  the extension doc).
- `meta.rev` — integer bumped by every accepted update; shown on the detail
  page ("Updated <date>, rev 2") so users can tell an entry changed since they
  installed it.
- `meta.additionalTestUrls` — extra URLs used by CI validation only; not part
  of the install contract.

## Contribution Flow — new entries (`submit-rule.yml`)

1. User clicks **Share a rule** on the site → lands on the repo's issue form
   (`…/issues/new?template=submit-rule.yml`). GitHub prompts for login.
2. The structured form collects: **Entry name**, **Your name / display name**
   (for `meta.author` credit), **Description**, and for each rule: **Match
   URL** (`from`), **Redirect To** (`to`), **Exclude** (optional), **Mode**
   (dropdown), **Test URLs** (at least one, one per line), plus
   **Category/tags** and checkbox acknowledgements (no malicious/tracking
   redirects, license agreement).
3. A GitHub Action parses the deterministic form body, runs the validation
   pipeline (below), comments the results on the issue — including the
   computed redirect for every test URL, so submitter and reviewers see exactly
   what the rule does — and on success opens a PR adding `rules/<slug>.json`.
4. The PR proceeds through the moderation pipeline (below); merge triggers the
   Pages build and the entry appears in the catalog.

Submitters never touch JSON, git, or the site generator.

## Contribution Flow — updates to existing entries (`update-rule.yml`)

Existing rules break (sites change their URL structure) or need improvement.
Requiring a hand-crafted PR would limit maintenance to developers, so updates
get their own non-developer path:

1. Every rule detail page has a **"Report a problem / suggest an update"**
   button linking to a second issue form, pre-filled with the entry's slug via
   the template query parameter
   (`…/issues/new?template=update-rule.yml&slug=<slug>`).
2. The form collects: **Slug** (pre-filled, validated against the catalog),
   **What's wrong / what should change** (textarea), the **proposed new field
   values** (same rule fields as the submission form — any field left blank
   means "keep current value"), at least one **URL demonstrating the problem
   or verifying the fix**, and a **request-removal checkbox** for takedown
   requests instead of edits.
3. CI resolves the slug, applies the proposed field changes over the current
   `rules/<slug>.json`, and validates the *result* through the exact same
   pipeline as a new submission. It additionally posts a **before/after
   diff** comment: old vs new field values and old vs new computed redirects
   for every test URL.
4. On success, CI opens a PR updating the file and bumping `meta.rev`. The
   original submitter is @-mentioned on the PR (best effort, from
   `meta.createdFromIssue`) so they can weigh in, but their approval is not a
   hard gate — original authors go inactive, and rules must stay fixable.
5. **Updates are never auto-merged.** A changed `to`/`from` on an
   already-published entry is a hijack vector (users reinstall what a slug
   currently points to), so every update PR routes to human review regardless
   of risk score. Removal requests likewise.

Developers can still PR `rules/*.json` directly; the same PR validation
applies either way.

## Validation Pipeline

Three layers, in order. Layer 1 is code the AI layer cannot override; this —
plus structured issue-form inputs — is the prompt-injection guard for
submission text.

### Layer 1 — deterministic hard gates (blocking, non-negotiable)

Plain CI (`validate-pr.yml` + `scripts/validate-rules.js`) fails the check
unless, for every rule in the changed entry:

- the JSON matches `schema/rule.schema.json` (only known keys, bounded rule
  count and string lengths, valid slug `^[a-z0-9][a-z0-9-]{0,63}$` matching
  the filename);
- `from` and `exclude` compile;
- every test URL matches `from` and produces a redirect different from the
  input URL;
- the slug is not a duplicate of another entry (new submissions).

Implementation note: v1 CI does **not** vendor extension code. The match logic
is reimplemented (or later published as a tiny shared package); wherever the
extension's `matchRule` does get reused, it depends on `urlpattern-polyfill`
(Node 20 has no native `URLPattern`).

### Layer 2 — risk signals (deterministic, route to humans)

The anti-phishing suite lives **here**, not in the extension: confusables
tables, shortener blocklists, and sensitive-domain lists are maintenance-heavy
and must be updatable without an extension release. `scripts/validate-rules.js`
emits `::error::`/`::warning::` annotations on the offending `rules/*.json`
lines:

| Signal | Severity | Effect |
|--------|----------|--------|
| Homoglyph / confusable characters in redirect-target host (Cyrillic/Greek lookalikes, digit-letter swaps, decoded punycode) | HIGH | human review required |
| Subdomain-of-target trick (`paypal.com.evil.example`) | HIGH | human review required |
| `from` matches a high-sensitivity (auth/financial) domain AND target is a different eTLD+1 | HIGH | human review required |
| Redirect target is a known URL shortener | MEDIUM | warning annotation |
| Excessive subdomain depth (> 4 labels) or hostname > 60 chars | MEDIUM | warning annotation |
| `from` is extremely broad (`.*`, `^http`) | MEDIUM | warning annotation |
| Target on a free-hosting service (`*.vercel.app`, …) | LOW | comment only |

Any HIGH signal labels the PR `needs-human-review` and disables auto-merge —
**never auto-approved**. It does not permanently block: a maintainer reviewing
and merging *is* the resolution (no "second maintainer forced-pass re-run"
flow; this project doesn't have that team).

The full presentation of these findings (per-signal explanations like "uses a
Cyrillic 'е' in place of 'e'") is stamped into the entry's **detail page at
build time**, where the user sees it before clicking Install — the extension
itself performs only silent schema/regex/duplicate validation.

Scope note: the list-backed signals (confusables table, shortener blocklist,
sensitive-domain list) are maintenance liabilities and can ship incrementally
— v1 can launch with the cheap structural checks (broad patterns,
subdomain-of-target, hostname shape) and grow the lists later. Because they
live in this repo's CI, expanding them never requires an extension release.

### Layer 3 — agentic review (gh-aw) for the default path

Not every rule should gate on a human. Everything that passes Layer 1 and
raises no HIGH signal in Layer 2 goes to an AI review step built with
[GitHub Agentic Workflows (gh-aw)](https://github.com/githubnext/gh-aw): a
markdown-defined workflow (`.github/workflows/risk-review.md`, compiled to a
`.lock.yml`) runs an agent that inspects the changed entry and must produce a
verdict:

- **approve** → the workflow approves and enables auto-merge; a bot merge
  publishes the entry.
- **flag** (anything suspicious the deterministic signals missed: misleading
  name/description vs. actual behavior, tracking-parameter injection,
  affiliate-ID insertion, semantic lookalikes) → label `needs-human-review`
  with a written reason; **the gh-aw check fails, blocking merge** until a
  maintainer resolves it.

Guardrails, per gh-aw's model and the maintainer's requirements:

- The agent runs with **read-only tools and no network**, and its only output
  channel is the verdict via `safe-outputs` (add-labels / PR review); it has no
  permission to push, merge, or edit files.
- It **cannot override Layer 1 or Layer 2**: branch protection requires those
  checks independently; the agent's approval is necessary only for the
  *auto*-merge path, and a HIGH signal disables that path before the agent
  runs.
- Submission text is treated as untrusted data (structured form fields, not
  instructions); the compiled workflow pins its prompt and tools so issue/PR
  bodies cannot reconfigure it.
- Rate limits: at most N open submissions per GitHub account (CI closes the
  excess with a polite comment) and a cooldown between submissions, so the
  agentic step can't be farmed.

### Backstop — post-hoc takedown

Merge-time review is not the last line: deleting `rules/<slug>.json` (revert
PR, or a removal request via the update form) rebuilds the site without the
entry **and** immediately 404s the extension's canonical fetch, so new installs
stop the moment the takedown merges. Already-installed copies are the user's
own data, as with manual entry — the extension does not phone home.

## Static Site (GitHub Pages)

Built by CI from `rules/*.json` — no client-side GitHub API calls, works
logged-out:

- **Index page** — searchable/filterable card list (name, description, tags,
  author) over a generated `rules.json` index.
- **Rule detail page** — the **confirmation surface** for installs (there is
  no in-extension install page): full field display for each rule in the
  entry, the test URLs *with their build-time-computed redirect results*,
  Layer 2 risk annotations if any, author credit, `rev`/updated date, and the
  always-visible disclaimer:

  > **Disclaimer:** Rules in this marketplace are community-supplied.
  > Installing a rule will cause the Redirector extension to redirect your
  > browsing traffic according to its match and redirect expressions. By
  > clicking Install you accept full responsibility for the effects of any
  > rule you install. Only install rules you have reviewed.

- **Install button** — rendered as `<button data-redirector-install
  data-slug="…">`. The extension's content script (see the extension doc)
  marks the page when present, binds the click handler itself, and flips the
  button to Installed / Reinstall based on the reported outcome. The page
  never constructs payloads and sends nothing but the slug.
- **Extension-not-installed fallback** — if the readiness marker
  (`data-redirector-extension` / `redirector:ready`) doesn't appear, the page
  swaps the Install button for store links plus a copy-to-clipboard JSON block
  compatible with the extension's existing **Import** feature. (Rev 1's
  separate `/install` fallback page is gone along with payload links.)
- **"Share a rule" / "Report a problem"** buttons — link to the two issue
  forms.

## Repository Layout

```
redirector-marketplace/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── submit-rule.yml        # new-entry form
│   │   └── update-rule.yml        # update/removal-request form (slug pre-filled)
│   ├── workflows/
│   │   ├── validate-submission.yml# issue → parse → validate → PR (create or update)
│   │   ├── validate-pr.yml        # Layer 1 hard gates + Layer 2 risk signals
│   │   ├── risk-review.md         # Layer 3 gh-aw agentic review (+ compiled .lock.yml)
│   │   └── deploy-pages.yml       # build & deploy static site on merge
│   └── CODEOWNERS                 # protects schema/, scripts/, .github/ (the pipeline itself)
├── rules/                         # one JSON file per entry (canonical artifacts)
├── schema/rule.schema.json
├── scripts/validate-rules.js      # hard gates + risk-signal analysis (Node.js)
├── site/                          # static site generator input + templates
└── README.md                      # contribution guide
```

`CODEOWNERS` covers the pipeline (`schema/`, `scripts/`, `.github/`, `site/`)
— those always need a maintainer. It deliberately does **not** cover `rules/`,
since the default path there is bot-merged after the pipeline passes.

## Coupling Contract with the Extension

The only interfaces the two repos share — keep them stable and versioned:

1. **Official origin** — hardcoded in the extension's `content_scripts` match
   and canonical-fetch base URL. Changing it requires an extension release.
2. **Canonical artifact URL & schema** — `GET <origin>/rules/<slug>.json`
   returning the `v`/`meta`/`rules` document above; bump `v` on breaking
   changes (the extension rejects unknown versions).
3. **DOM integration contract** — the readiness marker
   (`data-redirector-extension`, `redirector:ready`) and the Install button
   shape (`data-redirector-install`, `data-slug`), plus the button states the
   page must render for reported outcomes (installed / already-installed /
   not-found / invalid / quota-exceeded).

Everything else (site design, form wording, validation workflows, risk lists,
categories) evolves freely in the marketplace repo without an extension
release.
