# Purchased Template / Imported Codebase — Intake Audit

**StynerDev standard operating procedure.**
Run this on *every* template, theme, or third-party codebase before a single line of client work goes into it.

> **Why this exists.** A commercially purchased HTML template shipped to a non-profit client with an
> AddThis tracking script hard-coded into two separate JS bundles (using the *vendor's* publisher ID),
> an Envato affiliate "Buy Now" button injected into `<body>` on every page, a link to the vendor's own
> helpdesk, and a Google Maps API call using the *vendor's* API key pointed at Venice, Italy.
> None of it was documented. All of it ran on every page load. **Assume every template does this until proven otherwise.**

---

## 0. Before you start

> **Shell note.** These commands are written for **zsh** (macOS default). `--include` globs are
> quoted because zsh expands them otherwise — `grep -r --include='*.js'`, not `--include=*.js`.
> In bash either form works.

```bash
# Work on a copy. Snapshot first — most templates are not in git.
cp -R ./template ./template.ORIGINAL
cd ./template && git init && git add -A && git commit -m "vendor baseline, untouched"
```

Having a git baseline turns every later question ("did I break this or did it ship broken?") into a one-line diff.

**Note the file encoding immediately** — most templates ship **CRLF**. Regex and `sed` patterns
anchored on `\n` will silently fail to match. This wastes hours if you find it late.

```bash
grep -c $'\r' index.html          # >0 means CRLF; allow for \r in every pattern you write
file index.html                   # confirms line terminators + charset
```

---

## PHASE 1 — Phone-home audit (do this first, always)

**The question:** what does this codebase contact, and with whose credentials?

### 1.1 Every outbound domain in the whole tree

```bash
python3 - <<'PY'
import os,re,collections
DOM=re.compile(rb'(?:https?:)?//([a-zA-Z0-9.-]+\.[a-zA-Z]{2,})')
skip={'.png','.jpg','.jpeg','.gif','.ico','.woff','.woff2','.ttf','.eot','.otf','.svg','.mp4','.webp'}
hits=collections.defaultdict(set)
for root,dirs,fs in os.walk('.'):
    dirs[:]=[d for d in dirs if d not in ('.git','node_modules')]
    for f in fs:
        p=os.path.join(root,f)
        if os.path.splitext(f)[1].lower() in skip: continue
        try: b=open(p,'rb').read()
        except: continue
        for m in DOM.finditer(b):
            hits[m.group(1).decode('utf-8','replace').lower()].add(p)
for d in sorted(hits,key=lambda x:-len(hits[x])):
    print(f"{d:44} {len(hits[d]):3}  {sorted(hits[d])[0][:60]}")
PY
```

Ignore XML/spec namespaces (`w3.org`, `schema.org`, `purl.org`, `inkscape.org`, `ns.adobe.com`).
**Everything else needs a justification.**

### 1.2 Narrow to what actually loads at runtime

A domain in a comment is noise. A domain in a `src=` is a request.

```bash
# external resources fetched on page load (not links)
for f in *.html */index.html; do
  grep -ohE 'src="https?://[^"]+"' "$f" | sed -E 's/.*="//;s|(https?://[^/]+).*|\1|'
done | sort | uniq -c | sort -rn

# external @import / url() inside CSS and inline <style>
grep -rohE "@import url\(['\"]?https?://[^)]+" . --include='*.css' --include='*.html' | sort -u
grep -rohE "url\(['\"]?https?://[^)]+" . --include='*.css' --include='*.html' | sort -u
```

### 1.3 Dynamic injection — where trackers actually hide

The AddThis script in the reference case was **not** in any HTML file. It was a string inside a
minified JS bundle, appended to `<body>` at runtime. Grep the *behaviour*, not the domain:

```bash
grep -rnoE 'document\.write|createElement\(["'"'"']script|\.append\(|insertAdjacentHTML|innerHTML *=|eval\(|new Function\(|\.getScript|fetch\(|XMLHttpRequest|navigator\.sendBeacon' \
  --include='*.js' . | grep -v '\.min\.js:' | head -40
```

For every hit, read the surrounding 300 characters. You are looking for a URL string being built or appended.

```bash
# dump URL string-literals from ONLY the JS the pages actually load
for f in js/custom.js js/combining.js js/main.js; do
  echo "--- $f"; grep -ohE '(https?:)?//[a-zA-Z0-9.-]+\.[a-z]{2,}[^"'"'"' ,)]*' "$f" 2>/dev/null \
    | sed -E 's|^(https?:)?//([^/]+).*|\2|' | sort | uniq -c | sort -rn | head
done
```

⚠️ **Check the minified bundle separately.** Vendors frequently ship `custom.js` *and* a bundled
`combining.js`/`app.min.js` that contains a **duplicate copy** of the same injection. Cleaning one
and shipping the other is the classic miss. In the reference case both were live — the duplicate
buttons stacked at identical `position:fixed` coordinates, so the page *looked* correct.

### 1.4 Hard-coded credentials that aren't yours

```bash
grep -rnoE 'AIza[0-9A-Za-z_-]{35}|pk_live_[0-9A-Za-z]+|UA-[0-9]+-[0-9]+|G-[A-Z0-9]{10}|GTM-[A-Z0-9]+|pubid=[a-z0-9-]+|sk_[a-z]+_[0-9A-Za-z]+|AKIA[0-9A-Z]{16}' \
  . --include='*.js' --include='*.html' --include='*.php' --include='*.json' | head -20
```

Any Google Maps key, GA property, GTM container, or AddThis `pubid` you find is **the vendor's**.
Remove it. Never ship it — you are feeding their analytics with your client's traffic, and the key can
be revoked or quota-exhausted without warning.

### 1.5 Affiliate / promo injections

```bash
grep -rnoiE 'envato|themeforest|codecanyon|1\.envato\.market|affiliate|ref=|utm_source=|buy.?now|purchase.?now|support\.[a-z0-9-]+\.com' \
  --include='*.js' --include='*.html' . | grep -viE 'license|copyright' | head -30
```

### 1.6 Kill list

| Found | Action |
|---|---|
| Analytics / tracking script (AddThis, GA, Hotjar, FB Pixel) | **Remove.** Re-add only what the client explicitly consents to. |
| Affiliate or "Buy Now" link | **Remove.** |
| Vendor helpdesk / support widget | **Remove.** |
| Vendor API key (Maps, reCAPTCHA, etc.) | **Remove.** Replace with the client's own key or drop the feature. |
| Remote font / CDN (`fonts.googleapis`, `cdnjs`, `bootstrapcdn`) | **Self-host** — see Phase 4. |
| Third-party library license header (jQuery, Bootstrap, FA) | **KEEP.** MIT and similar *require* it. Removing it is the violation. |

> **Draw this line clearly.** Vendor promo ≠ library license. Strip the first, preserve the second.
> Document the second in a `LICENSE.txt` you ship.

When neutralising an injection inside a minified comma-expression, **replace the payload, don't delete the call** —
`jQuery("body").append('')` keeps the syntax valid. Deleting the whole expression leaves `,)` and breaks the bundle.

```bash
node --check js/combining.js && echo "syntax OK"   # ALWAYS verify after editing minified JS
```

---

## PHASE 2 — Integrity baseline

Know what shipped broken *before* you change anything, so you never chase your own tail.

```bash
# every local ref in every page, resolved against disk
for f in $(find . -name '*.html' -not -path './node_modules/*'); do
  grep -oE '(src|href)="[^"#]+"' "$f" | sed -E 's/.*="//;s/"//' \
    | grep -vE '^(https?:|//|mailto:|tel:|javascript:|#|data:)' | sort -u \
    | while read -r r; do [ -e "$(dirname "$f")/$r" ] || [ -e ".${r}" ] || echo "MISSING  $f -> $r"; done
done
```

Templates routinely reference plugin folders that were never packaged.
**Record the count.** That number is your "shipped broken" baseline — anything above it later is yours.

```bash
# tag balance, per file — compare before/after every bulk edit
for f in *.html; do
  printf "%-34s div %s/%s  li %s/%s\n" "$f" \
    "$(grep -oE '<div[ >]' $f|wc -l)" "$(grep -c '</div>' $f)" \
    "$(grep -oE '<li[ >]' $f|wc -l)" "$(grep -c '</li>' $f)"
done
```

---

## PHASE 3 — Placeholder & fake-content sweep

Everything below is fiction until replaced. Clients notice these; they read as unfinished work.

```bash
# lorem and filler
grep -rioE 'lorem|ipsum|dolor sit|consectet|dummy text|Aenean|Vestibulum|Praesent|Curabitur|Schlitz' --include='*.html' . | wc -l

# fake people
grep -rnoE '\b(John Doe|Jane Doe|John Smith|Admin|demongo|Robin Miles|Ethan Hunt|Logan Cee|Lorem Name)\b' --include='*.html' . | head

# fake contact details — these are the dangerous ones
grep -rnoE '\+?[0-9]{2,4}[)\-. ]?[0-9]{3,4}[\-. ][0-9]{3,4}|[a-z0-9._%-]+@[a-z0-9.-]+\.[a-z]{2,}' --include='*.html' . | head -20

# fake / stale dates and copyright years
grep -rnoE '\b(19|20)[0-9]{2}\b' --include='*.html' . | grep -vE 'speed|delay|width|height' | head -20

# stock coordinates (a template map is never pointed at your client)
grep -rnoE 'LatLng\([-0-9.]+, *[-0-9.]+\)|lat: *[-0-9.]+|lng: *[-0-9.]+' --include='*.js' .

# dead links that go nowhere
grep -rohE 'href="(javascript:void\(0\);?|#)"' --include='*.html' . | wc -l
```

**Never invent contact details.** If the client publishes no phone/email/address, remove the block or
leave an obvious `TODO` — do not leave the template's fake ones, and do not fabricate plausible ones.
A fake address on a live site is worse than no address.

---

## PHASE 4 — Privacy hardening

### 4.1 Self-host the fonts

```bash
# 1. what's requested?
grep -rohE "fonts\.googleapis\.com[^'\")]*" --include='*.css' --include='*.html' . | sort -u

# 2. which families/weights does the CSS ACTUALLY use? (don't download all 9 weights blindly)
grep -ohE 'font-family:[^;}]*' css/*.css | sed -E "s/font-family: *//;s/[\"']//g" | tr ',' '\n' | sort -u
grep -ohE 'font-weight: *[0-9]+' css/*.css | grep -oE '[0-9]+' | sort -n | uniq -c
```

Then fetch `css2?family=X:wght@...` with a **modern browser User-Agent** (Google serves woff2 only to
modern UAs; a default curl UA gets you bloated ttf), keep only the `latin` / `latin-ext` subsets,
download each `fonts.gstatic.com` file locally, rewrite the `@font-face` `url()` to a local path, and
delete the `@import`.

Check for the `@import` in **three** places: linked `.css`, minified `.min.css`, **and inline `<style>` blocks in the HTML**.
The inline one is the most commonly missed.

### 4.2 Content-Security-Policy

Meta tag works with no server config; ship header files too for the directives meta can't carry
(`frame-ancestors`, HSTS).

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; base-uri 'self'; object-src 'none'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; frame-src https://www.youtube.com; connect-src 'self'; form-action 'self'">
<meta name="referrer" content="strict-origin-when-cross-origin">
```

`'unsafe-inline'` is a pragmatic concession for templates full of inline handlers — it still blocks
**external** script origins, which is the whole point here. Tighten to hashes/nonces when you can.

Ship `.htaccess` (Apache) **and** `_headers` (Netlify/Cloudflare) with:
`Content-Security-Policy`, `Referrer-Policy`, `Permissions-Policy` (deny camera/mic/geo/interest-cohort),
`X-Content-Type-Options: nosniff`, `X-Frame-Options`, `Strict-Transport-Security`.

### 4.3 Verify the result

```bash
# should print "none" for every page except deliberate embeds
for u in / /about/ /contact/; do
  printf "%-18s %s\n" "$u" "$(curl -s "http://127.0.0.1:8000$u" \
    | grep -oE 'src="https?://[^"]+"' | sed -E 's|.*="(https?://[^/]+).*|\1|' | sort -u | tr '\n' ' ')"
