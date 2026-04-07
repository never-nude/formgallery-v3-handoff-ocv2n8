# Form Gallery — UI/UX audit and Codex handoff

Prepared for: Mike (never-nude)
Site reviewed: https://formgallery.org (live, hosted on GitHub Pages)
Stylesheet reviewed: `/museum/shared/museum.css?v=20260331-1502`
Date: 2026-04-06

This document is written so that Codex can read it and ship fixes without needing me to narrate again. It has three parts:

1. Findings — what is wrong and why
2. Prioritized fix list — in the order Codex should tackle them
3. Concrete change specs — file targets, selectors, proposed CSS/HTML, and acceptance criteria

I was not able to clone the repo — `github.com/never-nude/formgallery` returned 404 for me, and the local `Form Gallery v3` folder I can see is empty, so every code target below is expressed in terms of the selectors and URL paths that exist on the live site. Codex should map them to the actual source files (most of the issues trace back to `museum/shared/museum.css`, the homepage template under `museum/`, and the work-detail template).

---





### 1.1 The homepage is ~38,000 pixels tall and mostly empty

Measured on a 1720-wide viewport:

- `document.body.scrollHeight` = 38,637 px
- `main.stage` = 38,374 px of that
- `section.rooms-section` = 34,674 px of the stage

Inside `.rooms-section` there are 10 `.chronology-group` sections. Each group contains one `.gallery-grid`, and each `.gallery-grid` contains exactly one `.gallery-card`. The gallery-card is not a card — it is the entire gallery rendered inline as a stacked `<ul class="work-list">` of `<li class="work-item">` rows, and each row is 1,314 × 142 px of near-black gradient with no thumbnail.

Root cause, in two pieces:

**A. `.work-list` is `display: grid` with no `grid-template-columns`.**
Verbatim from `museum/shared/museum.css`:

```css
.work-list {
  margin: 0;
  padding: 0;
  list-style: none;
  display: grid;
  gap: 24px;
}
```

With no explicit track definition, CSS Grid falls back to a single implicit column, which stretches to fill the 1,314 px gallery-card. That is why every "work item" is a full-width horizontal strip.

**B. `.chronology-group .gallery-grid` uses `auto-fit` with only one item per grid.**
Verbatim:

```css
.chronology-group .gallery-grid {
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 300px), 1fr));
  gap: 26px 28px;
}
```

`auto-fit` collapses empty tracks, so when a chronology group contains only one gallery-card, that card is assigned the entire 1,360 px row (computed value: `1360px 0px 0px 0px`). The card is then tall because of (A).

Together, these two issues turn the homepage's room/chronology section into a vertical data dump where every work takes a full-width 142 px slab.

### 1.2 Work thumbnails aren't rendering on the homepage

Inside `.work-item > a.piece`, the computed background is `linear-gradient(#151c28 0%, #0f141d 100%)` and there's no visible `<img>` in the layout. Either:

- the thumbnail `<img>` / `<picture>` is missing from the template, or
- it's being lazy-loaded and never hydrated because it's far down the 38k-pixel page, or
- the image src is wrong and the gradient placeholder is all that renders.

The "New Additions" row just under the featured sculpture has the same issue on revisit — the cards alternate between showing a thumbnail and showing an empty dark frame, which strongly suggests lazy-load + long page + no `loading="eager"` on above-the-fold items.

### 1.3 No persistent navigation anywhere on the site

The hero box tells users:

> Browse the collection by gallery, era, region, or maker.
> 231 works • 13 galleries • 14 regions • 41 makers

But there is no navigation bar, no jump menu, no "Galleries / Eras / Regions / Makers" tabs. Once you scroll past the hero you lose orientation, and on a work-detail page the only way out is a single `BACK TO ATRIUM` button in the viewer sidebar. The four browse axes the hero promises are not reachable as index pages — `/museum/browse/` 404s with GitHub Pages' default page.

### 1.4 The Atrium hero card is visually broken

The `.museum-header` element is 219 px tall and sits 32 px from the top inside a centered column. Because the page background is a dark vignette that extends under the card and the card itself has rounded corners with a dark fill, the top edge of the card is clipped against the viewport edge and it reads as "the header is floating / cropped". It should either extend to the top edge of the viewport or sit on a clearly distinct background with breathing room above it.

