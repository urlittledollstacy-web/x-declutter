# AGENTS.md

## Project

`x-declutter.txt` — a Brave/AdBlock cosmetic filter list that declutters X (Twitter)
on Android and desktop. Hides Grok, Premium/Subscribe upsells, and sidebar nav
entries for Calls, Ads, Creator Studio, Business, Lists, Communities.

No build step. The file *is* the deliverable.

## Serving the file

The user is on Android and cannot copy from chat, so the list is served over HTTP:

- `python3 /tmp/serve_x.py` — BaseHTTP server on port 12000
- Public URL: https://work-1-rnaihkzzjjgwaotq.prod-runtime.all-hands.dev/x-declutter.txt
- **Reads the file from disk on every request — no restart needed after edits.**
  (An earlier version cached at startup; that is no longer true. Verified by
  appending a marker and re-fetching without restarting.)
- **Sends no `Cache-Control` header at all** — only `Last-Modified`. An earlier
  note claimed `no-cache, no-store, must-revalidate`; that was wrong. Because
  there is no cache directive, the browser/proxy *may* serve a stale copy, so
  after editing tell the user to remove and re-add the list in Brave, not just
  "Update now".
- **`/tmp` is wiped on every sandbox restart**, which kills the server process.
  Rebuild `serve_x.py` and restart it after any restart. The list itself lives
  in `/workspace/project` and survives.

User installs it in `brave://adblock` via "Add filter list by URL", then taps
"Update now" to pull changes.

## Validating changes

```python
import adblock
fs = adblock.FilterSet(); fs.add_filter_list(blob, 'standard', True, 'all')
adblock.Engine(fs, True).url_cosmetic_resources('https://x.com/home')
```

Check for: `parse failures: 0` and a sane hide-selector count.

### CRITICAL testing caveat

`url_cosmetic_resources()` and `Engine.serialize()` **cannot see procedural
rules** (`:has-text()`, `:has()`, `:upward()`, `:xpath()`, `:matches-css()`).

All of these report 0 selectors and zero serialize delta — *including rules that
demonstrably work*. This means:

- A `0` from these APIs is **not** evidence that a procedural rule is broken.
- Do **not** conclude "`:has()` is unsupported" from a `0`. That claim was made
  twice in this project and was never actually verified.
- The only reliable test is the user's own report of what is still visible.

## Hard-won facts about X's markup

Verified by reading X's own client bundles and live server-rendered HTML.

### Element types

| Attribute | In real X markup |
|---|---|
| `role="button"` | **exists** (8x on a profile page) |
| `role="menuitem"` | **does not exist** (0x, ever) |
| `data-slot="xds-menu-trigger"` | exists (16x) |
| `data-slot="xds-button"` | exists (4x) |

**Any `div[role="menuitem"]` rule is dead weight.** Do not add them.

### Real nav targets

From `assets/sidebar-nav-*.js` and `assets/nav-link-*.js`:

| Item | Target | Notes |
|---|---|---|
| Lists | `/<handle>/lists` | real `<a>` |
| Communities | `/<handle>/communities` | real `<a>` |
| Creator Studio | `/i/jf/creators/studio` | |
| Business | **`/i/premium-business`** | `id:vo-signup`, renders as `div[role="button"]` |
| Ads | `ads.x.com` (external) | |
| Calls | `/i/calls` | |

`/i/business` 307-redirects to `/i/verified` — it is **not** a Business destination.
`/i/ads`, `/i/ads_manager`, `/i/creator-studio`, `/i/creator_studio` are **not** real routes.

### The Business row specifically

- Sidebar def: `{id:`vo-signup`, to:`/i/premium-business`, labelKey: ... , decoration:a}`
- `nav-link-*.js` sets `decoration = "40% off"` on this row
- Renders as `div[role="button"]`, **not** an `<a>` — so no href rule can match it
- The "40% off" badge means its text is not just "Business", so anchored
  `:has-text(/^Business$/)` fails. Lists/Communities have no badge, which is why
  their anchored rules worked and Business's never did.

### The rule that actually works

