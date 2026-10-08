# pilah-docs

Personal engineering documentation for the PILAH 2.0 project (PPL, Fasilkom UI).

**Live:** <https://twentism.github.io/pilah-docs/>

Built with [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).
Pushing to `main` rebuilds and publishes automatically.

---

## Changing the colours

Open **`docs/stylesheets/extra.css`**. The first block is the only part you need
to touch:

```css
[data-md-color-scheme="pilah"] {
  --pilah-bg:     #f7f9f8;   /* page background */
  --pilah-brand:  #2e7d5b;   /* header, links, accents */
  ...
}
```

Change a value, save, reload. Every colour on the site comes from that block, so
you never have to hunt through theme internals. Hex codes, `rgb()` and colour
names all work.

The variables below the `====` line wire those colours into the theme — you can
ignore them.

## Changing the structure

`mkdocs.yml`:

- **`nav:`** — the sidebar and tabs. Add a page by creating the `.md` file under
  `docs/` and adding a line here.
- **`theme.features`** — layout behaviour. Remove `navigation.tabs` if you would
  rather have everything in the sidebar.
- **`site_name`**, **`site_description`** — the title and the search blurb.

## Writing pages

Plain Markdown, plus a few extras that are already enabled:

```markdown
!!! note "Optional title"
    A callout box. Also: tip, warning, danger, important, failure, info.

??? note "Click to expand"
    A collapsible version.

| Table | Works |
|-------|-------|
| yes   | yes   |
```

Status chips used in the contribution tables:

```html
<span class="chip chip-merged">merged</span>
<span class="chip chip-open">open</span>
<span class="chip chip-review">in review</span>
```

Those three classes are defined at the bottom of `extra.css`.

## Previewing locally

```bash
pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>. It live-reloads as you edit.

To check exactly what CI will check:

```bash
mkdocs build --strict
```

`--strict` turns warnings into errors, so a dead internal link fails the build
rather than reaching the site.

## Editing from the browser

Every page has a pencil icon in the top right that opens that file in GitHub's
editor. Committing to `main` triggers a redeploy — useful for a quick typo fix
without cloning.
