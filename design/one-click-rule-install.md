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

3. When such a navigation is detected, the background script redirects the tab
   to the extension's own install page, passing the fragment through:

   ```
   browser.runtime.getURL('/options.html') + '#/install?rule=' + payload
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
- Fragments survive the redirect into the extension page unchanged.
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
    "source": "https://github.com/<owner>/redirector-marketplace/blob/main/rules/old-reddit.json"
  }
}
```

- `v` — payload schema version; unknown versions are rejected with a friendly
  "please update the extension" message.
- `rule` — maps 1:1 onto the existing `MatchRule` interface in
  `src/lib/url.ts` (`from`, `to`, `exclude?`, `mode?`, `testUrl?`). `enabled`
  is intentionally **not accepted** from the payload; the confirmation screen
  decides it (default: enabled, user can toggle).
- `meta` — display-only. `source` lets the confirmation screen link back to
  the marketplace entry for provenance; it is stored nowhere unless we later
  decide to keep provenance.
- Size limit: reject payloads over ~8 KB after decoding.
- Multiple rules: `rule` may instead be `rules: MatchRule[]` (bounded, e.g.
  max 20) so curated bundles work; the confirmation screen lists each one with
  its own checkbox.

## Confirmation UX (install page)

A new route/section in the existing options app (Svelte), reusing the
`RuleDialog` field layout:

1. **Header** — "Install rule from community marketplace" with the `meta.name`
   and a link to `meta.source`.
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
   - redirect target host differs wildly from match host → informational note.
5. **Actions** — **Add Rule** (appends via existing storage write, respecting
   the current sync/local storage mode) and **Cancel** (closes/navigates to
   the rules list). A toggle chooses enabled/disabled on install
   (default enabled).
6. After install, land on the normal rules list with the new rule highlighted
   and a success toast.

## Security Considerations

- **No remote fetch.** The rule travels inside the link. There is no "install
  from URL" indirection, so a marketplace page can't swap the payload after
  review, and the extension needs no new host access at install time.
- **Origin allow-list.** The background interception only triggers on the
  configured marketplace origin(s) + exact install path. The list ships in the
  extension (constant), so adding origins requires an extension release —
  deliberate friction. Anyone can still craft the deep link manually, but it
  only works from allow-listed pages; pasting a payload elsewhere degrades to
  the fallback page.
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

## Implementation Sketch (extension repo)

Small, contained changes:

1. `src/lib/install.ts` — payload decode + schema validation + duplicate/
   broad-pattern checks (pure functions, unit-testable with vitest like
   `check.ts`).
2. `src/entrypoints/background.ts` — add the marketplace-origin navigation
   listener that rewrites the tab to the internal install page (a few lines,
   same pattern as existing redirect handling).
3. `src/entrypoints/options/` — an `#/install` route/state in `App.svelte`
   showing the confirmation screen; reuse `RuleDialog` fields,
   `RuleCheckResult`, `matchRule`, and the `rules` store's add path.
4. Tests — decode/validate unit tests; a store-level test that confirming
   appends exactly one normalized rule; a background test for the origin/path
   gate.

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
