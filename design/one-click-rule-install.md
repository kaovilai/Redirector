# One-Click Rule Install (Extension Side)

Status: Proposal
Scope: **This repo (the extension) only.** The community marketplace lives in a
separate repository — see [marketplace-repo.md](./marketplace-repo.md) for its
design. The extension's job here is intentionally small: expose a URI handle
that accepts rule data from an "Install" button click on a static GitHub Pages
site, show the user what they are about to install, and add the rule if they
confirm. The extension never becomes a marketplace, never lists rules, and
never talks to GitHub.

## Goals

- A user browsing a community marketplace page (static GitHub Pages) can click
  **Install in Redirector** and have the rule appear in their extension after a
  single confirmation.
- Work on every browser the extension ships to (Chrome, Edge, Firefox, Safari)
  without new permissions beyond what the manifest already requests.
- The user always sees exactly what will be installed (match pattern, redirect
  expression, exclude pattern, test URLs) *before* anything is saved.
- Nothing is auto-enabled or auto-saved without explicit user confirmation.

## Non-Goals

- Hosting, indexing, searching, rating, or moderating rules (marketplace repo's
  job).
- Fetching rule payloads from remote URLs at install time (see Security).
- Auto-updating installed rules when the marketplace copy changes.

## The URI Handle

Browser extensions cannot reliably register custom protocol handlers
(`redirector://…`) across all four target browsers, but every browser lets a
web page link to an extension page via its `chrome-extension://` /
`moz-extension://` URL — and more portably, the extension itself can open its
options page for any URL it observes. The most portable, least-privileged
mechanism is:

**An install page inside the extension, addressed by a well-known path and
parameterized via the URL fragment:**

```
<extension-origin>/options.html#/install?rule=<base64url(JSON)>
```

Because extension origins differ per browser *and per installation* (Firefox
randomizes them), the marketplace page cannot hard-code them. So the handle is
completed by a second, tiny piece: a **forwarder URL** on a fixed public origin
that the extension intercepts with its existing redirect machinery.

### Flow

1. The marketplace's static page renders an **Install in Redirector** button.
   The button is a plain `<a>` link to a fixed, well-known URL such as:

   ```
   https://kaovilai.github.io/redirector-marketplace/install#rule=<base64url(JSON)>
   ```

2. The extension's background script registers a `webNavigation.onCommitted` /
   `onBeforeNavigate` listener (permissions already in the manifest:
   `webRequest`, `webNavigation`, `<all_urls>`) that watches for navigations to
   the well-known install path on the allow-listed marketplace origin(s).

3. When such a navigation is detected, the background script:
   - extracts the `rule` value from the intercepted URL's fragment
     (`new URL(details.url).hash` → parse `rule=<payload>`),
   - constructs a new extension URL with that value:

   ```
   browser.runtime.getURL('/options.html') + '#/install?rule=' + extractedPayload
   ```

   This reuses the exact interception pattern the extension already uses for
   rule-based redirects — no new permissions, no protocol registration, no
   native messaging.

4. The install page decodes the payload, validates it, and shows a
   **confirmation screen** (see UX below). Only when the user clicks **Add
   Rule** is the rule appended to storage via the existing
   `writeRulesToMode()` path.

5. If the extension is **not installed**, step 2 never happens and the browser
   lands on the real `…/install` page on GitHub Pages, which the marketplace
   repo serves as a graceful fallback: "Redirector isn't installed — get it
   here" with store links, plus a manual copy/paste of the rule JSON for the
   existing Import feature. This makes the same button work for everyone.

### Why the fragment (`#`) and not a query string

- Fragments are never sent to the server, so the GitHub Pages host (and its
  logs) never see the payload.
- The background script explicitly extracts the `rule` parameter from the
  intercepted fragment and reattaches it to the extension install URL; the
  fragment value does not automatically carry over across the navigation.
- No length-sensitive server behavior; payloads stay client-side end to end.

## Payload Format

The payload is the rule itself, inlined — the extension never fetches anything.
It is a base64url-encoded UTF-8 JSON document:

```json
{
  "v": 1,
  "rule": {
    "from": "^https://www\\.reddit\\.com/(.*)",
    "to": "https://old.reddit.com/$1",
    "exclude": "",
    "mode": "regex",
    "testUrl": "https://www.reddit.com/r/cats"
  },
  "meta": {
    "name": "Old Reddit",
    "description": "Always use old.reddit.com",
    "source": "https://github.com/<owner>/redirector-marketplace/blob/main/rules/old-reddit.json",
    "sourceLabel": "Redirector Marketplace"
  }
}
```

- `v` — payload schema version; unknown versions are rejected with a friendly
  "please update the extension" message.
- `rule` — maps 1:1 onto the existing `MatchRule` interface in
  `src/lib/url.ts` (`from`, `to`, `exclude?`, `mode?`, `testUrl?`). `enabled`
  is intentionally **not accepted** from the payload; the confirmation screen
  decides it (default: enabled, user can toggle).
- `meta` — provenance data stored alongside the rule in extension storage.
  - `source` — full URL linking back to the marketplace entry; displayed as a
    link on the confirmation screen and in the rules list.
  - `sourceLabel` — human-readable name of the marketplace or source that
    generated this link (e.g. `"Redirector Marketplace"`). The user may rename
    it on the confirmation screen or later in the extension settings; the
    renamed label is stored with the rule and shown as a small badge next to the
    rule in the rules list so the user always knows where each installed rule
    came from. If a trusted origin entry already has a user-assigned title (see
    Origin allow-list below) it pre-fills this field; otherwise the payload's
    `sourceLabel` is used as the default.
- Size limit: reject payloads over ~8 KB after decoding.
- Multiple rules: `rule` may instead be `rules: MatchRule[]` (bounded, e.g.
  max 20) so curated bundles work; the confirmation screen lists each one with
  its own checkbox.

## Confirmation UX (install page)

A new route/section in the existing options app (Svelte), reusing the
`RuleDialog` field layout:

1. **Header** — "Install rule from community marketplace" with the `meta.name`,
   a link to `meta.source`, and an editable **Source label** field pre-filled
   from the trusted-origin title (if the user has set one) or `meta.sourceLabel`
   from the payload. The user can rename the source here before installing; the
   label is stored with the rule and is not overwritten on subsequent installs
   from the same origin.
2. **Rule details, read-only but expandable** — Match URL, Redirect To,
   Exclude, Mode, exactly as `RuleDialog` renders them.
3. **Live test** — the payload's `testUrl` is run through the existing
   `matchRule()` immediately and the resulting redirect target is displayed
   (reusing `RuleCheckResult`). The user can edit the test URL to try their
   own. This is the "show, don't tell" trust builder: the user sees what the
   regex actually does before installing.
4. **Warnings** (computed locally):
   - invalid regex / URLPattern → install blocked with error;
   - duplicate of an existing rule (`from`+`to` match) → "already installed";
   - very broad match pattern (e.g. matches `<all_urls>`-ish patterns like
     `.*` or `^http`) → yellow caution banner;
   - redirect target host differs wildly from match host → informational note;
   - **phishing risk detected** (see Anti-Phishing Analysis below) → prominent
     red warning banner with a per-signal explanation. The user must explicitly
     check a box ("I understand the risks") before the **Add Rule** button
     becomes active.
5. **Actions** — **Add Rule** (appends via existing storage write, respecting
   the current sync/local storage mode) and **Cancel** (closes/navigates to
   the rules list). A toggle chooses enabled/disabled on install
   (default enabled).
6. After install, land on the normal rules list with the new rule highlighted
   and a success toast. The rule row shows a small **source badge** (the stored
   `sourceLabel`) next to the rule name so the user can always see where each
   marketplace-installed rule came from. Clicking the badge opens `meta.source`
   (the original marketplace entry URL).

## Security Considerations

- **No remote fetch.** The rule travels inside the link. There is no "install
  from URL" indirection, so a marketplace page can't swap the payload after
  review, and the extension needs no new host access at install time.
