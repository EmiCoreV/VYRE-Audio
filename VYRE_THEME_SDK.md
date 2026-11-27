# VYRE Audio 2.1.0 — Community Theme SDK

This guide describes the desktop host's current theme API, not standalone HTML skins. A theme rearranges and styles real host components. It cannot replace the application, add executable UI or change DSP, plugin behavior, official branding text, version or copyright. Changes apply to the host UI; the app icon is independent of the selected theme.

## Installation

Open **App Settings → Appearance → Open theme folder**, create a direct child folder and put `theme.json` there. The directory is `userData/themes` (normally `~/Library/Application Support/VYRE Audio/themes` on macOS; on Windows use the folder opened by the app). Choose **Refresh themes**, then select the theme in Appearance. Changes require another refresh. The selected theme persists; an invalid or removed theme falls back to Aurora. Neon Mirage is the fresh-install default, while Aurora remains the technical fallback. Do not distribute `node_modules` or require a local server.

## Minimal manifest and stylesheet

`theme.json` (all fields shown here are required):

```json
{
  "format": "vyre-theme",
  "apiVersion": 1,
  "id": "my-theme",
  "name": "My Theme",
  "version": "1.0.0",
  "author": "Example Author",
  "description": "A restrained community theme.",
  "palette": {
    "background": "#10141d",
    "surface": "#1a202a",
    "ink": "#f5efdf",
    "muted": "#b6ad9b",
    "accent": "#e3c07d",
    "secondary": "#b482d6",
    "line": "#766b60"
  },
  "layout": {
    "navigation": "vertical",
    "overviewOrder": ["hero", "summary", "progress"],
    "corners": "round",
    "typeface": "system"
  }
}
```

Optional `style.css`:

```css
.nav { border-radius: 10px; color: #f5efdf; }
.nav:hover { background-color: #2d3745; }
.metric { border-color: #766b60; }
@media (max-width: 1100px) { .metric { padding: 12px; } }
```

IDs are lowercase ASCII letters, digits and hyphens, 1–48 characters and start with a letter or digit. Version must be `major.minor.patch` with an optional prerelease. Name, author and version are nonempty and at most 80 characters; description at most 300; control characters and unknown manifest fields are rejected. Seven palette values must be six-digit hex colors. `ink` must contrast against both `surface` and `background` by at least 3:1; additional live checks apply. `layout` can be supplied in `layout.json` instead of inline.

The complete accepted top-level field set is `format`, `apiVersion`, `id`, `name`, `version`, `author`, `description`, `palette`, `layout`, `fonts`. The `layout` keys are `navigation`, `overviewOrder`, `corners`, `typeface`, `sidebar`, `cardOrder`, `settingsOrder`, `importPosition`, `regions`, `reorders`, `moves`, `brandingPosition`, `brandingLayout`, `containers`, `navigationGroups`, `pages`. Unknown keys are rejected. `navigation` is `vertical` or `horizontal`; `corners` is `round` or `square`; `typeface` is `system`, `serif`, `mono` or `custom`; `overviewOrder` is a permutation of `hero`, `summary`, `progress`. `sidebar` is `left`/`right`; `importPosition` is `normal`/`first`; `cardOrder` is a permutation of `receiver`, `measurement`, `target`, `microphone`. `settingsOrder` still parses as a permutation of `measurement`, `target`, `microphone`, `advanced` but does not change the host's fixed calibration-settings order. Defaults and richer complete examples are in `examples/Obsidian Wave` and `examples/Ion Atlas` in the source distribution.

## Assets and fonts

Optional folder files are `style.css`, `layout.js`, `assets/` and `fonts/`. Manifest, layout, program, CSS, image and font files are each bounded to 64 MiB (67,108,864 bytes). A standalone declarative `<id>.vyre-theme.json` is also recognized; custom fonts require a folder. At most 128 themes and 3 GiB of encoded theme catalog data are accepted; each theme has a 1536 MiB encoded budget. CSS is also limited to 67,108,864 JavaScript characters. Local asset paths must have the form `assets/name.png`, `.jpg`, `.jpeg` or `.svg` with a simple ASCII filename; no nested asset paths, symlinks or external URLs. There are at most 64 image references and 384 MiB of resolved CSS resources, including embedded image data; the compiled CSS has the same 384 MiB character limit. SVG is restricted to validated static geometric markup; scripts, external references and arbitrary SVG are rejected. Up to eight WOFF2 fonts can be declared, each at most 64 MiB and at most 768 MiB encoded total (the WOFF2 declared uncompressed size is limited to 512 MiB), with distinct weights 100–900 in steps of 100:

```json
"fonts": [
  {"file":"fonts/regular.woff2","weight":400},
  {"file":"fonts/bold.woff2","weight":700}
]
```

`layout.typeface: "custom"` uses `VyreCommunityFont`. When `typeface` is `custom` and `fonts` is omitted, `fonts/theme.woff2` is required as the weight-400 fallback. Supply appropriate redistribution licenses for any assets and fonts. Symlinks, malformed font data and path traversal fail validation. Validation is not a guarantee against image/font decoder vulnerabilities.

## Components, pages and layout

The component registry exposes existing DOM IDs and `data-vyre-component` identifiers. Dynamic controls use semantic keys where available; anonymous IDs may change when host structure changes, and ambiguous identities cannot be moved individually. Inspect the `api.components` snapshot in a layout program. Core navigation views are `overview`, `calibration`, `target`, `system`, `run`, `export`; Final Curves remains reachable through the native calibration flow. Useful host IDs include `view-overview`, `view-calibration`, `view-target`, `view-system`, `view-run`, `view-export`, `advancedOptions`, `crossoverRangeList`, `peqWorkbench`, `peqImportButton`, and `vyreWorkflowTab-*`. Plugin settings and other dynamic components are registered as they appear; do not assume an unkeyed anonymous ID remains stable.

`layout.navigationGroups` groups and relabels the six actual navigation buttons. Each view must occur exactly once. Alternatively, `layout.pages` combines complete existing views; each of the six views must appear once, and each page's first view supplies the real navigation button. These features cannot be combined. Other buttons are hidden only when their entire views remain accessible on a grouped page; arbitrary hiding or duplicate fake buttons is forbidden.

`layout.regions` can arrange the required slots in `row`, `column` and `grid` containers. Region → required slots:

- `overview`: `header`, `hero`, `session-title`, `cards`, `progress`.
- `overview-controls`: `tag`, `headline`, `description`, `import`, `new-project`, `input`, `error`.
- `target`: `header`, `modes`, `status`, `studio`.
- `system`: `header`, `overview`; `results`: `header`, `empty`, `curves`.
- `run`: `header`, `progress`; `export`: `header`, `actions`, `summary`, `polarity`.

Each required slot must occur exactly once. Trees allow at most 80 nodes, depth eight, 20 children per container, grid columns 1–4 and gap 0–40 px. Up to 128 containers, 512 moves and 32 reorder lists are accepted; each reorder list contains 2–20 unique IDs. A legacy `calibration` region may be accepted in the manifest but is not applied: settings use the fixed host order. `layout.containers` creates empty host-approved containers, `layout.moves` moves real registered nodes, and `layout.reorders` reorders siblings. Example layout fragment:

```json
{
  "containers": [{"id":"theme-system-tools","into":"view-system","direction":"column"}],
  "moves": [{"id":"peqImportButton","into":"theme-system-tools","optional":true}]
}
```

The host restores moved nodes on theme changes and rebinds eligible dynamic replacements. Moves are rejected when they break implicit labels, forms, modal boundaries, event handlers or control reachability; moving a whole functional group is safer. Existing handlers stay host-owned; component IDs and action metadata do not grant an API to invoke host actions. Some event mediation is deliberately conservative and not a full emulation of the original DOM event path.

Official brand, version and copyright are protected host nodes. Approved `brandingLayout` moves the whole brand group plus the existing version and copyright; it cannot rewrite them. Titlebar and window controls stay protected. `brandingPosition` may be `top` or `bottom`. Alternatively, `brandingLayout` has `{into,direction,order}`: `direction` is `row`/`column`; `order` is a permutation of `brand`, `version`, `copyright`; `into` is `rail-top`, `rail-bottom`, `topbar` or an empty theme container directly in `branding-rail`/`branding-topbar`. It cannot be combined with `brandingPosition`. The host checks visibility, overlap and clipping, and restores original nodes on a theme change.

## Sandboxed layout.js

`layout.js` is a JavaScript **function body**, not an ES module. It receives `api.layout` and `api.components` and returns a layout object. Example, preserving the validated initial layout:

```js
return { ...api.layout };
```

