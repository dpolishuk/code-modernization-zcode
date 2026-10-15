# Banner usage

The repo banner is [`docs/banner.svg`](banner.svg) — a single self-contained SVG,
1280 × 320 (4:1), dark theme, no external references (fonts, images or scripts),
which is exactly what GitHub's image proxy allows to render.

Design: legacy COBOL card → chevrons → modern TypeScript card with a `PROVEN` seal,
the six core pipeline commands as chips, and the palette the plugin itself uses for
its generated reports.

| Token | Value | Used for |
|---|---|---|
| background | `#1a1b20 → #1e1e1e → #242019` | canvas gradient (matches the report background `#1e1e1e`) |
| accent | `#cc783c` / `#e0975f` | chevrons, `FOR ZCODE` pill, modern keywords (the report accent) |
| text | `#d4d4d4` / `#f2f2f4` | body text / title (the report text color) |
| muted | `#8b8f99` / `#6f7681` | labels, comments |
| success | `#7ec699` | `PROVEN` seal, canary line |
| cards | `#202227` / `#22242a` on `#32353d` strokes | code panels |

Fonts are the system stack only (`-apple-system/Segoe UI` for text,
`ui-monospace/SF Mono/Menlo/Consolas` for code), so it renders correctly on
macOS, Windows and Linux without loading webfonts.

## 1. In this repo's README

Already embedded at the top of [`README.md`](../README.md):

```html
<p align="center">
  <img src="docs/banner.svg" width="880"
       alt="Code Modernization for ZCode — guided legacy-codebase modernization with independent behavioral proof">
</p>
```

- Relative paths (`docs/banner.svg`) work because README and image share the repo.
- `width="880"` keeps it below GitHub's ~1010px content width on all screens; drop
  the attribute to render it at full 1280px on wide screens.
- Always keep a meaningful `alt` — the SVG's internal `<title>` is not read by
  screen readers when embedded via `<img>`.

## 2. In another repo, site, or docs page

Use the `raw` URL (GitHub proxies it through camo, which re-renders SVG safely):

```markdown
![Code Modernization for ZCode](https://raw.githubusercontent.com/dpolishuk/code-modernization-zcode/main/docs/banner.svg)
```

To force external pages to pick up a new version immediately (camo caches
aggressively), pin a commit SHA instead of the branch name:

```text
https://raw.githubusercontent.com/dpolishuk/code-modernization-zcode/<full-commit-sha>/docs/banner.svg
```

## 3. As the GitHub social preview

GitHub's **Settings → General → Social preview** does not accept SVG — use the
pre-exported PNG, [`docs/banner.png`](banner.png) (5120 × 1280, rendered from the
SVG at 4×). Steps:

1. Repo → **Settings** → **General** → scroll to **Social preview** → **Edit**.
2. Upload `docs/banner.png`, then **Save changes**.
3. Expect link-unfurls (Slack, X, Discord…) to show it after their caches expire
   (usually minutes to an hour; some services need a cache-busting query string).

## 4. Re-exporting the PNG after editing the SVG

Any Chromium-based browser renders it exactly as GitHub will (macOS Quick Look
previews SVG at the wrong scale — don't judge the layout by them):

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless --disable-gpu --hide-scrollbars \
  --screenshot=docs/banner.png --window-size=2560,640 --force-device-scale-factor=2 \
  "file://$PWD/docs/banner.svg"
```

(Or open the SVG in Chrome at 100% and screenshot; or `rsvg-convert -w 2560 docs/banner.svg > docs/banner.png` if you have librsvg.)

## 5. Editing rules that keep it working on GitHub

- **No external references.** No `<image href="http…">`, no `@import`, no webfont
  URLs — GitHub's image proxy strips or blocks them, leaving broken output.
- **No scripts.** `<script>` is stripped; interactivity is impossible in `<img>` context.
  SMIL/CSS *animations* do work, if you ever want one.
- **Keep it one file.** Everything (gradients, patterns, fonts-by-name) stays inline.
- **System fonts only** (already the case), so rendering is identical on macOS,
  Windows and Linux.
- After editing, validate the XML (`python3 -c "import xml.dom.minidom,sys; xml.dom.minidom.parse(sys.argv[1])" docs/banner.svg`)
  and re-render a PNG per §4 to check the layout before committing.

## 6. Using it as the marketplace icon (optional)

ZCode marketplace entries accept an `icon` HTTPS URL in `plugins/marketplace.json`.
The banner is 4:1 and would be squashed by square-icon slots — if you want an icon,
make a square variant (chevrons + title + `FOR ZCODE` pill stacked, cards dropped,
`width`/`height` set to 256) and reference that file's raw URL instead.
