# One-Click Rule Install (Extension Side)

Status: Proposal — **Rev 2**, reworked per maintainer review
([PR #3 review](https://github.com/kaovilai/redirector/pull/3#issuecomment-5181559627)).
The Rev 1 deep-link design (fragment payloads + origin allow-list + in-extension
install page) is replaced: the origin allow-list was not a real trust boundary,
because the payload lived in a fragment that any page could construct.

Scope: **this repo (the extension) only.** The community marketplace lives in a
separate repository — see [marketplace-repo.md](./marketplace-repo.md). The
extension's job is intentionally small: detect a trusted marketplace site (in
release builds: the one official site), accept an install request that names a
published entry by **slug**, fetch
the canonical reviewed artifact from that site, validate it, and store it. The
extension never becomes a marketplace, never renders marketplace content, and
never installs anything a web page hands it directly.

## Model

The Chrome Web Store is the model: the marketplace **rule detail page is the
confirmation surface**. It already shows everything an in-extension
confirmation screen would re-render — match pattern, redirect expression,
exclude, and build-time-computed test results — so there is **no in-extension
install page** and no new options-app route (the options app has no router
today, and this design does not add one). One click on the official page
installs; the page reflects the outcome.

## Goals

- One-click install of a published marketplace entry from its detail page on
  a **trusted marketplace origin** — in release builds, the official origin
  only.
- The install source is authenticated and the installed data is non-forgeable:
  the extension only ever installs what is actually published at the trusted
  origin the request came from, never what a page supplied.
- Takedowns take effect immediately for new installs.
- Works on Chrome, Edge, Firefox, and Safari.

## Non-Goals

- Installing from arbitrary or user-added origins. There is exactly one
  marketplace origin in release builds, on a domain/repo controlled by the
  extension maintainer. In code it is a **compile-time array of trusted
  origins** so forks and development builds can append their own testing
  origin (see Install Mechanism below) — but it is never runtime-extensible.
- An in-extension install/confirmation page, router, or marketplace UI.
- Payload-carrying deep links (`…/install#rule=<base64>`), origin allow-lists,
  or user-editable trusted-origin settings — all dropped from Rev 1.
- Auto-updating installed rules when the marketplace copy changes (reinstall is
  the manual update path, see Provenance below).
- Rule bundles in a payload format; a marketplace **entry** (slug) simply
  contains 1..N rules.

## Install Mechanism

### 1. Content script on trusted origins

A content script is registered for the trusted marketplace origins (in release
builds: the single official origin, e.g.
`https://<owner>.github.io/redirector-marketplace/*`). Because content scripts
only exist in documents genuinely served from those origins, their presence
*is* the source authentication — no allow-list logic needed.

The trusted deployments live in one source-level constant that everything
else derives from — each entry is a base URL (origin plus optional path
prefix, since GitHub Pages project sites serve under a path):

```ts
// src/lib/marketplace-origins.ts
export const TRUSTED_ORIGINS: readonly string[] = [
  'https://<owner>.github.io/redirector-marketplace',
  // Forks/dev builds append their testing deployment here, e.g.:
  // 'https://<fork-owner>.github.io/redirector-marketplace',
]
```

The manifest's `content_scripts` matches (`<base>/*`) and the background's
sender/fetch checks are all generated from this array. Upstream release
builds ship it with exactly the official deployment; a fork testing
marketplace changes appends its own Pages base URL and builds — a code change
and rebuild, deliberately **not** a runtime setting, so the installed
population's trust surface stays exactly one maintainer-controlled origin.

On load it marks the page as "extension installed" so the site can render its
Install buttons only when installing is actually possible (otherwise the site
shows store links — see the marketplace doc's fallback behavior). The marker
can be a DOM attribute plus a custom event, or a MAIN-world global; all four
browsers support MAIN-world injection (Safari logs a console warning), but a
DOM marker from the isolated world is sufficient and simplest:

```ts
document.documentElement.dataset.redirectorExtension = '1'
document.dispatchEvent(new CustomEvent('redirector:ready'))
```

### 2. Click handling — slug only, genuine gesture required

The page renders Install buttons carrying only a slug:

```html
<button data-redirector-install data-slug="old-reddit">Install</button>
```

The **content script binds the click listeners itself** (delegated listener on
the document) and requires `event.isTrusted`, so an install needs a genuine
user gesture — a synthetic `click()` dispatched by page script fails the check.
Even an XSS'd marketplace page cannot mass-install silently, and whatever it
forges can only ever name a published, reviewed entry. The page sends nothing
but the slug; it cannot inject rule data.

### 3. Background fetches the canonical artifact

The content script relays `{ type: 'install', slug }` to the background via
`runtime.sendMessage`. The background:

1. Validates the slug shape (`^[a-z0-9][a-z0-9-]{0,63}$`) — this is a URL path
   segment, so reject anything else before building the URL (no traversal).
2. Resolves the sender against `TRUSTED_ORIGINS`: `sender.origin` must equal
   the origin of an entry (defense in depth — only trusted-origin content
   scripts exist anyway), and that entry becomes the fetch base. Fetches
   `<base>/rules/<slug>.json` with `cache: 'no-cache'` (so takedowns and
   updates are seen promptly). The fetch base comes from the compile-time
   entry matched by the validated sender origin, never from message data, so
   each marketplace instance can only ever install what it itself publishes.
   The existing `<all_urls>` host permission already covers this fetch.
3. Validates the document (see Validation below). A 404 means the entry was
   taken down or never existed: takedown is immediately effective for installs.
4. Appends the entry's rules to storage via the existing
   `writeRulesToMode()` path, tagged with provenance (see below).
5. Replies with the outcome; the content script forwards it and the page flips
   the button ("Installed" — shown as "Reinstall" for already-installed slugs,
   since reinstalling is the manual update path — or an error state). The
   extension does not track entry revisions (see storage quota below); the
   detail page displays its own `meta.rev` / updated date, and the user decides
   whether to reinstall.

```ts
type MarketplaceRequest =
  | { type: 'query-installed'; slugs: string[] }
  | { type: 'install'; slug: string }

type InstallStatus =
  | 'installed'
  | 'already-installed'
  | 'quota-exceeded'
  | 'not-found'      // fetch 404 — taken down or unknown slug
  | 'invalid'        // schema/regex validation failed

type MarketplaceResponse =
  | { type: 'installed-state'; installed: string[] }
  | { type: 'install-result'; slug: string; status: InstallStatus }
```

`query-installed` lets the page render already-installed state for the slugs
it displays. It answers only for the specific slugs asked about, and only for
senders on a trusted origin.

Note the deliberate inversion of Rev 1's "no remote fetch" principle: that
principle's actual goal was "a page can't swap the payload after review".
Fetching the reviewed artifact from the canonical origin is the strong version
of that goal; inline payloads were the loophole.

## Canonical Entry Document

The fetched `rules/<slug>.json` is the marketplace's published artifact (full
schema in [marketplace-repo.md](./marketplace-repo.md)):

```json
{
  "v": 1,
  "meta": { "name": "Old Reddit", "slug": "old-reddit", "…": "…" },
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

- `v` — document schema version; unknown versions → `invalid` outcome (page
  shows "update the extension").
- `rules` — 1..N objects mapping onto the existing `MatchRule` in
  `src/lib/url.ts` (`from`, `to`, `exclude?`, `mode?`, `testUrl?`).
- **`enabled` is never honored from the outside.** The extension strips it if
  present; installed rules go through the existing `normalizeRules()`
  defaulting.
- `meta` is used for display on the site; the extension persists none of it
  except the slug (see storage quota below).

## Extension-Side Validation (silent, before the storage write)

The extension performs only machine checks — no warning UI, since the user
already saw the human-readable presentation (patterns, computed test results,
risk annotations, disclaimer) on the detail page, stamped at site build time:

- JSON parse + schema check: only known keys of expected types accepted,
  everything else dropped; bounded rule count and string lengths.
- Every `from`/`exclude` compiles (same `try/catch` approach as
  `isRegexMatch()`); failure → `invalid`.
- Duplicate detection: an identical rule set already installed under the same
  slug → `already-installed`; reinstall semantics below.

The anti-phishing signal suite from Rev 1 (confusables table, shortener
blocklist, sensitive-domain list) **moves to marketplace CI** where the lists
can be updated without an extension release — see the marketplace doc. There
is no `src/lib/phishing.ts` in this repo and this design no longer plans one.

## Provenance & Lifecycle (slug-keyed)

Install, uninstall, and reinstall are all keyed by slug:

- Each installed rule carries a small optional field:
  `source?: { slug: string }` — enough for a source badge in the rules list
  and a **"remove all rules from this entry"** action, without building a full
  group-management feature.
- **Reinstalling a slug replaces the rules that still carry that slug.** This
  doubles as the manual update path — no auto-update machinery. The detail
  page's button becomes "Reinstall" for installed slugs.
- **Editing an installed shared rule drops its provenance** (the `source`
  field is cleared): it becomes a plain local rule. Reinstalling the slug then
  no longer touches it, so user edits are never silently overwritten. (This
  settles Rev 1's open question in favor of predictability.)
- Uninstalling and reinstalling the extension does not restore marketplace
  rules; rules are user data, as today.

## storage.sync Quota (real, current constraint)

The extension stores **all rules under a single sync key** (`RULES_KEY` in
`src/lib/storage.ts`), and `chrome.storage.sync` caps a single item at 8 KB
(`QUOTA_BYTES_PER_ITEM`). Users already hit this — the options UI has a
quota-exceeded path suggesting Local mode. Typical rules serialize to ~150–300
bytes, so the whole budget is roughly 25–40 rules. Therefore:

- Synced provenance is **minimal**: `source: { slug }` only (~25–35 bytes per
  rule). Entry names, descriptions, authors, and source URLs are **not**
  stored in the synced rule objects — the detail page is always reachable from
  the slug, and any display cache (e.g. slug → entry name for the badge) lives
  in `storage.local`.
- An install that would exceed quota fails cleanly with `quota-exceeded`; the
  page surfaces it and the options UI's existing "switch to Local mode" advice
  applies.
- Sync chunking (splitting rules across keys) is a **separate later refactor**,
  not a dependency of this design.

## Security Analysis

- **Source authentication:** the content script only exists in documents
  really served from a `TRUSTED_ORIGINS` origin — a compile-time constant, so
  there is no runtime origin configuration to get wrong and no
  user-configurable trust surface. Release builds contain exactly the
  official origin; widening it means shipping a different build (which is the
  point — forks own their own trust decisions).
- **Non-forgeable payloads:** the extension installs only what it fetched from
  the sender's validated trusted origin. `meta`/provenance can no longer be
  forged by a link constructor (Rev 1's flaw), and one trusted origin cannot
  serve installs for another.
- **Genuine gesture:** `event.isTrusted` gating means installs require a real
  user click on the official page.
- **Immediate takedown:** removing `rules/<slug>.json` 404s all new installs.
- **Slug hygiene:** strict slug regex before URL construction; response
  size-capped and schema-validated; all strings treated as data, never HTML.
- **No silent state changes:** nothing is written without a user click, and
  `enabled` is never accepted from the outside.
- **Residual surface:** a third-party site can deep-link a user *to* a rule's
  detail page, but that can only lead to installing a published, reviewed rule
  and still requires the user's own click on the official page — the same
  exposure as any store listing link.

## Implementation Sketch

1. `src/lib/marketplace-origins.ts` — the `TRUSTED_ORIGINS` constant (official
   origin only in upstream; forks append their testing origin).
2. `src/entrypoints/marketplace.content.ts` — WXT content script matching the
   trusted origins: readiness marker, delegated `isTrusted` click handler for
   `[data-redirector-install]`, message relay, button-state updates.
3. `src/lib/install.ts` — pure, unit-testable functions: slug validation,
   entry-document schema validation, duplicate detection, slug-keyed
   merge/replace of the rules array (vitest, like `check.ts`).
4. `src/entrypoints/background.ts` — `runtime.onMessage` handler for the two
   request types: check sender origin, fetch canonical JSON from it, call
   `install.ts` logic, write via the existing storage path, reply with the
   outcome.
5. `src/entrypoints/options/` — rules list renders a small source badge for
   rules with `source.slug` (linking to the detail page) and a
   "remove all rules from this entry" action. No new route.
6. **Manifest:** one addition — a `content_scripts` entry generated from
   `TRUSTED_ORIGINS` (Rev 1's "no manifest changes required" no longer
   holds). No new permissions: `storage` and `<all_urls>` host access already
   exist.
7. Tests — unit tests for `install.ts`; a store-level test that installing
   appends slug-tagged normalized rules and reinstalling replaces them; a
   message-contract test for the background handler (including 404,
   quota-exceeded, and untrusted-sender outcomes).

## Browser Notes

- **Chrome / Edge / Firefox:** static `content_scripts` + `runtime.sendMessage`
  + background `fetch` — all standard MV3, no per-browser branches expected.
- **Safari:** no `webRequest` dependency in this flow (nothing to intercept),
  so the existing Safari differences don't apply. If a MAIN-world marker is
  used, Safari supports it but logs a warning; the DOM-attribute marker avoids
  even that.
- **Extension not installed:** the content script never runs, the readiness
  marker never appears, and the site shows store links instead of Install
  buttons (marketplace repo's responsibility). No interception fallback needed.
