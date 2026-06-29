# CLAUDE.md

Guidance for building a **standalone kbenestad business-document app** from this
template. For the design system and the rules that keep an app pixel-identical to
the bizdocs family, see [DESIGN.md](DESIGN.md).

## What this is

This repository is **`basis`** — a self-contained starter for a single small web
app that produces a business document (invoice, quote, packing slip, contract,
…) as a PDF, looking and behaving **exactly** like the apps in the
[bizdocs](https://github.com/kbenestad/bizdocs) series, but shipping on its own
(its own repo, its own hosting), with **no build step and no backend**.

The whole app is **one `index.html`** (all HTML/CSS/JS inline) plus a
`config.yml` and a bundled `assets/` folder. You build an app by copying this
template and replacing the marked app-specific parts — never by hand-assembling
the chrome, which is what guarantees the pixel-perfect match.

## Repository layout

```
index.html          the app: chrome + boot + your form + your PDF (all inline)
config.yml          branding + all UI strings (localisation), read at boot
assets/
  style.css         design tokens / colour scheme + reset + page shell   ┐ shared
  ui.css            the kb-* UI component library                        │ design,
  app.js            shared runtime (DOM, theme, modals, i18n, config, …) ┘ DO NOT EDIT
  favicon.svg       placeholder icon — replace with a real set per app
  site.webmanifest
sync.sh             refresh the three shared files from bizdocs (see below)
CLAUDE.md · DESIGN.md · README.md
```

### The shared files are bundled, and must stay byte-identical

Unlike an in-series bizdocs app (which links one shared `../assets/`), this app
**bundles its own copy** of the design + runtime in `assets/` and references it
locally (`assets/style.css`, not `../assets/style.css`). That is what makes the
folder standalone.

The price of self-containment is duplication, so the discipline is strict:
**`assets/style.css`, `assets/ui.css` and `assets/app.js` are byte-identical
copies of bizdocs' shared files. Never hand-edit them.** A bizdocs UI change is
pulled in wholesale, not patched here:

```bash
BIZDOCS_REF=main ./sync.sh --from-github   # pull the three shared files from GitHub
# or, if a bizdocs checkout sits alongside this repo:
./sync.sh ../bizdocs/assets
git diff -- assets                         # review, then commit
```

If you ever need app-specific CSS, it goes in `index.html`'s inline `<style>`;
if you need app-specific copy, it goes through `config.yml` + `S()`. The moment
you edit the shared files directly, drop-in sync stops being clean and the app
drifts from the family. Don't.

## How the app boots

1. A tiny inline pre-paint script in `<head>` reads `localStorage['kb-theme']`
   and sets `data-theme` before first paint (avoids a flash). Keep it.
2. CDN libraries load: `js-yaml` (config) and `pdf-lib` (PDF output).
3. `assets/app.js` loads and defines the shared globals.
4. The inline `<script>` runs: `loadYamlConfig()` fetches + parses `config.yml`,
   **`assertValidConfig()` validates it** (see below), `normaliseConfig()`
   flattens the `localisation:` block, `applyAccent()` / `initFontScale()` /
   `applyLang()` apply settings, and `render()` builds the UI. State persists to
   `localStorage` under app-specific keys.

Boot order to preserve: `loadYamlConfig → assertValidConfig → normaliseConfig →
applyAccent → initFontScale → applyLang → render`.

### Config validation (keep this guard)

`config.yml` is fetched at runtime. On a static host a **missing** `config.yml`
— or an SPA fallback, or a stale cached page — is served as an **HTML page with
a 200**. `jsyaml.load()` then returns a plain *string*, `CFG.localisation` is
undefined, `normaliseConfig()` silently early-returns, and the app paints its
full chrome with **every label showing as a raw key, an empty language dropdown,
and no error at all**. It looks broken with no clue why.

`assertValidConfig()` in the inline boot turns that into a clear, actionable
error:

```js
function assertValidConfig(cfg) {
  if (!cfg || typeof cfg !== 'object' || !cfg.localisation)
    throw new Error('config.yml loaded but did not parse to a valid configuration. '
      + 'The server most likely returned an HTML page (a 404 / SPA fallback, or a '
      + 'stale cached page) instead of the YAML file. …');
}
```

Keep this in the inline boot — **not** in `assets/app.js`, which must stay
byte-identical to bizdocs. (`loadYamlConfig()` deliberately doesn't validate
shape; that's the app's job.)

## Building your app from this template

1. **Rename / rebrand.** Set the `<title>`, the doc-title `<h1>`, the brand
   fallback text, and the `basis-lang` localStorage key (and any other per-app
   keys) to your app's name.
2. **Build the form.** Replace `buildExampleCard()` with your real form, built
   **only from `kb-*` components** (see DESIGN.md). Put any bespoke layout
   (column grids, repeating rows, signature pad) in the inline `<style>`.
3. **Build the PDF.** Replace `onDownload()` with your real document using
   **pdf-lib** (`PDFLib`, already loaded).
4. **Fill in `config.yml`.** Keep the header keys; add every user-facing string
   to the `ui:` block of **every** language; route all copy through `S('key')`.
5. **Drop in real icons.** Replace `assets/favicon.svg` / `site.webmanifest`
   with a full favicon / PWA icon set.

## Running / previewing locally

The app `fetch`es `config.yml`, so it must be served over HTTP — opening
`index.html` via `file://` will fail.

```bash
python3 -m http.server 8000
# then open http://localhost:8000/
```

## Verifying a change

There are no automated tests — verify visually by rendering the app with a
headless Chromium:

```bash
CHROME=/path/to/chromium      # e.g. /opt/pw-browsers/chromium-*/chrome-linux/chrome
"$CHROME" --headless=new --no-sandbox --disable-gpu \
  --virtual-time-budget=8000 --window-size=1200,1600 \
  --screenshot=out.png "http://localhost:8000/index.html"
```

`--virtual-time-budget` lets the JS-built UI settle before the screenshot.

**Caveat:** in sandboxed environments the browser often cannot reach the CDNs,
so `js-yaml` fails to load and you'll see the config error. To verify a full
render, vendor `js-yaml` locally **for the test only** — download it, drop a copy
into `assets/`, point a throwaway copy of `index.html` at the local file, and
screenshot that. Delete the throwaway files afterwards; never commit them.
(`pdf-lib` is only needed to generate a PDF, not for the initial render.) Worth
screenshotting after a change: the full form, dark mode, the About modal, and a
non-English language.

## Conventions (the pixel-perfect rules)

- **Reuse the shared layer; never fork it.** Styling and cross-cutting logic
  live in `assets/`. Don't reintroduce design tokens, `kb-*` components, DOM
  helpers, or theme/modal/i18n/format code in the app — use what `app.js` and
  the CSS already provide.
- **Same classes/IDs for the same element** as the bizdocs apps, so a synced
  `ui.css` change lands correctly. Only genuinely app-specific layout belongs in
  the inline `<style>`.
- **Scope.** The main inline script is wrapped in an IIFE
  (`(async function(){ … })()`), so its functions are not globals; the globals
  from `app.js` ($, el, makeSizeControl, kbAbout, lookupString, …) are visible
  inside it.
- **localStorage keys.** Theme = `kb-theme`, font scale = `kb-font-scale`
  (shared, don't rename). Per-app data uses your own keys (e.g. `basis-lang`).
- **No hardcoded user-facing text.** Add a key to every language block in
  `config.yml` and look it up via `S()`. Use `{placeholder}` tokens for values.
- **PDF output uses pdf-lib.** It can draw from scratch and embed/append
  existing PDF/image bytes (receipts, signatures), so it covers every app.
- **The container is ephemeral / hosting is static.** Commit your work; deploy
  `index.html` + `config.yml` + `assets/` together as plain static files.