- **Origin allow-list.** The background interception only triggers on the
  configured marketplace origin(s) + exact install path. A built-in list of
  well-known marketplace origins ships with the extension; adding a new origin
  requires no extension release — users can opt in to additional trusted
  sources via an **extension settings page** (a user-editable allow-list stored
  in `sync` storage). Each user-added origin is shown a one-time confirmation
  dialog ("Allow rules from `https://example.github.io/my-marketplace/`?") before
  it is saved, so no origin is silently trusted. The settings page also lets
  the user assign a **human-readable title** to each trusted origin (e.g.
  `"My Team's Marketplace"`); this title is used as the pre-filled source label
  on the install confirmation screen and stored as the `sourceLabel` on every
  rule installed from that origin, so the provenance badge in the rules list
  reflects the user's own naming. The built-in list can only be expanded through
  a normal extension release — that friction stays for *default* trust; the
  opt-in path keeps the user in control without requiring a release.
- **Validation before render.** Payload is JSON-parsed inside `try/catch`,
  schema-checked (only known keys of the expected types accepted, everything
  else dropped), and all displayed strings are rendered as text (Svelte's
  default escaping) — never as HTML.
- **No silent state changes.** Nothing is written to storage until the user
  clicks Add. Closing the tab installs nothing.
- **Regex safety.** The same constraints as manual entry apply (rules are
  user-supplied regexes already); the confirmation preview compiles the regex
  in `try/catch`, mirroring `isRegexMatch()`.
- **`enabled` not honored from payload** so a malicious link can't sneak in a
  pre-armed rule without the user seeing the toggle.

## Anti-Phishing Analysis

Redirect rules that steer users toward lookalike or impostor domains are among
the most dangerous payloads this install flow can carry. The confirmation screen
runs a local phishing-risk analysis on the decoded rule *before* displaying
it. Signals are combined into a risk score; any HIGH-severity signal (or two or
more MEDIUM signals) triggers the red warning banner described in the
Confirmation UX section above.

### Signals checked

**1. Homoglyph / confusable character substitution in the redirect target host**

Attackers replace letters with visually identical Unicode characters or
ASCII lookalikes. The extension compares the `to` host against the `from` host
and a built-in list of high-value domains using confusable-character
normalization (Unicode Confusables dataset, subset covering the most common
spoofs):

| Substitution class | Examples |
|--------------------|----------|
| Cyrillic / Latin mixed | `а` (U+0430) → `a`, `е` (U+0435) → `e`, `о` (U+043E) → `o`, `р` (U+0440) → `r`, `с` (U+0441) → `c`, `х` (U+0445) → `x` |
| Greek | `α` → `a`, `β` → `b`, `ο` → `o`, `ρ` → `r`, `υ` → `u` |
| ASCII digit/letter swaps | `0` → `o`, `1` → `l` / `i`, `3` → `e`, `5` → `s`, `6` → `g`, `8` → `b` |
| Homoglyph punctuation | `‐` (U+2010) → `-`, `．` (U+FF0E) → `.` |
| IDN encoding | punycode (`xn--`) decoded before comparison |

If after normalization the redirect-target host differs from the matched host
only by these substitutions (or matches a sensitive domain via this path),
severity = **HIGH** and the UI shows:
> "This rule redirects to a domain that looks like **\<original host\>** but
> differs by character substitutions commonly used in phishing (e.g.
> `rеddit.com` uses a Cyrillic 'е'). Check the address bar carefully."

**2. Redirect-target host is a known URL shortener or redirector service**

A built-in blocklist of popular shortener domains (`bit.ly`, `t.co`, `tinyurl.com`,
`ow.ly`, `is.gd`, `buff.ly`, `rebrandly.com`, etc.). Severity = **MEDIUM**
(shorteners obscure the final destination).

**3. Excessive subdomain depth or very long hostname**

Hosts with more than 4 labels or a total hostname length over 60 characters
are a common phishing indicator. Severity = **MEDIUM**.