```
x.com##div.r-18u37iz.r-1awozwy.css-g5y9jx:nth-of-type(9)
```

Purely **declarative** (no `:has-text`), so it is guaranteed to execute.

Fragile: `:nth-of-type(9)` is positional and `css-g5y9jx` is an emotion class hash
that rotates each deploy. Expected to break when X adds/removes a nav row.

**Never use a bare `##` prefix** — it applies to every site. Always scope to `x.com##`.

## Root cause of the long debugging loop

Every nav item except Business had a working **declarative** href rule. Business
had no usable href, so its only possible match was a **procedural** `:has-text()`
rule, which did not take effect. The item that stayed visible was precisely the
one with no declarative fallback.

Lesson: prefer declarative selectors (attributes, classes, `href`) over
procedural text matching. When a rule must be procedural, treat it as unverified
until the user confirms it works.

## Debugging methodology

What wasted the most time here was **inferring targets from URL patterns** instead
of reading them. Four revisions guessed at paths (`/i/business`, `/i/ads`, …) that
did not exist.

Do this instead:

1. Read X's own client bundles (`assets/sidebar-nav-*.js`, `nav-link-*.js`) for
   the authoritative nav table — not guesswork.
2. Check live server-rendered HTML (`curl https://x.com/<handle>`) for real
   element types and attributes. Note: grep it with `-a`, the HTML is binary-ish.
3. Ask the user for the **element-picker output** or **outerHTML** of a stubborn
   element. This is what finally solved Business — after four failed guesses.
4. When a positional picker path is the only thing that works, prefer it over
   guessing, but flag the fragility and ask for `outerHTML` so a stable
   attribute-anchored rule can replace it.

### The Business row — SOLVED (rev 6)

The live DOM (captured via console) shows the row is a real `<a>`:

```html
<a href="/i/verified-orgs-signup" role="link" class="css-g5y9jx r-16y2uox ...">
  <div class="css-g5y9jx r-1awozwy r-18u37iz ...">
    <div class="css-146c3p1 ..."><span class="css-1jxf684 ...">Business</span></div>
  </div>
</a>
```

It lives inside `div[role="dialog"]` — the **"More" overflow menu**, not the
main sidebar.

**Three wrong paths were guessed before this, at ~2 revisions each:**

| Path | Reality |
|---|---|
| `/i/business` | 307 → `/i/verified`. Not the nav target. |
| `/i/premium-business` | In nav config as `to:`, and the route table maps `/i/verified-orgs-signup` → `/i/premium-business`. But **not the rendered href**. |
| `/i/verified-orgs-signup` | **The actual rendered href.** |

The bundle confirms the mapping:
```js
OK=U(`/i/verified-orgs-signup`)({beforeLoad:()=>{throw J({to:`/i/premium-business`,search:!0})}})
```

Also wrong: the earlier belief that the row was a `div[role="button"]` and that
`:has-text` must therefore be unanchored. It is an `<a>`, so plain href rules
work.

**Working rule (declarative, stable):**
```
x.com##a[href^="/i/verified-orgs-signup"]
x.com##a[href*="verified-orgs-signup"]
```

This replaced the fragile positional rule
`div.r-18u37iz.r-1awozwy.css-g5y9jx:nth-of-type(9)`, now commented out.

**Lesson:** the config value (`to:`) is not necessarily the rendered `href`.
Read the live DOM, not just the bundle.

### Class names are runtime-injected

X's `css-*` / `r-*` classes do **not** appear in the static `.css` files fetched
from `abs.twimg.com` (verified: 0 occurrences across 303 KB of CSS). They are
generated and injected at runtime by emotion. So you cannot validate a
class-based rule by grepping downloaded CSS — only against the live DOM.

The Business row's **label** element looks like:

```html
<div dir="ltr" class="css-146c3p1 r-dnmrzs r-1udh08x ... r-16y2uox" style="color:...">
  <span class="css-1jxf684 r-bcqeeo r-1ttztb7 r-qvutc0 r-poiln3">Business</span>
</div>
```

