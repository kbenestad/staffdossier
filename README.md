# basis

A self-contained template for building a **standalone kbenestad
business-document app** — one that produces a PDF in the browser and looks and
behaves **exactly** like the apps in the
[bizdocs](https://github.com/kbenestad/bizdocs) family, but ships on its own:
its own repo, its own static hosting, **no build step, no backend**.

Copy this repository, replace the marked app-specific parts, and you have a new
app that is pixel-identical to the family.

## What's here

```
index.html          the app — chrome + boot + form + PDF, all inline
config.yml          branding + every UI string (localisation)
assets/
  style.css         design tokens / colour scheme        ┐ shared bizdocs design
  ui.css            the kb-* UI component library         │ + runtime — bundled
  app.js            shared runtime (DOM, theme, i18n, …)  ┘ copies; DO NOT EDIT
  favicon.svg       placeholder icon — replace per app
  site.webmanifest
sync.sh             pull the three shared files from bizdocs
CLAUDE.md           how to build an app from this template (read this first)
DESIGN.md           the design system + the pixel-perfect rules
```

## Quick start

```bash
# serve it (the app fetches config.yml, so file:// won't work)
python3 -m http.server 8000   # then open http://localhost:8000/
```

Then follow **[CLAUDE.md](CLAUDE.md)**: rename/rebrand, build your form from
`kb-*` components, build your PDF with pdf-lib, fill in `config.yml`, and drop in
a real icon set.

## Staying pixel-perfect

`assets/style.css`, `assets/ui.css` and `assets/app.js` are **byte-identical
copies** of bizdocs' shared design + runtime — that's what keeps this app looking
like the family. **Never hand-edit them.** Pull bizdocs UI updates wholesale:

```bash
BIZDOCS_REF=main ./sync.sh --from-github   # from GitHub, no checkout needed
./sync.sh ../bizdocs/assets                # or from a local bizdocs checkout
git diff -- assets                         # review, then commit
```

App-specific CSS goes in `index.html`'s inline `<style>`; app-specific text goes
through `config.yml` + `S()`. See [DESIGN.md](DESIGN.md) for the full design
system and the pixel-perfect checklist.
