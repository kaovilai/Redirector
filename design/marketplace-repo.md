# Community Marketplace Repository (Rough Design)

Status: Proposal — describes a **separate repository** (e.g.
`redirector-marketplace`). Nothing in this document is implemented in the
extension repo; it exists so the extension-side design in
[one-click-rule-install.md](./one-click-rule-install.md) has a concrete
counterpart. The extension repo's only coupling to this repo is the fixed
install URL and the payload format.

## Purpose

A community-curated catalog of Redirector rules, published as a **static
GitHub Pages site**, where each rule has an **Install in Redirector** button
that deep-links into the extension. Contribution happens entirely through
GitHub's built-in features — no backend, no database, no auth code:

- **"Login with GitHub"** is simply GitHub itself: contributors submit via a
  GitHub **Issue Form**, which requires being signed in to GitHub. No OAuth
  app needed.
- Moderation is a maintainer merging a PR.
- Hosting is GitHub Pages built by GitHub Actions.

## Contribution Flow (user-friendly path)

1. User clicks **Share a rule** on the Pages site → lands on the repo's
   pre-selected issue form (`https://github.com/<owner>/redirector-marketplace/issues/new?template=submit-rule.yml`).
   GitHub prompts for login if needed.
2. User fills a structured **issue form** with the same fields the extension
   has today:
   - **Rule name** (required, short text)
   - **Description** (optional, textarea)
   - **Match URL** — the regex/URLPattern to match (`from`) (required)
   - **Redirect To** — the replacement expression (`to`) (required)
   - **Exclude Pattern** (`exclude`) (optional)
   - **Mode** (dropdown: `auto`, `regex`, `url-pattern`)
   - **Test URLs** (`testUrl`) — one or more URLs, textarea, one per line
     (required: at least one, so submissions are verifiable)
   - **Category / tags** (dropdown or free text, for browsing)
   - Checkbox acknowledgements (no malicious/tracking redirects, license
     agreement).
3. A GitHub Action triggers on the new issue:
   - parses the issue form body (deterministic Markdown structure),
   - validates: regex compiles, `to` templates are well-formed, every test
     URL actually matches `from` and produces the expected style of output,
     no duplicate of an existing rule,
   - runs the rule through the same match logic as the extension (the
     extension's `matchRule` logic can be published as a tiny npm package or
     vendored),
   - comments on the issue with the validation result — including the
     computed redirect for each test URL, so the submitter and reviewers see
     exactly what the rule does,
   - on success, opens (or updates) a **PR** that adds
     `rules/<slug>.json` generated from the form.
4. A maintainer reviews the PR (human moderation gate) and merges.
5. On merge to `main`, the **Pages build** Action regenerates the static site
   and the rule appears in the marketplace with its Install button.

Submitters never touch JSON, git, or the site generator — they fill a form.

## Repository Layout

```
redirector-marketplace/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   └── submit-rule.yml        # issue form (fields above)
│   └── workflows/
│       ├── validate-submission.yml# issue → validate → PR
│       ├── validate-pr.yml        # re-validate rule JSON on PRs
│       └── deploy-pages.yml       # build & deploy static site on merge
├── rules/
│   ├── old-reddit.json            # one file per rule (source of truth)
│   ├── google-to-ddg.json
│   └── …
├── schema/
│   └── rule.schema.json           # JSON Schema for rule files
├── site/                          # static site generator input
│   ├── index build script (any SSG or a plain build script)
│   └── templates/ (listing page, rule detail page, install fallback page)
└── README.md                      # contribution guide
```

### Rule file format (`rules/<slug>.json`)

Mirrors the extension's `MatchRule` plus display metadata; this is also
exactly what gets base64url-encoded into the install link:

```json
{
  "v": 1,
  "meta": {
    "name": "Old Reddit",
    "slug": "old-reddit",
    "description": "Always use old.reddit.com instead of the redesign",
    "author": "github-username",
    "tags": ["reddit", "ui"],
    "createdFromIssue": 123
  },
  "rule": {
    "from": "^https://www\\.reddit\\.com/(.*)",
    "to": "https://old.reddit.com/$1",
    "exclude": "",
    "mode": "regex",
    "testUrl": "https://www.reddit.com/r/cats"
  },
  "testUrls": [
    "https://www.reddit.com/r/cats",
    "https://www.reddit.com/r/programming/comments/abc"
  ]
}
```

## Static Site (GitHub Pages)

Built by CI from `rules/*.json` — no client-side GitHub API calls, no rate
limits, works logged-out:

- **Index page** — searchable/filterable card list (name, description, tags,
  author). Search can be pure client-side JS over a generated `rules.json`
  index file.
- **Rule detail page** — full rule display: match pattern, redirect
  expression, exclude, and the test URLs *with their computed redirect
  results* (computed at build time, so visitors see proof the rule works).
- **Install button** — a plain link, generated at build time:

  ```html
  <a href="https://<owner>.github.io/redirector-marketplace/install#rule=<base64url(payload)>"
     class="install-button">Install in Redirector</a>
  ```

  If the extension is installed, its background script intercepts this
  navigation and opens the in-extension confirmation screen (see the
  extension-side design). The page itself does nothing special.
- **`/install` fallback page** — a real static page at that path. Visitors
  *without* the extension land here; it reads the fragment client-side,
  pretty-prints the rule, links to the extension's store listings, and offers
  a copy-to-clipboard JSON block compatible with the extension's existing
  **Import** feature. One URL, two graceful outcomes.
- **"Share a rule" button** — links to the issue form.

## Moderation & Safety

- **Two human-visible gates:** CI validation comment on the issue, then
  maintainer PR review before anything is published.
- CI flags high-risk submissions for extra scrutiny: overly broad `from`
  patterns, redirects to URL shorteners/unknown hosts, lookalike domains,
  patterns touching auth/banking domains.
- Rules are plain data (regex + template string) — the marketplace never
  distributes code, and the extension never executes anything from it.
- Takedown = revert PR; the site rebuilds without the rule. (Already-installed
  copies are the user's own, as with manual entry — the extension does not
  phone home.)
- `CODEOWNERS` on `rules/` so merges always require a maintainer.

## Coupling Contract with the Extension

The only interfaces the two repos share — keep them stable and versioned:

1. **Install URL shape:** `https://<owner>.github.io/redirector-marketplace/install#rule=<base64url(JSON)>`
   (origin + path are allow-listed in the extension).
2. **Payload schema:** the `v`/`rule`/`meta` document defined in
   [one-click-rule-install.md](./one-click-rule-install.md); bump `v` on
   breaking changes, and the extension rejects unknown versions gracefully.

Everything else (site design, issue form wording, validation workflows,
categories) can evolve freely in the marketplace repo without an extension
release.