This is the **label, not the row**. The row is an *ancestor* with `role="button"`
containing the icon + this label. Hiding the label alone leaves the icon and the
clickable area behind. CSS cannot select ancestors (no `:has`), so this markup is
**not directly usable** to hide the row.

## RESOLVED: the profile Grok button

**Fixed by targeting the Grok logo SVG, not the button.**

The button carries no stable attribute of its own. `aria-label` differs by
surface (`"Grok"` in the desktop profile header, `"Grok actions"` on per-post
buttons), and `data-icon` is present in SSR but **dropped at hydration**.
The SVG that survives has two usable hooks:

| hook | SSR | live | unique on page |
|---|---|---|---|
| `data-icon="icon-grok-logo"` | yes | **gone** | n/a |
| `viewBox="0 0 33 32"` | yes | **yes** | 1 (all other icons are 24x24) |
| `path d="M12.745 20.54..."` | yes | **yes** | 1 (557 chars) |

Working rules:

```
x.com##svg[viewBox="0 0 33 32"]
x.com##button:has(svg[viewBox="0 0 33 32"])
x.com##div[role="button"]:has(svg[viewBox="0 0 33 32"])
x.com##path[d*="M12.745 20.54"]
```

Hiding the `svg` alone leaves an empty button, so the `:has()` rules hide the
button around it. The `path` rule is a fallback in case X changes `viewBox`.

### Two traps worth remembering

1. **`data-icon` does not survive hydration.** Any rule keyed on it matches
   nothing on the live DOM. `viewBox` and `path d` do survive.
2. **The Python `adblock` binding (0.6.0) cannot parse procedural filters.**
   `:has-text()`, `:has()`, `:upward()` all return an empty selector list and
   `filter_exists() == False`. This is a limitation of the test tool, **not**
   evidence the rule is dead. Brave 1.73+ supports procedural filtering. Do not
   conclude a rule is broken from this binding alone - confirm in the browser.

## Decoy test: the ONLY reliable way to prove the list is active

Testing whether known-hidden elements are hidden **does not work on mobile**.
Business / Lists / Communities are left-sidebar items, and at narrow widths
(<640px) the sidebar is not rendered at all, so those selectors match 0
elements regardless of whether any filter is running. A "not found" verdict is
therefore meaningless. Likewise the desktop profile header is `sm:flex hidden`,
so it is absent on a phone.

The reliable method is to **inject decoy elements** that match the rules and read
their computed `display`. adblock-rust emits `hide_selectors`, which the browser
applies as a stylesheet, and CSS applies to elements added later too. So:

```js
var el = document.createElement('button');
el.setAttribute('aria-label', 'Grok actions');
document.body.appendChild(el);
getComputedStyle(el).display === 'none'   // true => list is active
```

Include a **control** decoy (e.g. `aria-label="Search"`) that must stay visible,
otherwise the test itself is broken.

This also distinguishes two cases that look identical otherwise:
- list not active in this browser
- list active but the selector does not match the real element

### Likely cause of the long Grok thread

Brave for Android has no developer console. If the snippet is run in a *different*
browser from the one with the filter list installed, every selector will still
"match" (that is just `querySelectorAll`), while nothing is ever hidden. Confirm
which browser holds the list before debugging selectors again.

## Profile header action bar (Back | name | Grok | Search)

The user's description — "top right, beside the search button" — matches this
SSR structure exactly:

```
div.bg-primary/80.sticky.top-0.z-30.h-[53px].backdrop-blur-md
  [0] <button aria-label="Back">
  [1] <div>  name / @handle
  [2] <button aria-label="Grok" data-slot="xds-button">   <- TARGET
  [3] <button aria-label="Search">                        <- "beside it"
```

Key facts:
- The header exists **only on a profile page** (`/<handle>`), inside `<main>`.
- Its class list ends in **`sm:flex hidden`** — so at <640px (Android phone) this
  desktop header is **not rendered**; mobile uses a different header. The mobile
  DOM has not been captured.
- The Grok button here has `aria-label="Grok"` **and** `data-slot="xds-button"`,
  unlike the per-post "Grok actions" buttons.

### Diagnostic lesson