**4. Redirect target is on a free-subdomain hosting service**

Built-in list of services frequently abused for phishing: `*.github.io` (except
the allow-listed marketplace origin), `*.vercel.app`, `*.netlify.app`,
`*.pages.dev`, `*.glitch.me`, `*.repl.co`, `*.web.app`, etc. Severity = **LOW**
(legitimate rules also use these, so it is informational only unless combined
with other signals).

**5. The `from` pattern matches authentication or financial domains**

If the regex in `from` matches URLs on a list of high-sensitivity domains
(banks, OAuth providers, e-mail providers, government sites), the rule is
subject to heightened scrutiny regardless of where `to` points. Severity =
**MEDIUM** (could be legitimate — e.g. "always use HTTPS" rules — but warrants
extra review).

**6. Subdomain-of-target trick**

The redirect target host ends with the matched host but prepends a deceptive
label (e.g. matched `paypal.com`, target `paypal.com.evil.example`). Severity
= **HIGH**.

### Warning UI

When one or more signals fire, the confirmation screen shows:

```
⛔ Phishing risk detected — review carefully before installing

This rule may redirect you to a deceptive site. Specific concerns:
  • The redirect target "rеddit.com" resembles "reddit.com" but uses
    a Cyrillic character in place of the letter 'e'.
  • The target host has more than 4 subdomain levels.

Only install rules from sources you trust. [Learn more ↗]

[ ] I have reviewed the above warnings and still want to install this rule.
                                               [Cancel]  [Add Rule — disabled until checked]
```

The **Add Rule** button remains disabled until the acknowledgement checkbox is
checked; it never disappears, so users are never blocked from installing a rule
they genuinely want.

### Implementation note

All analysis runs **synchronously, locally** in `src/lib/install.ts` — no
network requests, no external API for phishing classification. The confusable
map and blocklists are bundled as static JSON at build time (small: the pruned
confusables table relevant to domain names is ~15 KB). This keeps install time
instant and the extension offline-capable.

## Implementation Sketch (extension repo)

Small, contained changes:

1. `src/lib/install.ts` — payload decode + schema validation + duplicate/
   broad-pattern checks + phishing-risk analysis (pure functions, unit-testable
   with vitest like `check.ts`; includes bundled confusables map and blocklists).
2. `src/lib/phishing.ts` — standalone module: confusable normalization,
   shortener blocklist, high-sensitivity domain list, risk-score aggregation,
   and human-readable signal descriptions returned to the UI.
2. `src/entrypoints/background.ts` — add the marketplace-origin navigation
   listener that rewrites the tab to the internal install page (a few lines,
   same pattern as existing redirect handling).
3. `src/entrypoints/options/` — an `#/install` route/state in `App.svelte`
   showing the confirmation screen; reuse `RuleDialog` fields,
   `RuleCheckResult`, `matchRule`, and the `rules` store's add path. The
   source-label field and the rules-list source badge are also rendered here.
4. `src/lib/sources.ts` — manages the user-editable trusted-origin allow-list
   (stored in `sync` storage); each entry has `origin`, user-assigned `title`,
   and `addedAt`. The settings UI reads/writes this store; the install page
   uses it to pre-fill the source label.
5. Tests — decode/validate unit tests; a store-level test that confirming
   appends exactly one normalized rule with the correct `sourceLabel`; a
   background test for the origin/path gate; a sources-store test covering
   add/rename/remove.

No manifest changes required: `webNavigation` + `<all_urls>` are already
requested, and options pages already exist.

## Browser Notes

- **Chrome / Edge / Firefox:** the webNavigation interception path works
  as-is.
- **Safari:** `webRequest` blocking is unavailable (already handled in
  `wxt.config.ts` via `tabs`), but the existing tab-update redirect fallback
  the extension uses for Safari covers the install URL the same way it covers
  rule redirects.
- **Extension not installed:** fallback page on GitHub Pages (marketplace
  repo's responsibility) shows install links + manual JSON for Import.