### 1.5 Work-detail page: scroll accidentally manipulates the 3D model

On `/sub-saharan-africa/kongo-maternity-figure/`:

- The page has zero scrollable content (`scrollHeight === innerHeight === 1288`).
- The right pane is a 3D viewer; scrolling over it zooms the model. Scrolling anywhere else on the page silently does nothing because there is nowhere to go.
- Users coming from any thumbnail tend to try scrolling down for "more info" and instead accidentally zoom the sculpture. There's no affordance that tells them there is no more content below.

The viewer itself is impressive and probably the strongest part of the product — but the surrounding UX undersells it. Specifically:

- "Viewer Controls" exposes raw numeric parameters (`Zoom 2.55`, `Key Angle 18.00`, `Light Power 2.00`, `Exposure 0.50`, `Roughness 0.20`) and engineering toggles (`Manipulate`, `Multi-Light`, `Wireframe`). For a general audience these read as a debug panel, not a museum.
- "Publication metadata" is labeled "100,000 triangles | High-fidelity GLB". Visitors don't need to know the triangle count and GLB format.
- The `BACK TO ATRIUM` button is mixed in with the viewer hardware controls instead of being part of a page chrome.
- There's no "Next work / Previous work" or "Back to this gallery" navigation, so the detail page is a dead end.

### 1.6 Typography & tone are inconsistent

The museum-quality display face works beautifully on `Atrium`, `Kongo Maternity Figure`, and the chronology titles. But the detail-page sidebar mixes it with a technical/dev tone ("100,000 triangles | High-fidelity GLB", "Drag to rotate. Scroll or pinch to zoom. Shift-drag to pan."). Pick a register and commit to it. A general audience wants provenance, context, and a sense of place, not a README.

### 1.7 Missing or default 404

`/museum/browse/` falls through to GitHub Pages' generic 404 page, which dumps users out of the site chrome entirely. There should be a custom `404.html` at the repo root matching the museum visual language.

### 1.8 No visible top-of-site index / landing

`https://formgallery.org/` resolves straight to `/museum/`. That's fine, but it means the `<title>` and OG metadata for the root URL are "Atrium — Form Gallery", which is not what you want for social shares or Google. A real landing page — or at minimum a proper `<meta>` block on `/museum/` that talks about Form Gallery instead of "Atrium" — would help.

### 1.9 Accessibility smells

- Skip-link exists (`.skip-link` → `#main-content`), good.
- But the rooms-section is a 34k-pixel virtual stack of works with no landmark separation — keyboard users tabbing through will hit hundreds of anchors with no section labels. `.gallery-card` should be a `<section aria-labelledby>` rather than `<article>`, and each chronology group should expose its heading as the region label.
- The viewer "sliders" in the detail page are unlabeled for assistive tech beyond their visible label. Verify they are `<label for>`-associated.
- Color contrast on the dark chrome is not audited here — given how dark the new additions cards render when the image is missing (`#151c28` → `#0f141d` over a near-black page), the thumbnail fallback state is effectively invisible.

### 1.10 Performance / asset smells

- Page shipping 10 chronology sections worth of DOM (hundreds of `.work-item` LIs) on the homepage at once is expensive. At minimum, `loading="lazy"` on thumbnails; ideally, render each chronology as a horizontally scrolling rail of thumbnails and hydrate on scroll.
- The CSS file is versioned with a query string (`?v=20260331-1502`), which is fine for GitHub Pages but means you ship the whole 45 KB stylesheet even for the detail page. Worth measuring.

### 1.11 Mobile / responsive

I could not force the Chrome viewport below 1720 px from this session, so I did not verify mobile visually. Based on reading the CSS:

- The only breakpoint for `.browse-grid` is `min-width: 1500px`. There is no narrow-viewport breakpoint that I can see, which means the layout below ~1000 px is almost certainly untested.
- The detail page's left sidebar is fixed-width; it will overflow on phones.
- The `.new-additions-grid` uses `grid-auto-columns: minmax(240px, 28vw)` with `overflow-x: auto` — that's a good pattern and should work on mobile.

Codex should explicitly test at 375, 414, 768, 1024, 1280, and 1440 px.

---

## 2. Prioritized fix list

Ship in this order. Each item is a single focused change with a clear acceptance criterion.

