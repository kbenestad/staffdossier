# DESIGN.md

The design system and architecture behind this template. The single goal of
this document: **everything you need to make a standalone app render
pixel-identically to the [bizdocs](https://github.com/kbenestad/bizdocs)
family.** For day-to-day build steps, see [CLAUDE.md](CLAUDE.md).

## Principles

- **Single-file app, no build, no backend.** The app is one `index.html` that
  runs from static hosting. State lives in the browser (`localStorage`); nothing
  is sent to a server.
- **Config-driven.** Branding and all UI copy come from `config.yml`. Code reads
  config; it does not hardcode an organisation's specifics.
- **Self-contained, but not forked.** This app bundles its own copy of the
  shared design + runtime in `assets/` so it can ship alone — but those files
  stay **byte-identical** to bizdocs' shared layer. The look is shared *by
  copying*, not by re-implementing. A single class/ID vocabulary means a bizdocs
  change drops straight in.

## The shared layer (`assets/`)

Three files, referenced locally by the app as `assets/…` (an in-series bizdocs
app references the same files as `../assets/…` — that path is the **only**
difference, and it's why this folder is standalone).

> These three files are verbatim copies of bizdocs' shared files. **Do not edit
> them here.** Pull updates with `sync.sh`. App-specific CSS → inline `<style>`;
> app-specific copy → `config.yml`.

### `style.css` — colour scheme & foundation
The single source of truth for the **colour scheme**. Defines:
- Design tokens as CSS custom properties on `:root`: accent, surfaces, text,
  status colours (danger/warning/success/info), radii, shadows, and a typography
  scale (`--fs-*`) driven by `--font-scale`.
- **Dark mode** in two forms: automatic via `@media (prefers-color-scheme:
  dark)`, and forced via `:root[data-theme="dark"]`. The pre-paint inline script
  in `<head>` sets `data-theme` from `localStorage['kb-theme']` before first
  paint.
- Reset, base typography, and the page shell (`.kb-wrap`).

Accent can be overridden per app via `accent-colour` in `config.yml` (applied by
`applyAccent()`) — that is the right way to recolour, not editing `style.css`.

### `ui.css` — reusable components
The `kb-*` component library — build your entire form from these so it matches:
toolbar (`.kb-toolbar`, `.kb-seg`, `.kb-iconbtn`, `.kb-sz-label`), header/brand
(`.kb-header`, `.kb-brand`, `.kb-doctitle`), cards (`.kb-card`,
`.kb-card__title`), form grids/fields (`.kb-grid`, `.kb-field`, `.kb-label`),
inputs (`.kb-input/.kb-select/.kb-textarea` with `.num`, `.is-error`,
`.is-warn`), buttons (`.kb-btn` + `--primary/--ghost/--soft/--dashed/--danger/
--lg/--sm/--block`, `.kb-circbtn`), notes/banners (`.kb-note--error/--warning/
--success/--info`), totals (`.kb-totals`), chips (`.kb-chip`), the modal
(`.kb-overlay`, `.kb-modal`, `.kb-modal__hdr/__body/__footer`), loading/error
states, and the footer (`.kb-footer`, `.kb-mark`).

App-specific layout (column grids, repeating rows, a signature pad) is **not**
here — it stays in the app's inline `<style>`.

### `app.js` — shared runtime
Cross-cutting JavaScript, loaded just before the app's inline script. It is a
classic (non-module) script, so the names below are globals visible to the app
script, including inside its IIFE.

| Area        | API |
| ----------- | --- |
| DOM         | `$`, `$$`, `el(tag, attrs, children)`, `uid()` |
| Markdown    | `markdown(md)` — minimal MD→HTML for About boxes |
| Brand/icons | `KB_BRAND_SVG`, `KB_FOOTER_MARK_SVG`, `KB_ICON` |
| Theme       | `currentTheme()`, `toggleTheme()`, `updateThemeIcon()`, `makeThemeButton()`, `makeAboutButton(onClick)`; key `KB_THEME_KEY = 'kb-theme'` |
| Modals      | `kbModal(opts)` and `kbConfirm` / `kbAlert` / `kbAbout` |
| Config      | `loadYamlConfig(url)`, `applyAccent(cfg)` |
| Numbers     | `formatAmount(n, {fallback, locale})`, `parseAmount(s)` |
| Dates       | `formatDate(iso, pattern, full, short)`, `MONTHS_FULL`, `MONTHS_SHORT` |
| Font scale  | `currentScale()`, `setFontScale()`, `bumpFontScale(dir)`, `initFontScale()`, `makeSizeControl()`; key `KB_SCALE_KEY = 'kb-font-scale'` |
| i18n        | `buildLangTable(loc)`, `lookupString(table, key, lang, defLang, vars)`, `pdfOutputLang(cfg, uiLang)` |

## The chrome (assemble it exactly)

`render()` builds the page in this order — keep it, this is the family layout:

1. `kb-toolbar`: language `<select>` · `.spacer` · `makeSizeControl()` (A−/A+) ·
   `makeThemeButton()` · `makeAboutButton(showAboutModal)`.
2. `kb-header`: the brand lockup (`KB_BRAND_SVG` tile + org name/tagline) on the
   left, the document title (`.kb-doctitle` `<h1>`) on the right.
3. Your content — `kb-card`s built from `kb-*` components.
4. The action row (`.kb-actions`): a ghost button and a primary Download button.
5. `kb-footer`: the `KB_FOOTER_MARK_SVG` mark, ©, a repo link, and About.

Don't hand-roll any of this differently — the template already wires it up.

## The `<head>` (preserve exactly)

In order: `<meta>`s and `<title>`; favicon + manifest; the **pre-paint theme
script**; the CDN libs (`pdf-lib`, `js-yaml`); then the three shared references —
`assets/style.css`, `assets/ui.css`, and `assets/app.js` **before** the app's
inline `<script>`; then the app's inline `<style>`.

## Config loading & validation

`loadYamlConfig()` (in `app.js`) only `fetch`es and `jsyaml.load()`s the text; it
does **not** validate shape (it can't know your required keys, and must stay
byte-identical). Validation is the boot's job — because the common deployment
failure is silent:

On a static host a missing `config.yml` (or an SPA fallback / stale cache) is
returned as an **HTML page with a 200**. `jsyaml.load()` parses that to a plain
*string*, so `CFG.localisation` is undefined, `normaliseConfig()` early-returns,
and the app paints its full chrome with every label as a raw key, an empty
language dropdown, and **no error**. `assertValidConfig()` in the inline boot
catches exactly this and throws a clear message, which the boot's `try/catch`
renders as a `.kb-note--error`. Keep that guard.

## Localisation

`config.yml` carries a unified `localisation:` block:

```yaml
localisation:
  default-language: en
  languages:
    - { code: en, name: English }
    - { code: de, name: Deutsch }
  en:
    ui:    { key: "English text", … }
    about: { title: …, content: …, button: … }
  de:
    ui:    { key: "Deutscher Text", … }
```

`normaliseConfig()` calls `buildLangTable()` to flatten `localisation[lang].ui`
into a `{ key: { lang: value } }` table; `S('key', vars)` delegates to
`lookupString()`, which falls back `lang → default → key` and interpolates
`{placeholder}` tokens. `pdfOutputLang()` decides whether the PDF follows the UI
language or the config default (`output-language`).

**Rule:** no user-facing English in code. Add a key to every language block and
look it up. Use `{token}` placeholders for values.

## Theme & text size

- **Theme** is a single attribute (`data-theme`) plus the `kb-theme` storage
  key, toggled by `toggleTheme()`. All theme buttons carry `.kb-theme-btn` so
  their icons stay in sync; call `updateThemeIcon()` after a re-render.
- **Text size** is unified onto the `--font-scale` token (the `--fs-*` tokens are
  `calc(... * var(--font-scale))`). The A−/A+ control from `makeSizeControl()`
  clamps to 0.5–1.5 and persists under `kb-font-scale`.

## Modals

One structure: `.kb-overlay` > `.kb-modal` with `.kb-modal__hdr/__body/__footer`,
built by `kbModal()` and its wrappers (`kbConfirm`, `kbAlert`, `kbAbout`), which
return Promises and handle backdrop-click / Escape. About content is rendered
through `markdown()`.

## PDF generation — use pdf-lib

Generate the document with **pdf-lib** (`PDFLib`, loaded from the CDN in
`<head>`). It can both draw a layout from scratch and load/embed/append existing
PDF or image bytes (receipts, signatures, letterheads), so it covers every kind
of document. The user-facing contract is just a Download button that produces a
PDF; the engine is an implementation detail, but standardise on pdf-lib.

## The pixel-perfect checklist

Before shipping, an app is "family-identical" when:

1. `assets/style.css`, `ui.css`, `app.js` are unmodified copies (verify with
   `sync.sh` + `git diff -- assets` showing nothing).
2. The `<head>` order and the pre-paint theme script are intact.
3. The chrome is assembled by the shared helpers (toolbar, header, footer) — no
   bespoke re-implementation.
4. The form is built from `kb-*` components; only true app-specific layout is in
   the inline `<style>`.
5. Every string goes through `S()`; the config validation guard is present.
6. It renders correctly in light + dark and in each configured language.