A report run on `https://x.com/home` cannot say anything about this button — the
header is not on that page. The diagnostic must be run on a **profile page**.

Also note: the diagnostic snippet tests selectors with `querySelectorAll` in JS.
It does **not** test whether the filter list is applying them. Those are
different questions — a selector can "match" in JS while the list is inactive.

## Grok button — rev 9

The **live DOM differs from SSR**, which caused several wasted rounds:

| | Markup |
|---|---|
| SSR | `<button data-slot="xds-button" aria-label="Grok">` — never matches live |
| LIVE | `<a href="/i/grok" aria-label="Grok" role="link">` — 1, already hidden |
| LIVE | `<button aria-label="Grok actions" role="button">` — **3, these show** |

The visible button's label is **"Grok actions"**, not "Grok". The SVG's
`data-icon` attribute exists in SSR but is **dropped at hydration**, so
`[data-icon*="grok"]` matched nothing live and was removed in rev 9.

Working rules:
```
x.com##button[aria-label="Grok"]
x.com##button[aria-label="Grok actions"]
x.com##[aria-label*="Grok" i]      <- load-bearing catch-all
x.com##a[href^="/i/grok"]
```

### IMPORTANT: verify the list is active before debugging selectors

A user report showed Business and Lists as `display=flex` even though both were
confirmed hidden — meaning the list was not applying in that tab at all. Check
this FIRST.

Also: `getComputedStyle(e).display` does **not** catch a hidden ancestor. Use
`e.offsetParent === null` to test whether an element is really hidden. An earlier
diagnostic snippet used the former and produced misleading output.

## Future work

- Re-verify the 8 `:has-text()` rules once the list is confirmed active. If they
  are dead, remove them.
- Capture the **mobile** profile header DOM. The desktop header is `sm:flex
  hidden`, so on an Android phone the Grok button is a different element that has
  not yet been seen.

## Known issues in the current list (audited rev 7)

### Confirmed bugs, fixed

1. **rev 5** — `x.com##a[href*="business.x.com"]` was hiding the footer
   **"Ads Info"** link (`business.x.com/en/help/troubleshooting/...`), not the
   sidebar Business row. Removed.
2. **rev 7** — `a[href$="/lists"]:not([href="/lists"])` (and the `/communities`
   twin) matched **absolute external URLs** that merely ended in `/lists`, e.g.
   `https://example.com/lists` or `https://reddit.com/r/communities` linked in a
   tweet. Fixed by adding `[href^="/"]`, which restricts the match to relative
   (internal x.com) links.

### Redundancy trimmed (rev 7)

`a[href*="/i/premium"]` already covers `/i/premium_sign_up` and
`/i/premium-business`; `a[href*="/creators/studio"]` already covers
`/i/jf/creators/studio`. Removed the subsumed rules. 24 -> 20 active rules.

Note `/i/verified-choose` is **not** covered by `/i/premium` and needs its own
rule.

### Corrected claims

Earlier comments asserted `:has()` is "NOT supported" and that "only flat
`:has-text()` works". Neither was verified — the test method is blind to all
procedural rules. Rewritten to "could not be verified".

The "Business has no usable href, so it must use :has-text" theory was also
wrong: the row is an `<a href="/i/verified-orgs-signup">`. It always had an href.

### Open risks (cannot be verified without the user's logged-in DOM)

1. **~7 rules are procedural (`:has-text`) and unconfirmed.** They may be doing
   nothing, or may be working. Unknown. Every item the user confirmed hidden has
   a declarative rule, so these are likely redundant rather than load-bearing.
2. **The `:not([href="..."])` guards only exclude the relative form.** If X ever
   renders `@lists`/`@communities` links absolutely, the guard misses. The
   `[href^="/"]` guard added in rev 7 makes this *less* likely to bite, since an
   absolute account URL would no longer match at all.

### Audit-script pitfalls (made these twice)

- An `href$="/lists"` rule legitimately hides `/X/lists` (your own sidebar row).
  That is the **intended** behaviour, not a false positive.