### P0 — Stop the homepage from being 38,000 pixels tall

1. **Fix `.work-list` to be a real multi-column grid.**
2. **Change the `.chronology-group .gallery-grid` template so a single gallery-card doesn't stretch full width.**
3. **Make sure work thumbnails actually render (eager-load the first N, lazy-load the rest).**

Acceptance: on a 1440-wide viewport, the homepage is ≤ 8,000 px tall, every visible `.work-item` shows a thumbnail, and scrolling from top to bottom feels continuous rather than passing through black voids.

### P1 — Put navigation on the site

4. Add a sticky top bar with "Atrium · Galleries · Eras · Regions · Makers · About" on every page.
5. Build index pages for `/museum/galleries/`, `/museum/eras/`, `/museum/regions/`, `/museum/makers/`. Each is a simple grid of cards that link into existing work pages. The data can be generated at build time from the same source the homepage uses.
6. Add "Previous work / Next work / Up to gallery" controls on the work-detail page.

Acceptance: from any work page, a visitor can reach any of the four browse axes in ≤ 2 clicks, and the hero's promise ("Browse by gallery, era, region, or maker") is fulfilled.

### P1 — Rescue the detail page

7. Split the sidebar into two tabs or collapsible sections: **About** (artist, region, medium, dimensions, collection, description) and **Viewer** (the engineering controls, collapsed by default).
8. Relabel the "publication metadata" block — drop "100,000 triangles | High-fidelity GLB" from the default view; show it under a "Technical details" disclosure.
9. Move `BACK TO ATRIUM` out of the viewer controls and into the page header as a crumb: `Atrium / Sub-Saharan Africa / Kongo Maternity Figure`.
10. Disable wheel-zoom inside the viewer by default; enable it only after the user clicks into the canvas (add a `Click to interact` overlay). This prevents accidental zoom on scroll.

Acceptance: a visitor who has never used a 3D viewer can land on a work page, understand what the work is, and leave without ever touching a slider.

### P2 — Hero, typography, polish

11. Either make the Atrium hero full-bleed (flush to top of viewport, full width, with interior padding) or give it a clear margin above and below so it reads as an intentional card.
12. Remove the "Loading sculpture preview / Loading optimized source model..." stub from the featured work card and replace with a low-res poster image or a shimmering skeleton the same shape as the final media.
13. Commit to one voice — curator/museum, not developer.

### P2 — Accessibility, 404, metadata

14. Ship a custom `404.html` in the museum visual language.
15. Fix the root `<title>` / OG metadata so `/` doesn't say "Atrium".
16. Add landmark roles and region labels to the homepage sections.
17. Verify slider `<label>`s on the viewer are associated.

### P3 — Responsive / perf

18. Add breakpoints at 1024, 768, 480.
19. Eager-load the first 4–6 thumbnails; lazy-load the rest.
20. Consider splitting `museum.css` into a shared base + per-template add-ons.

---

## 3. Concrete change specs (for Codex)

Everything below is written as "change X to Y, then verify Z". Codex should open the corresponding files in the repo, find the selectors/templates mentioned, and apply the change.

### Change 1 — `.work-list` multi-column grid

File: `museum/shared/museum.css` (around line 1963)

Current:

```css
.work-list {
  margin: 0;
  padding: 0;
  list-style: none;
  display: grid;
  gap: 24px;
}
```

Replace with:

```css
.work-list {
  margin: 0;
  padding: 0;
  list-style: none;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(min(100%, 220px), 1fr));
  gap: 24px;
  align-items: start;
}
```

Acceptance: inside a 1,314 px-wide `.gallery-card` on desktop, `.work-list` should render 5 columns with `repeat(auto-fill, minmax(220px, 1fr))`. At 768 px viewport, it should render 3 columns. At 375 px, 1 column.

### Change 2 — `.chronology-group .gallery-grid` shouldn't stretch a single card

File: `museum/shared/museum.css` (line ~1897)

Current:

```css
.chronology-group .gallery-grid {
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 300px), 1fr));
  gap: 26px 28px;
}
```

The problem with `auto-fit` is that a single child expands to fill the row. Either:

**Option A — preferred:** Use `auto-fill` so empty tracks are preserved:

```css
.chronology-group .gallery-grid {
  grid-template-columns: repeat(auto-fill, minmax(min(100%, 320px), 1fr));
  gap: 26px 28px;
  justify-content: start;
}
```

But `.gallery-card` is not meant to be a 320 px tile — it is currently a full gallery section. That ties into Change 3.

**Option B — recommended:** Change what a "gallery card" is. A gallery card on the homepage should be a *summary* card (title, era, region, hero thumbnail, work count, CTA), and the actual work list should live on a dedicated gallery page. If Codex does this, the homepage renders a clean 3–4 column grid of gallery cards and the rooms-section drops to ~2,500 px.

Either option is fine; Option B is the right product decision. Specs for Option B in Change 3.

### Change 3 — Turn `.gallery-card` into a summary card and move work lists to gallery pages

**Data model:** each gallery already has a slug (e.g. `sub-saharan-africa`, `assyrian`, etc). The homepage lists them. Let Codex:

1. Create a page template at `museum/galleries/<slug>/index.html` that renders the full `work-list` for that gallery. The existing `.gallery-card` HTML in `museum/index.html` can be lifted wholesale into this template.
2. Replace the homepage `.gallery-card` with a summary card:

```html
<article class="gallery-card gallery-card--summary">
  <a class="gallery-card__link" href="/museum/galleries/{slug}/">
    <div class="gallery-card__media">
      <img src="{hero}" alt="" loading="lazy" decoding="async" />
    </div>
    <div class="gallery-card__body">
      <p class="gallery-card__kicker">{region} · {era}</p>
      <h3 class="gallery-card__title">{title}</h3>
      <p class="gallery-card__meta">{workCount} works</p>
    </div>
  </a>
</article>
```

3. Add the CSS:

```css
.gallery-card--summary {
  padding: 0;
  border-radius: 16px;
  overflow: hidden;
  background: linear-gradient(180deg, #f2e7d8 0%, #ebdecb 100%);
  border: 1px solid #c3b091;
}
.gallery-card__link {
  display: grid;
  grid-template-rows: auto 1fr;
  color: inherit;
  text-decoration: none;
}
.gallery-card__media {
  aspect-ratio: 4 / 3;
  background: #e6d9c2;
  overflow: hidden;
}
.gallery-card__media img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  transition: transform 400ms ease;
}
.gallery-card__link:hover .gallery-card__media img {
  transform: scale(1.03);
}
.gallery-card__body {
  padding: 18px 20px 22px;
  display: grid;
  gap: 6px;
}
.gallery-card__kicker {
  margin: 0;
  font-size: 10px;
  letter-spacing: 0.22em;
  text-transform: uppercase;
  color: #7b6548;
}
.gallery-card__title {
  margin: 0;
  font-family: var(--font-display);
  font-size: clamp(1.2rem, 1.6vw, 1.6rem);
  line-height: 1.1;
}
.gallery-card__meta {
  margin: 0;
  font-size: 12px;
  color: #6b5a3f;
}
```

Acceptance: the homepage `.rooms-section` drops from ~34,700 px to under ~3,000 px. Clicking a gallery card takes the visitor to a page that renders the old inline work list, unchanged.

### Change 4 — Eager-load above-the-fold thumbnails

For the first 6 thumbnails on the homepage (featured work + new-additions rail), set `loading="eager"` and `fetchpriority="high"`. For everything else, keep `loading="lazy"` and `decoding="async"`.

Acceptance: on first paint at a cold cache, the featured sculpture and the first "new additions" cards show images, not empty dark frames.

### Change 5 — Add a sticky top nav

Create `museum/shared/_nav.html` (partial) and include on every page:

```html
<header class="site-nav" aria-label="Primary">
  <a class="site-nav__brand" href="/museum/">
    <span class="site-nav__brand-kicker">Form</span>
    <span class="site-nav__brand-title">Gallery</span>
  </a>
  <nav class="site-nav__menu">
    <a href="/museum/">Atrium</a>
    <a href="/museum/galleries/">Galleries</a>
    <a href="/museum/eras/">Eras</a>
    <a href="/museum/regions/">Regions</a>
    <a href="/museum/makers/">Makers</a>
    <a href="/museum/about/">About</a>
  </nav>
</header>
```

