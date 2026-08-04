# Community Marketplace Repository (Rough Design)

Status: Proposal — **Rev 2**, reworked per maintainer review
([PR #3 review](https://github.com/kaovilai/redirector/pull/3#issuecomment-5181559627)).
Describes a **separate repository** (e.g. `redirector-marketplace`). Nothing in
this document is implemented in the extension repo; it exists so the
extension-side design in [one-click-rule-install.md](./one-click-rule-install.md)
has a concrete counterpart.

The marketplace site is hosted on a **single official origin under the
extension maintainer's control** — release builds of the extension trust
exactly that origin, so it cannot live under an arbitrary account. (In the
extension source the trusted origin is a compile-time *array* so forks and
development builds can append a testing origin — e.g. a fork of this repo
publishing to its own Pages origin to test marketplace fixes — but upstream
ships only the official one; see the extension doc's Install Mechanism.)
Hosting details are settled when this moves forward.

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

## Contribution Flow — example-based suggestions (`suggest-rule.yml`)

Writing a working match pattern is the hardest part of submitting, so a third
form serves users who can only describe what they want **by example**:

1. **Suggest a rule** on the site links to
   `…/issues/new?template=suggest-rule.yml`.
2. The form collects: **Entry name**, **Your name / display name**,
   **Description**, and 1..N **example URL pairs** — "when I'm on `<URL>`,
   take me to `<URL>`" — plus the same acknowledgement checkboxes as
   `submit-rule.yml`. No regex, no mode, no field syntax.
3. A gh-aw **rule-generation workflow** (`generate-rule.md`, same framework
   and guardrails as Layer 3: pinned prompt and tools, no push/merge/edit
   permissions, `safe-outputs` only, form text treated as untrusted data)
   derives `from`/`to`/`exclude`/`mode` from the example pairs, generalizing
   conservatively — match only the demonstrated URL shape rather than broad
   patterns. If it cannot produce fields whose computed redirects reproduce
   every example pair, it labels the issue `needs-human-fix` with an
   explanation instead of opening a PR.
4. The example pairs become the entry's **test URLs** (first pair →
   `testUrl`, the rest → `meta.additionalTestUrls`), so the generated PR
   always includes test URLs, and CI's issue comment shows each example
   source URL's *resolved* redirect next to the redirect the suggester asked
   for — lookable proof the rule does what they intended.
5. From there the PR is a normal new-entry submission — same Layer 1 → 2 → 3
   pipeline — plus, because the suggester never typed the field values
   themselves, the same **`/approve` self-ack** gate before auto-merge as
   describe-only fixes (update flow, step 6).

## Contribution Flow — updates to existing entries (`update-rule.yml`)

Existing rules break (sites change their URL structure) or need improvement.
Requiring a hand-crafted PR would limit maintenance to developers, so updates
get their own non-developer path — one deliberately easy enough that reporters
who only know *that* something is broken (not how to fix it) can still use it:

1. Every rule detail page has a **"Report a problem / suggest an update"**
   button linking to its own issue form, pre-filled with the entry's slug via
   the template query parameter
   (`…/issues/new?template=update-rule.yml&slug=<slug>`).
2. Only two fields are required: **Slug** (pre-filled, validated against the
   catalog) and **What's wrong / what should change** (textarea). Everything
   else is optional: the **proposed new field values** (same rule fields as
   the submission form — any field left blank means "keep current value"), a
   **URL demonstrating the problem or verifying the fix**, and a
   **request-removal checkbox** for takedown requests instead of edits.
3. The proposed fix comes from one of two places, depending on whether the
   reporter filled any fix fields:
   - **Fix fields supplied** → CI applies them over the current
     `rules/<slug>.json` directly.
   - **All fix fields left blank (describe-only report)** → the gh-aw
     **rule-generation workflow** (`generate-rule.md`, shared with the
     suggestion form above) reads the problem description and the current
     entry, checks the demonstration URL and existing test URLs, works out
     corrected field values, and emits them via `safe-outputs` into the same
     CI machinery — the agent cannot push, merge, or edit files itself, and
     the report text is untrusted data against a pinned prompt (a description
     cannot reconfigure the agent). If it cannot produce a fix that passes
     validation, it labels the issue `needs-human-fix` with a comment
     explaining what it tried, instead of opening a PR.
   Fixes from either source prefer **amending over replacing**: when a site
   grows a new URL shape, keep the existing rules intact and add a rule (or
   extend `exclude` / add test URLs) beside them, so URLs the old rules
   still handle correctly keep working — a utility amendment cannot regress
   installed behavior. Existing rules are rewritten or removed only for
   structural breakage, where the old behavior is itself broken; that is a
   breaking change and is gated accordingly (step 5).
4. Either way, CI validates the *resulting* entry through the exact same
   pipeline as a new submission. **Generated PRs always include test URLs**:
   the entry's existing test URLs are kept (updated by the fix if the site's
   structure changed) and the report's demonstration URL, when given, is
   added as one. CI posts a **before/after diff** comment @-mentioning the
   reporter: old vs new field values and, for every test URL, the resolved
   redirect it now produces — so the reporter can look at the resolved URLs
   and see they are as intended. On success it opens a PR updating the file
   and bumping `meta.rev`. The original submitter is @-mentioned too (best
   effort, from `meta.createdFromIssue`) so they can weigh in, but their
   approval is not a hard gate — original authors go inactive, and rules
   must stay fixable.
5. CI classifies every update PR deterministically by replaying the entry's
   pre-existing test URLs against the new rules:
   - **Amendment** — purely additive (rules added, `exclude` extended, test
     URLs added): every pre-existing test URL still resolves to the same
     redirect. Amendments go through the **same auto agentic-review merge
     pipeline as new submissions** (Layer 1 → Layer 2 → Layer 3, bot merge
     on the default path); gh-aw-generated fixes get no shortcut for being
     machine-made — the Layer 3 reviewer is a separate pinned-prompt agent
     run that judges the diff on its own.
   - **Breaking change** — an existing rule is rewritten or removed, or any
     pre-existing test URL's resolved redirect changes. Breaking changes to
     already-published rules **always require human review**
     (`needs-human-review`, auto-merge disabled), and the entry's original
     submitter is **tagged to review them as well** — the author of the PR
     that introduced the entry or, for generated PRs, the author of the
     originating issue (`meta.createdFromIssue`); their response is still
     not a hard gate, per step 4. Repointing a published slug is a hijack
     vector (users reinstall whatever a slug currently points to), and a
     describe-only report is exactly how one would try to launder a repoint
     through the automated path — a redirect target moving to a different
     eTLD+1 additionally remains its own HIGH signal in Layer 2. Removal
     requests always route to human review.
6. **Reporter self-ack (`/approve`):** for describe-only fixes the reporter
   never typed the field values themselves, so auto-merge additionally waits
   for them to comment `/approve` on the PR — their acknowledgement, after
   checking the resolved test URLs in the diff comment, that this approach
   works for them. CI counts the command only from the reporting account,
   and it is an *extra* condition on top of Layers 1–3, never a bypass (a
   HIGH signal still disables auto-merge regardless of `/approve`). If the
   reporter goes silent for 14 days, the PR is labeled `needs-human-review`
   instead of merging unacknowledged.

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
| Update repoints the redirect target to a different eTLD+1 (update PRs only) | HIGH | human review required |
| Breaking update: existing rule rewritten/removed, or a pre-existing test URL's resolved redirect changes (update PRs only) | HIGH | human review required + original submitter tagged |
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
  publishes the entry. (PRs whose field values were generated by gh-aw —
  suggestions and describe-only fixes — additionally wait for the
  submitter's `/approve` self-ack before the bot merges.)
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
- **"Share a rule" / "Suggest a rule" / "Report a problem"** buttons — link
  to the three issue forms.

## Repository Layout

```
redirector-marketplace/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── submit-rule.yml        # new-entry form (fields supplied by submitter)
│   │   ├── suggest-rule.yml       # example-based suggestion form (gh-aw drafts the fields)
│   │   └── update-rule.yml        # update/removal-request form (slug pre-filled)
│   ├── workflows/
│   │   ├── validate-submission.yml# issue → parse → validate → PR (create or update)
│   │   ├── generate-rule.md       # gh-aw rule generation: suggestions + describe-only fixes (+ compiled .lock.yml)
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

1. **Official origin** — first entry of the extension's compile-time
   `TRUSTED_ORIGINS` array, from which its `content_scripts` matches and
   canonical-fetch base URLs derive. Changing it requires an extension
   release; forks testing marketplace changes append their own origin in
   their own builds.
2. **Canonical artifact URL & schema** — `GET <base>/rules/<slug>.json`
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