- `a[href*="/i/premium"]` subsumes `/i/premium_sign_up` and
  `/i/premium-business`, but **not** `/i/verified-choose`. Verify subsumption
  by actually testing the substring, not by eyeballing rule names.
- Prefer the **substring** form for route rules: `*= "verified-orgs-signup"`
  also matches absolute URLs and query strings, where `^= "/i/..."` does not.


## The X logo in the top bar — SOLVED (rev 16)

Three separate rules failed before this one worked. The reason is worth
recording, because the failure mode was **testing against the wrong DOM**.

### What actually worked

```
x.com##path[d^="M21.742"]
```

### Why the first three attempts failed

The logo's markup differs by client. Every rule built from the *scraped
logged-out* page keyed off `aria-label="X"` — but the client that renders on
the user's device has **no `aria-label` on the logo at all**.

| Attempt | Selector | Why it failed |
|---|---|---|
| rev 14 | `div[class*="sm:hidden"] a[aria-label="X"]` | Tailwind classes absent in RN-Web build |
| rev 15 | `a[aria-label="X"]` | element has **no** `aria-label` |
| rev 16 | `path[d^="M21.742"]` | **works** |

The user's devtools returned:

```html
<svg viewBox="0 0 24 24" aria-hidden="true"
     class="r-4qtqp9 r-yyyyoo r-dnmrzs r-bnwqim r-lrvibr r-m6rgpd
            r-18jsvk2 r-16y2uox r-lwhw9o">
  <g><path d="M21.742 21.75l-7.563-11.179 ..."></path></g>
</svg>
```

`aria-hidden="true"`, no `aria-label`, no Tailwind class. So both attribute-based
rules matched zero elements.

### Three different logo renderings exist

Do not assume one:

| Variant | Structure | `d` starts | Classes |
|---|---|---|---|
| scraped desktop | `svg > path` | `m21.62 21.5` | Tailwind |
| scraped (alt) | `svg > g > path` | `M18.244 2.25` | `r-*` |
| **user's device** | `svg > g > path` | `M21.742 21.75` | `r-*` |

### Lessons

1. **Anchor on `path[d^=...]` for icons.** The glyph's `d` is a constant of the
   brand mark: stable across class-hash rotation, immune to attribute changes.
   It is the most robust icon selector available.
2. **Never trust a scraped logged-out page to represent a logged-in client.**
   Two rules were written and shipped against markup the user never receives.
   Ask for devtools output *early* on a stubborn element, not after three tries.
3. **`:has()` was NOT verified either way. Do not claim it is dropped.**
   An earlier version of this note asserted "adblock-rust silently drops `:has()`
   rules", citing `svg:has(path[...]) -> hide=[]` versus
   `path[d^="M21.742"] -> [selector]`. That reasoning was **wrong**: the probe
   returns `[]` for *every* procedural form, so `[]` proves nothing. Measured:

   | rule | `hide_selectors` |
   |---|---|
   | `div.plain-rule` | `['div.plain-rule']` |
   | `div[role="button"]:has-text(/^Lists$/)` | `[]` |
   | `button:has(svg[viewBox="0 0 33 32"])` | `[]` |
   | `a:has-text(Calls):upward(1)` | `[]` |

   `:has()` and `:has-text()` are indistinguishable under this probe. Whether
   the engine *keeps* or *drops* them is unknown. The `path[d^=...]` rule was
   chosen because it is **declarative**, not because `:has()` was proven dead.
   Only the user's on-device report can settle it. This is the same mistake the
   project made twice before (see "Corrected claims" above) - do not repeat it.
4. Five of the user's picker classes (`r-lwhw9o`, `r-16y2uox`, `r-1nao33i`,
   `r-m6rgpd`, `r-bnwqim`) appear in **none** of the scraped markup. That was the
   early signal that the DOMs differed, and it was available before rev 14.

### Delivery

The HTTP link dies on every sandbox restart and has now cost several rounds.
The user has a live GitHub Pages site (`https://urlittledollstacy-web.github.io/`)
which would make the list permanent via a subfolder — offered, awaiting consent,
since it publishes to their public repo.