```css
.site-nav {
  position: sticky;
  top: 0;
  z-index: 50;
  display: flex;
  align-items: center;
  gap: 32px;
  padding: 14px 24px;
  background: rgba(10, 13, 20, 0.82);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.06);
}
.site-nav__brand {
  color: #fff;
  text-decoration: none;
  font-family: var(--font-display);
  letter-spacing: 0.04em;
}
.site-nav__brand-kicker {
  opacity: 0.65;
  margin-right: 4px;
  text-transform: uppercase;
  font-size: 11px;
  letter-spacing: 0.24em;
}
.site-nav__menu {
  display: flex;
  gap: 22px;
  margin-left: auto;
}
.site-nav__menu a {
  color: rgba(255, 255, 255, 0.82);
  text-decoration: none;
  font-size: 13px;
  letter-spacing: 0.12em;
  text-transform: uppercase;
}
.site-nav__menu a:hover,
.site-nav__menu a[aria-current="page"] {
  color: #fff;
}
@media (max-width: 720px) {
  .site-nav { gap: 16px; padding: 12px 16px; }
  .site-nav__menu { display: none; }
  /* ship a mobile drawer — out of scope for P1 */
}
```

Acceptance: the top bar is visible and sticky on `/museum/`, `/museum/galleries/...`, and every work-detail page.

### Change 6 — Work-detail page: split sidebar, add breadcrumb, block accidental viewer zoom

Target: the work-detail template (currently under `/{region}/{slug}/` on the live site, e.g. `sub-saharan-africa/kongo-maternity-figure/`).

1. Wrap the content in a page header:

```html
<header class="work-header">
  <nav class="work-crumbs" aria-label="Breadcrumb">
    <a href="/museum/">Atrium</a> ·
    <a href="/museum/galleries/sub-saharan-africa/">Sub-Saharan Africa</a> ·
    <span aria-current="page">Kongo Maternity Figure</span>
  </nav>
  <div class="work-nav">
    <a class="work-nav__prev" href="{prev}">← Previous</a>
    <a class="work-nav__next" href="{next}">Next →</a>
  </div>
</header>
```

2. Split the sidebar:

```html
<aside class="work-sidebar">
  <section class="work-about">
    <h1>{title}</h1>
    <p class="work-about__artist">{artist}</p>
    <dl>
      <dt>Medium</dt><dd>{medium}</dd>
      <dt>Dimensions</dt><dd>{dimensions}</dd>
      <dt>Collection</dt><dd>{collection}</dd>
    </dl>
    <p class="work-about__description">{description}</p>
  </section>

  <details class="work-technical">
    <summary>Technical details</summary>
    <p>{triangle_count} triangles · {asset_format}</p>
    <p>Source: <a href="{source_url}">{source_name}</a></p>
  </details>

  <details class="work-viewer-controls">
    <summary>Viewer controls</summary>
    <!-- existing sliders / toggles -->
  </details>
</aside>
```

3. In the viewer component (wherever `Zoom / Key Angle / ...` is wired), disable wheel zoom until the canvas is clicked:

```js
let viewerActive = false;
canvas.addEventListener('click', () => { viewerActive = true; canvas.classList.add('is-active'); });
canvas.addEventListener('mouseleave', () => { viewerActive = false; canvas.classList.remove('is-active'); });
// Replace whatever currently calls event.preventDefault() on wheel:
canvas.addEventListener('wheel', (e) => {
  if (!viewerActive) return; // let the page scroll
  e.preventDefault();
  // existing zoom logic
}, { passive: false });
```

And an overlay:

```html
<div class="viewer-overlay" aria-hidden="true">Click to interact</div>
```

```css
.viewer-overlay {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  color: rgba(255,255,255,0.7);
  background: rgba(0,0,0,0.18);
  pointer-events: none;
  font-size: 13px;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  transition: opacity 200ms;
}
.viewer-canvas.is-active + .viewer-overlay { opacity: 0; }
```

Acceptance: scrolling the page with the cursor hovering over the 3D canvas scrolls the page (if the page has scrollable content), never zooms the model. Clicking the canvas activates interaction. The sidebar default state shows title + about, with Technical and Viewer Controls collapsed.

### Change 7 — Custom 404

Create `404.html` at repo root:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Not found — Form Gallery</title>
  <link rel="stylesheet" href="/museum/shared/museum.css">