Execution occurs in a separate sandboxed renderer without Node, filesystem, network, Electron bridge or host DOM. The returned layout is validated. It cannot create a website, iframe, custom HTML, new labeled controls, hidden copies or executable host actions. CSS and layout are applied to the actual host UI.

## CSS and floating/dock layouts

CSS is parsed with Chromium CSSOM (at most 1000 rules and eight levels of nesting) and scoped to the app, settings, plugin and report surfaces. IDs, classes and `[data-vyre-component]` selectors, supported pseudo-classes, grid/flex, CSS custom properties, media/container queries, gradients, shadows and approved local asset URLs are supported. Supported WebKit scrollbar/range/progress pseudo-elements are subject to the same validation. CSS escapes, external URLs, `@import`, `@font-face`, `@namespace`, `@property`, `@layer`, `attr()`, generated text and changes to protected branding are rejected. `display:none`, hidden controls, unsafe clipping, filters, masks, blend/3D effects and other concealment are not a supported way to make a dock. The host also checks actual readability, size, visibility and hit targets and can fall back to Aurora even when CSS parses.

A macOS-inspired dock can restyle and reposition the **existing** navigation; use horizontal navigation, approved containers, spacing, colors and static hover colors. `position:fixed` is limited to `.navigation` or a host-created `theme-floating-…` container. Fixed layers must remain wholly below the titlebar and inside the viewport, cover at most 40% of the window and not obscure required controls; transformed, filtered or layout/paint-contained ancestors are prohibited. `z-index` is 0–10 and offsets are `auto`, zero, 0–512 px or 0–100%. The host rechecks on scrolling, resizing and state changes. There is no replacement native macOS Dock, arbitrary overlay, second navigation or permission to hide original controls. `Ion Atlas` illustrates fixed real navigation.

Hover styles cannot compromise reachability. Animation and transition assignments are limited to approved non-interactive areas without interactive ancestors or descendants; a macOS-style magnification animation on actual navigation buttons is therefore not a guaranteed supported operation. A theme cannot manufacture tooltip text with CSS `content`, inject markup or script, or create new tooltip controls. Existing host labels/tooltips remain available. CSS cannot hide original navigation merely to replace it with a decorative copy.

## Motion limits

Allowed 2D transforms: `translate`, `translateX/Y`, `scale`, `scaleX/Y`, `rotate`, or their individual properties; maximum eight functions, total translation at most 100 px or 20%, each scale and cumulative axis scale 0.5–3, total rotation at most 360 degrees. No variables in sensitive transform values, matrices, perspective or 3D. `transform-origin` permits up to two direction words, zero, or 0–100 px/%.

Keyframes must be named `theme-…`: at most 32 animations, 32 frames each and 128 total. Allowed frame properties are transforms, opacity (at least 0.75), `background-color`, `border-color`, `box-shadow`, `outline-color`, `fill`, `stroke`. Up to four animations per list, duration and absolute delay at most 120 seconds, iterations at most 100 or `infinite`; supported timing is standard ease/linear, bounded cubic Bézier, or `steps` up to 60. Transitions support transform, opacity, color, background/border/outline color, box shadow, fill and stroke, not `all` or layout properties. Motion values cannot use variables or `!important`. The host namespaces keyframes and disables theme motion under reduced-motion preferences. Runtime visibility checks still apply throughout motion.

## Troubleshooting

- **Unsupported theme format/API/ID**: use `vyre-theme`, API 1 and a unique lowercase ID.
- **Insufficient text contrast**: change `ink`, `surface` and `background` to reach the required 3:1 minimum; also check controls and actual backgrounds.
- **Invalid layout/region**: include each required slot exactly once; do not mix `pages` with `navigationGroups`; avoid rearranging host-owned calibration settings.
- **External or unsupported CSS resources**: remove remote URLs, CSS escapes, imports, unsupported SVG, or excessive assets; use validated local files.
- **Theme hides/covers a required control**: restore visibility and hit targets, reserve space for fixed navigation, and check small windows and open dialogs.
- **Invalid move or ambiguous identity**: use the current component snapshot, move a complete labeled group, and retain native handler/form boundaries.
- **Invalid animation/transform**: use a `theme-` keyframe name, explicit bounded 2D values and approved properties; do not animate interactive controls.

These checks protect supported host behavior but do not prove every theme state safe. Test your own theme across viewports, keyboard use and reduced motion before distributing it.
