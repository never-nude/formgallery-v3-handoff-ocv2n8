# Form Gallery — Codex brief

This file is the kickoff for Codex. It assumes Codex has read-write access to the `formgallery` repo (or a fork) and a working dev loop for GitHub Pages.

There are three files in `Form Gallery v3/`:

- `formgallery-ui-audit-and-codex-handoff.md` — the audit, with prioritized fixes, current CSS, target CSS, and acceptance criteria.
- `formgallery-modern-mockup.html` — the visual target. A self-contained single-file mockup of the new homepage. Open it in a browser to feel the motion, type, and color.
- `codex-brief.md` — this file. The plan and the rules of engagement.

---

## Paste this into Codex first

> You are picking up an in-progress UI/UX overhaul of formgallery.org — a static digital sculpture museum hosted on GitHub Pages. There is a finished audit and a finished visual mockup waiting for you in the `Form Gallery v3/` folder. Read both, then ship the work as a sequence of small, verifiable pull requests in the order described in `codex-brief.md` § PR sequence. Do not bundle PRs together. After each PR, take a screenshot of the affected pages at 1440 px and 390 px and attach them to the PR description so I can review without running the site. Use the design tokens from `codex-brief.md` § Design tokens for any new CSS — do not invent new colors or fonts. The mockup is the visual target, not a pixel-perfect spec; match its tone, scale, and motion, but use the existing site's content and structure. If you need to make a non-obvious decision (data model change, removing a feature, anything that touches build tooling), open a draft PR and tag me before doing the work.

That's the prompt. Everything below is the reference Codex should consult while working.

---

## Prereqs Codex needs from me before starting

- Read+write access to `github.com/never-nude/formgallery` (or a fork I can pull from). I could not reach the repo from my session — confirm visibility.
- The build/deploy story. The CSS file is versioned with a query string (`?v=20260331-1502`) so I assume there is a generator or a manual bump step. Codex should document whatever it finds in a top-level `DEVELOPING.md` if one doesn't exist.
- Where the work data lives. The homepage clearly iterates over a structured list of works/galleries/eras/regions/makers — Codex should locate the source of truth (a `.json`, a frontmatter set, a directory of files) before touching templates.
- Confirmation on font licensing. The mockup uses Fraunces (OFL) and Inter (OFL) from Google Fonts — both fine to self-host. If there is a different display face the project owns, use that instead and tell me.

---

## PR sequence

Ship in this order. Each PR is independently verifiable and independently revertable.

### PR 1 — Stop the homepage from being 38,000 px tall  (P0)

Scope: the broken `.work-list` grid and the broken `.chronology-group .gallery-grid` collapse. This is the bug that makes the homepage feel broken; fix it before doing anything visual.

Likely files: `museum/shared/museum.css`, the homepage template (probably `museum/index.html` or whatever generates it), the work-card partial.

Required changes:

1. `.work-list` — give it `grid-template-columns: repeat(auto-fill, minmax(min(100%, 220px), 1fr));` and `align-items: start;`. Verbatim target CSS in audit § Change 1.
2. `.gallery-card` on the homepage should become a *summary card* that links to a per-gallery page. The full work list moves to `museum/galleries/<slug>/index.html`. Audit § Change 3 has the HTML/CSS skeleton; do not copy it verbatim, adapt to the project's existing template system.
3. Eager-load above-the-fold thumbnails: featured work + first 6 new-additions cards get `loading="eager" fetchpriority="high"`. Everything else stays `loading="lazy" decoding="async"`. Audit § Change 4.

Acceptance:

- `document.body.scrollHeight` on `/museum/` at 1440 px viewport is ≤ 8,000 px (it is currently 38,637 px).
- Every visible thumbnail above the first scroll renders an image, not a dark gradient placeholder.
- Clicking any gallery summary card on the homepage takes the visitor to a gallery page that contains the same work list that was previously inlined.
- Lighthouse performance score on `/museum/` ≥ 80 (it will get better in PR 4).

Branch: `fix/homepage-collapse`
Commit style: conventional commits, e.g. `fix(homepage): give .work-list an explicit grid template`.

### PR 2 — Site navigation + four browse index pages  (P1)

Scope: the persistent top nav and the four browse landing pages the hero promises (galleries, eras, regions, makers).

Likely files: a new `museum/shared/_nav.html` (or whatever partial system the project uses), `museum/galleries/index.html`, `museum/eras/index.html`, `museum/regions/index.html`, `museum/makers/index.html`, plus CSS additions to `museum.css`.

Required changes:

1. Sticky top nav included on every museum page. Audit § Change 5 has the markup and CSS.
2. Index pages for galleries / eras / regions / makers — each is a simple grid that pulls from the existing data source. No new data model; if the data is currently flat per work, derive these indexes at build time.
3. On the work-detail page, add a breadcrumb (`Atrium / <region> / <work>`) and a Prev / Next pair that walks the current gallery in order. Audit § Change 6.

Acceptance:

- From any page on the site, every primary nav target is reachable in one click.
- `/museum/galleries/`, `/museum/eras/`, `/museum/regions/`, `/museum/makers/` all return 200 with content, not GitHub's default 404.
- Tabbing through the homepage hits the nav before any in-page content; `aria-current="page"` is set on the active link.

Branch: `feat/site-nav-and-browse-pages`

### PR 3 — Rescue the work-detail page  (P1)

Scope: the 3D viewer page is the strongest part of the product but the surrounding UX undersells it.

Likely files: the work-detail template, the viewer's JS module, possibly a small shared component for `<details>` panels.

Required changes:

1. Split the sidebar into **About** (open by default) and two collapsed `<details>` blocks: **Technical details** (triangle count, asset format, source) and **Viewer controls** (the existing sliders and toggles). Audit § Change 6, sub 2.
2. Replace the `BACK TO ATRIUM` button with the breadcrumb from PR 2 plus a Prev / Next pair.
3. Disable wheel-zoom on the 3D canvas until the user clicks into it. Show a subtle "Click to interact" overlay; once clicked, the overlay fades and wheel events are captured. Audit § Change 6, sub 3. This is the single biggest UX win on the detail page — visitors currently scroll-zoom the model by accident.
4. Move the strings "100,000 triangles" and "High-fidelity GLB" out of the default sidebar view; surface them only inside Technical details.

Acceptance:

- A first-time visitor can read the work info without ever touching a slider.
- Scrolling with the cursor over the canvas does not zoom the model unless the canvas has been clicked.
- Keyboard focus order is: breadcrumb → prev/next → about content → technical details summary → viewer controls summary.

Branch: `feat/work-detail-rescue`

### PR 4 — Visual direction + responsive + 404 + metadata  (P2)

Scope: this is where the site starts to look like the mockup. Take it as a single PR because the changes are tightly coupled — type ramp, color tokens, hero treatment, motion contract.

Required changes:

1. Adopt the design tokens in § Design tokens below. Replace the existing color and font usages in `museum.css` with these tokens at the top of the file. Do not break the current dark/cream palette — it's already close.
2. Restyle the Atrium hero to match the mockup's hero: full-bleed dark, massive Fraunces title, gold underline accent that draws on load, counter row, featured sculpture in a cream-lit plinth on the right with cursor-driven 3D parallax and a slow auto-rotation. Reference: mockup `.hero` and the JS at the bottom of the file. Use the existing 3D viewer here if possible — replace the auto-rotation with a real rotating model.
3. Adopt the editorial section heads from the mockup (kicker in tracked caps, large display title with one italic word in gold, lede on the right). Reference: mockup `.section__head`.
4. Convert the chronology rows on the homepage to the marquee + horizontal-rail pattern from the mockup. Drag-to-scroll is non-negotiable on desktop; touch already gets it for free.
5. Custom 404. Audit § Change 7 has the markup; theme it with the new tokens.
6. Root metadata. Audit § Change 8.
7. Responsive breakpoints at 1200, 1024, 768, 480. Audit § Change 9 has the rules but expand them to cover the new sections from this PR. The site has effectively no narrow-viewport CSS today.
8. Honor `prefers-reduced-motion: reduce` everywhere — disable the marquee, hero auto-rotation, scroll-driven reveals, and parallax. The mockup does NOT yet do this; Codex must add it.

Acceptance:

- Side-by-side screenshots of `/museum/` and `formgallery-modern-mockup.html` at 1440 px should read as the same project. They do not need to be identical; type, color, and motion should match.
- The site renders without horizontal scroll and without overlapping chrome at 375, 414, 768, 1024, 1280, and 1440 px.
- Visiting any 404 path renders the themed 404 page.
- A user with `prefers-reduced-motion: reduce` sees no marquee, no parallax, no auto-rotation, and no fade-ins.

Branch: `feat/visual-direction`

### PR 5 — Accessibility and polish  (P3)

Scope: the things that aren't bugs but are necessary for shipping.

Required changes:

1. Add region landmarks and `aria-labelledby` on every section on the homepage. Audit § Change 10.
2. Verify viewer slider labels.
3. Audit color contrast — the dark thumbnail fallback (`#151c28` → `#0f141d`) is invisible on the page background. Pick a fallback that meets WCAG AA against `#08090d`.
4. Add a count-up animation to the hero counter (231 / 13 / 14 / 41) on first scroll into view.
5. Replace any remaining `<article class="gallery-card">` with `<section>` where the content is not actually article-shaped.

Acceptance:

- axe-core / Lighthouse accessibility ≥ 95 on `/museum/`, a gallery page, and a work-detail page.
- No horizontal scroll on any breakpoint, on any page.

Branch: `chore/a11y-and-polish`

---

## Design tokens

Lift these from the mockup. They are the only colors and fonts the project should use.

```css
:root {
  /* color */
  --ink:        #08090d;
  --ink-2:      #10131a;
  --ink-3:      #171b25;
  --cream:      #f4ece1;
  --cream-2:    #e6d9c0;
  --warm:       #c9b896;
  --gold:       #c8a96a;
  --gold-2:     #dbbd7d;
  --rule:       rgba(244, 236, 225, 0.08);
  --rule-2:     rgba(244, 236, 225, 0.14);

  /* type */
  --display:    'Fraunces', Georgia, serif;
  --sans:       'Inter', 'Helvetica Neue', sans-serif;

  /* motion */
  --ease:       cubic-bezier(.2, .8, .2, 1);
  --reveal-y:   50px;
  --reveal-d:   1.2s;
}
```

Type ramp (used in the mockup):

| Role                 | Family     | Size (clamp)              | Weight | Notes                                  |
|----------------------|------------|---------------------------|--------|----------------------------------------|
| Hero title           | Fraunces   | clamp(64px, 11.5vw, 196px) | 300    | `font-variation-settings: "opsz" 144`, line-height .84, letter-spacing -.045em |
| Section title        | Fraunces   | clamp(48px, 7.5vw, 124px)  | 300    | one italic word in `--gold-2`           |
| Editorial H3         | Fraunces   | clamp(40px, 5.4vw, 82px)   | 300    | line-height .92                         |
| Era year             | Fraunces   | clamp(60px, 9vw, 148px)    | 300    | gold, italic optional                   |
| Card title           | Fraunces   | 24–34px                   | 300    | line-height 1.05                        |
| Kicker / metadata    | Inter      | 10–11px                   | 500    | text-transform: uppercase, letter-spacing .22–.32em |
| Body                 | Inter      | 14–16px                   | 300    | line-height 1.6, color rgba(244,236,225,.66) |

Motion contract:

- Scroll progress bar in the top nav, 1px gold, updates on `scroll`.
- Section reveals via IntersectionObserver, threshold .12, translateY 50→0 + opacity 0→1 over 1.2s with `--ease`.
- Cards lift on hover (`translateY(-12px)`) over .8s.
- Hero plinth has both cursor-driven parallax (perspective 1400, ±8deg Y / ±5deg X) and a slow ambient rotation (~.0035 rad/frame) when the cursor is not over it.
- A horizontal rail uses pointer-drag with a 1.2× movement multiplier and shows a "Drag to scroll" hint below it.
- All of the above are gated by `@media (prefers-reduced-motion: no-preference)`.

---

## Definition of done (whole project)

- The homepage at 1440 px is ≤ 8,000 px tall, every above-the-fold thumbnail renders, and the page reads as the same project as `formgallery-modern-mockup.html`.
- Every primary nav target is reachable from every page in one click.
- The work-detail page no longer accidentally zooms when scrolled.
- A visitor on a 390 px viewport can navigate the entire site without horizontal scroll.
- Lighthouse on the homepage: Performance ≥ 90, Accessibility ≥ 95, Best Practices ≥ 95, SEO ≥ 95.
- A custom 404 page exists and is themed.
- A user with `prefers-reduced-motion: reduce` sees a still, dignified version of the site with no animation.
- Every PR has 1440 px and 390 px screenshots in its description.

---

## What Codex should NOT do without asking

- Change the data model (the source-of-truth for works, galleries, eras, regions, makers).
- Add a build dependency that requires Node modules to run (this is GitHub Pages — a one-line shell script or a `_config.yml` is fine, a `package.json` with 200 deps is not).
- Replace the existing 3D viewer with a different 3D library.
- Touch the model files or rescan anything.
- Change content/copy beyond what's in the audit and the mockup. Curatorial voice is the user's call.
- Ship a single mega-PR. The sequence above is the contract.