done
```

---

## PHASE 5 — Ownership & attribution

```bash
# vendor fingerprints
grep -rnoiE '<meta[^>]*(author|generator|designer)[^>]*>' --include='*.html' . | head
grep -rnoiE 'themeforest|envato|dexign|w3itexperts|<!--.*(template|theme|author|created by).*-->' --include='*.html' . | head
```

Add to **every** page and to your own CSS/JS:
- Banner comment above `<!DOCTYPE>` — ownership, all-rights-reserved, developer name + URL
- `<meta name="author">`, `designer`, `copyright`
- `/humans.txt` + `<link rel="author">`
- `/LICENSE.txt` listing retained third-party licenses

> Code-level credit survives; a footer line is the first thing edited out. Ask the client which they want visible.
>
> **CSS class prefixes** (`dlab-`, `tp-`, `elementor-`) are a namespace, not attribution.
> Renaming ~200 classes risks breaking layout for zero gain. Leave them.

---

## PHASE 6 — Structure & delivery

```bash
# .html in URLs and a directory listing at / both read as amateur.
# Convert each page to <slug>/index.html, make the homepage index.html.
```

- Rewrite asset paths to **root-relative** (`/css/…`) so they survive the move to a subdirectory
- Grep JS for **page-relative** plugin paths (`jsFileLocation:"plugins/…"`) — these resolve against the
  *page*, not the script, and break silently the moment a page moves one level deep
- Move all backups/archives **out of the web root** — `_removed-pages/` returning `200` means Google indexes it
- Re-run the Phase 2 link check afterwards; expect **0 broken**

---

## Sign-off checklist

Do not hand over until every line is checked.

- [ ] Git baseline of the untouched vendor package exists
- [ ] Every outbound domain enumerated and justified
- [ ] Zero analytics/tracking scripts (checked source **and** every minified bundle)
- [ ] Zero affiliate / promo / vendor-helpdesk links
- [ ] Zero vendor API keys, GA/GTM IDs, or `pubid`s
- [ ] Minified bundles re-checked for duplicate copies of any removed injection
- [ ] `node --check` passes on every edited JS file
- [ ] Fonts and all CDN assets self-hosted; no `fonts.googleapis` / `gstatic` anywhere
- [ ] CSP + security headers shipped (meta **and** `.htaccess`/`_headers`)
- [ ] Third-party library licenses **preserved**; `LICENSE.txt` written
- [ ] Developer credit in banner comments, meta tags, `humans.txt`, `LICENSE.txt`
- [ ] Zero lorem/placeholder text, fake names, fake contact details, stock coordinates
- [ ] No fabricated contact info anywhere
- [ ] Dates and copyright year correct and consistent across all pages
- [ ] Clean URLs; no directory listing at `/`
- [ ] Archives moved outside the web root and confirmed `404`
- [ ] Full link+asset crawl: **0 broken** (or exactly the documented pre-existing count)
- [ ] Tag balance unchanged vs. baseline on every edited file

---

## The prompt

Paste this to an AI agent working in the codebase.

```text
You are auditing a commercially purchased HTML/CSS/JS template before it goes into client work.
Assume the vendor embedded self-serving code that is not documented anywhere. Your job is to find
all of it. Work through the phases in order and REPORT FINDINGS BEFORE CHANGING ANYTHING.