</head>
<body class="museum museum--dark">
  <!-- include site-nav partial here -->
  <main class="page-404">
    <p class="page-404__kicker">404</p>
    <h1>This plinth is empty.</h1>
    <p>The work you were looking for isn't in this wing of the collection.</p>
    <a class="button" href="/museum/">Return to the Atrium</a>
  </main>
</body>
</html>
```

Acceptance: visiting a bad URL renders the Form Gallery-themed 404, not GitHub's default.

### Change 8 — Root metadata

On `museum/index.html`:

- Keep `<title>Atrium — Form Gallery</title>` for the page itself if you prefer, **but** add `<meta property="og:title" content="Form Gallery — a digital sculpture collection">` and `<meta property="og:description" content="A digital sculpture collection spanning antiquity through the twenty-first century. Browse 231 works across 13 galleries, 14 regions, and 41 makers.">` with an OG image that is the Atrium hero, not a single sculpture.

Acceptance: sharing the root URL on social previews the gallery, not "Atrium".

### Change 9 — Responsive breakpoints

Add to `museum/shared/museum.css`:

```css
@media (max-width: 1200px) {
  .museum-header, .rooms-section, .browse-section, .new-additions-section, .featured-work {
    width: min(100%, calc(100vw - 48px));
  }
}
@media (max-width: 900px) {
  .featured-work { grid-template-columns: 1fr; }
  .featured-work__media { order: -1; }
  .work-layout { grid-template-columns: 1fr; } /* detail page */
  .work-sidebar { position: static; width: 100%; }
}
@media (max-width: 560px) {
  .museum-header { padding: 20px 18px; }
  .site-nav__menu { display: none; }
  .work-list { grid-template-columns: 1fr; }
}
```

Acceptance: at 375, 414, 768, 1024, 1280, and 1440 px the homepage, a gallery index, and a work-detail page all render without horizontal scroll and without overlapping chrome.

### Change 10 — Accessibility pass

- Change `<article class="gallery-card">` to `<section class="gallery-card" aria-labelledby="gallery-{slug}-title">` and make sure the title has the matching id.
- Ensure every `<img>` has `alt` text. Decorative thumbnails can be `alt=""`.
- Ensure the sliders in the viewer controls use `<label for="...">` or `aria-labelledby`.
- Add `aria-current="page"` on the active nav link.

Acceptance: axe-core / Lighthouse accessibility score ≥ 95 on `/museum/` and on a work-detail page.

---

## 4. Items I couldn't verify from the live site

Flagging these so Codex (or you) can check in the repo directly:

- Mobile layout below ~1000 px. My browser session would not let me force a narrow viewport; the CSS has almost no narrow-viewport rules, which suggests mobile is either unbuilt or relying on shrink-to-fit. Needs a real test on device / DevTools.
- Whether the root `/` is served by `museum/index.html` via a redirect or by a separate top-level `index.html`. Worth checking the repo layout.
- The exact JS that wires the 3D viewer and the "Zoom / Key Angle / Light Power / Exposure / Roughness" sliders. Change 6 assumes a plain JS wheel handler.
- The build/templating system (plain HTML, a static site generator, a data file? the homepage seems to be generated from a dataset of works because the chronology groups are pre-ordered).
- The repo itself — `github.com/never-nude/formgallery` returned 404 for me. If it's private, grant Codex access or confirm the repo name.

---

## 5. Quick reference: what the bug actually is, in one paragraph

The homepage renders every gallery inline as a full-width block because (a) `.work-list` is a CSS grid with no `grid-template-columns`, so every work item collapses into a single implicit 1,314 px column, and (b) each chronology group only contains one `.gallery-card`, and that card lives inside a `.gallery-grid` that uses `repeat(auto-fit, minmax(min(100%, 300px), 1fr))`, which causes the single card to stretch across the entire 1,360 px row. Those two facts together turn the homepage into a 38,000-pixel vertical stack where each "gallery card" is really an entire gallery, and because thumbnail `<img>`s aren't eager-loaded, most of it renders as empty dark strips. The fix is to give `.work-list` an explicit `repeat(auto-fill, minmax(220px, 1fr))` template, convert the homepage `.gallery-card` into a summary card that links out to a dedicated gallery page, and eager-load the first handful of thumbnails.