PHASE 1 — PHONE-HOME AUDIT
1. Enumerate every external domain referenced anywhere in the tree. Exclude XML/spec namespaces.
2. Separate "referenced in a comment" from "actually requested at runtime". Only the second matters.
3. Grep for dynamic injection behaviour, not just domain names: document.write, createElement('script'),
   .append(, innerHTML=, insertAdjacentHTML, eval(, new Function(, fetch(, XMLHttpRequest, sendBeacon.
   Read 300 chars of context around every hit.
4. CRITICAL: check minified bundles separately from source files. Vendors ship duplicate copies of the
   same injection in both. Cleaning one and shipping the other is the standard failure.
5. Find hard-coded credentials that belong to the vendor: AIza… Maps keys, UA-/G-/GTM- analytics IDs,
   addthis pubid, any *_live_ or sk_ token.
6. Find affiliate/promo links: envato, themeforest, codecanyon, ref=, utm_source=, "Buy Now",
   vendor helpdesk URLs.

Report a table: what was found, which file, whether it executes at runtime, and what it leaks.

Then remove: trackers, affiliate links, vendor helpdesk widgets, vendor API keys.
Do NOT remove third-party library license headers (jQuery/Bootstrap/Font Awesome) — those are legally
required. Distinguish vendor promo from library license explicitly.
When editing minified JS, replace the payload (e.g. append('') ) rather than deleting the call, so the
comma-expression stays valid. Run `node --check` on every file you touch.

PHASE 2 — INTEGRITY BASELINE
Resolve every local src/href in every HTML file against disk and report broken ones. This is the
"shipped broken" baseline — record the number before making changes. Record div/li tag balance per file.

PHASE 3 — PLACEHOLDER SWEEP
Find and report: lorem/filler text; fake names (John Doe, Admin, demongo…); fake phone numbers,
emails and addresses; stale dates and copyright years; stock map coordinates; dead href="javascript:void(0)"
links. NEVER invent replacement contact details — if the client publishes none, remove the block or
leave an explicit TODO.

PHASE 4 — PRIVACY HARDENING
Self-host all remote fonts and CDN assets. Check for @import in linked CSS, minified CSS, AND inline
<style> blocks in the HTML. Download only the families/weights the CSS actually uses, latin subsets only.
Then add a Content-Security-Policy (meta tag + .htaccess + _headers), Referrer-Policy,
Permissions-Policy, nosniff, X-Frame-Options, HSTS.
Verify: no page should fetch any external host at load time except deliberate embeds.

PHASE 5 — OWNERSHIP
Remove vendor attribution meta/comments. Add developer credit as a banner comment above <!DOCTYPE>,
meta author/designer/copyright, humans.txt with <link rel="author">, and LICENSE.txt listing the
retained third-party licenses. Leave CSS class-name prefixes alone — they are a namespace, not credit.

PHASE 6 — STRUCTURE
Convert to clean directory URLs (<slug>/index.html), homepage as index.html. Rewrite asset paths to
root-relative. Grep JS for page-relative plugin paths (e.g. jsFileLocation) which break when pages move
a level deep. Move backups out of the web root and confirm they 404.

RULES THROUGHOUT
- Check line endings first: templates are usually CRLF, so allow for \r in every pattern. A pattern
  that silently matches 1 of 9 files is the failure mode — always verify the match COUNT per file.
- After every bulk edit, re-verify: tag balance vs baseline, full link crawl, node --check on JS.
- Report honestly: if a scan errored or was inconclusive, say so — do not claim clean on an unrun check.
- Distinguish "referenced but never called" from "actively executing" before recommending removal.
```

---

*StynerDev — https://www.styner.dev/*
