# lawn — Quartz Migration Design Roadmap

> Meta-doc for the pivot from Quarto to a Quartz-based blog stack.
> lawn.dawnfire.casa = main blog (Quartz fork)
> garden.dawnfire.casa = future Quarto sideblog for executable-code posts (Tidy Tuesday etc.)

---

## Stack Overview

| Concern | Choice | Notes |
|---|---|---|
| SSG | Base Quartz 4 (fork) | Not TurnTrout's repo — clean foundation, cherry-pick his plugins |
| Writing env | Obsidian | Quartz speaks wikilinks + Obsidian callout syntax natively |
| Font | Recursive (self-hosted woff2) | Variable font, one file, full axis control |
| Colors | Catppuccin Mocha (dark) / Latte (light) | CSS custom properties, not Bootstrap SCSS |
| Comments | Giscus | GitHub Discussions backend, Quartz built-in support |
| ATProto | Sequoia CLI | Publishes Standard.site lexicon records to PDS on deploy |
| Hosting | K3s (existing) | Traefik + cert-manager + Longhorn, same as current setup |

### Why base Quartz, not TurnTrout's fork

TurnTrout's repo has ~6000 commits of accumulated customization for *his* specific choices: EB Garamond line-heights baked into dozens of CSS values, 3660 TypeScript tests for his content behavior, his color system threaded through every component. Reskinning it means archaeology before writing a line of your own code. Base Quartz is ~150 commits of clean, extensible foundation. His plugins (`favicons.ts`, `spoiler.ts`, etc.) are largely self-contained files — you copy them into your codebase, wire them into the transformer pipeline, done. Adding his functionality to your codebase is much easier than carving your design out of his.

---

## Recursive Font Axis Hierarchy

All five axes controlled via CSS custom properties to avoid inheritance issues.

```css
/* Base text */
--MONO: 0.51; --CASL: 0; --CRSV: 1; font-weight: 350;
/* MONO 0.51 = pseudo-proportional (visual warmth without full fixed-width)
   CRSV 1 = single-story a/g upright as default letterform */

/* Italic / em */
--MONO: 0.51; --CASL: 1; --slnt: -13;
/* Casual + slanted, CRSV auto (will apply cursive alternates at this slant) */

/* Bold / strong */
--MONO: 0.51; --CASL: 1; --CRSV: 1; font-weight: 500;

/* Headings */
--MONO: 0.51; --CASL: 0.5; font-weight: 600;

/* Code blocks */
--MONO: 1; --CASL: 1; --CRSV: 0; font-weight: 400;
/* Full monospace, casual, roman alternates (not cursive) */

/* Marginalia / footnotes / captions */
--MONO: 0.51; --CASL: 0; --CRSV: 0; font-weight: 300;
/* Lighter, roman alternates, legible at small sizes */
```

---

## Build Phases

### Phase 1: Base Quartz + Fonts + Colors

**Goal:** Something rendering locally with correct typography and color system before any custom features.

Subtasks:
- [ ] `npx create-quartz@latest` scaffold
- [ ] Copy `Recursive.woff2` into `quartz/static/fonts/`
- [ ] Write `quartz/styles/recursive.css` — `@font-face` declaration + CSS custom property system for all five axes
- [ ] Write `quartz/styles/catppuccin.css` — Mocha and Latte palette as CSS custom properties (`--ctp-base`, `--ctp-mauve`, etc.)
- [ ] Write `quartz/styles/custom.scss` — wire font and color variables into Quartz's theme system, override base component styles
- [ ] Configure `quartz.config.ts` — site title, base URL, basic plugin pipeline
- [ ] Configure `quartz.layout.ts` — sidebar layout, basic component placement
- [ ] Get `quartz build --serve` rendering locally with correct fonts + dark/light toggle
- [ ] Port `_quarto.yml` navbar structure to Quartz equivalent

**Column width calibration:** Measure Recursive proportional at your chosen base size (aim for 65–75 characters per line in the content column). TurnTrout uses 750px / ~75 chars with EB Garamond — Recursive's character metrics are different, so measure and set explicitly rather than copying his pixel value.

---

### Phase 2: Marginalia

**Goal:** True Tufte-style sidenotes. This is load-bearing layout work — do it before adding any other layout-affecting features.

**How it works:** Two pieces working together:
1. **CSS layout** — the content column is a CSS grid with a main text track and a right margin track. `.sidenote` elements use `float: right` with a large negative right margin to pull them into the gutter. On narrow viewports they collapse to inline with a small toggle label.
2. **Transformer** — a Quartz plugin that walks the HTML AST, finds footnote definitions (currently collected at the bottom), and re-emits them as `<span class="sidenote">` elements placed adjacent to their `<sup>` reference points in the document flow rather than at the end.

**Why it has to come first:** The content column grid with a margin track affects the width available to everything — body text, code blocks, images, callouts. If you set column width first assuming no margin track, you'll have to redo it.

Subtasks:
- [ ] Design content column CSS grid: `[text] 1fr [margin] 240px` (adjust margin width to taste)
- [ ] Set main column to ~65-75 chars wide after accounting for margin track
- [ ] Write `.sidenote` CSS — float, negative margin, `font-size: 0.8em`, Recursive marginalia axis settings
- [ ] Write narrow-viewport collapse CSS — sidenotes become inline blocks with `[note]` toggle
- [ ] Write `quartz/plugins/transformers/sidenotes.ts` — AST transformer that moves footnote nodes
- [ ] Wire transformer into `quartz.config.ts` plugin pipeline
- [ ] Test with a post that has both `^[inline margin notes]` and `[^traditional footnotes]`

**Inline syntax:** `^[text]` produces a sidenote. `[^1]` traditional footnotes still work but go to page bottom. Quartz's Obsidian syntax transformer already handles `^[text]` as inline footnotes — the sidenote transformer just intercepts before they're collected at the bottom.

---

### Phase 3: TurnTrout Plugin Cherry-picks

Each plugin is a self-contained file (or small set of files) from his `quartz/plugins/transformers/` directory. They're TypeScript and slot into Quartz's transformer pipeline with minimal modification.

#### `favicons.ts` + `countFavicons.ts`
**What it does:** Fetches favicons for external links and injects them inline before the link text, with no-break logic so the favicon never wraps away from the first word of the link text.  
**Why it's worth it:** Immediately communicates "this goes to GitHub / this goes to a paper / this goes to a blog" without the user having to hover or click.  
**Subtasks:**
- [ ] Copy `favicons.ts`, `countFavicons.ts`, related test files
- [ ] Add favicon fetch caching (he has this — check his implementation)
- [ ] Test that favicon + `(arch.)` superscript don't create visual jumble (they're different semantic registers — should be fine, but verify)

#### `spoiler.ts`
**What it does:** Lines starting with `>!` become hidden spoiler blocks revealed on click.  
**Why it's worth it:** ~50 lines, trivially small, immediately useful for media posts / discussion of plot-relevant things.  
**Subtasks:**
- [ ] Copy `spoiler.ts`
- [ ] Write CSS for the spoiler reveal animation (he uses a blur transition)
- [ ] Test on mobile

#### `formatting_improvement_html.ts` + `punctilio`
**What it does:** Post-processing pass over rendered HTML to apply smart typography: proper em/en dashes, non-breaking spaces before short words, smallcaps for acronyms (NASA, API, etc.), upright punctuation in italic contexts. `punctilio` is his extracted npm package that does the actual transforms.  
**Why it's worth it:** Works across HTML element boundaries (something no other library does correctly), and smallcaps acronyms are a genuine readability improvement at body text sizes.  
**Subtasks:**
- [ ] `npm install punctilio` 
- [ ] Copy `formatting_improvement_html.ts`
- [ ] Verify it doesn't conflict with Quartz's existing Pandoc-style smart quote processing (check if base Quartz does smart quotes — may need to disable one)
- [ ] Check smallcaps rendering with Recursive: `font-variant: small-caps` or explicit `font-size: 0.85em; text-transform: uppercase`

#### Hover preview extensions
**What it does:** Base Quartz has internal link popover previews. TurnTrout extends these with better positioning, scroll behavior, and rendering.  
**Subtasks:**
- [ ] Get base Quartz hover preview working first
- [ ] Review his `scripts/` directory for the relevant extensions
- [ ] Assess whether his extensions are needed or if base Quartz's is sufficient

---

### Phase 4: Custom Features

#### Dropcaps
**Design:** CASL=1, wght=800-900, CRSV=1, two line-heights tall, `--ctp-base` or `--ctp-crust` background box (no border), randomized Catppuccin accent color on page load applied to the letter color.

**Implementation:**
- CSS: `.dropcap::first-letter` or explicit `<span class="dropcap">` on first paragraph (explicit span is more reliable across SSGs)
- JS: small inline script on page load picks from accent array, sets `--dropcap-color` custom property
- Quartz transformer: auto-wraps first letter of first paragraph in posts (not pages, not index) with `.dropcap` span

```js
// Page load accent randomizer
const accents = ['--ctp-mauve','--ctp-blue','--ctp-green','--ctp-peach',
                 '--ctp-pink','--ctp-teal','--ctp-sky','--ctp-yellow'];
document.documentElement.style.setProperty(
  '--dropcap-color', 
  `var(${accents[Math.floor(Math.random() * accents.length)]})`
);
```

Subtasks:
- [ ] Write dropcap CSS with Recursive axis settings
- [ ] Write transformer to inject `.dropcap` span on first letter of post body
- [ ] Write page-load JS snippet
- [ ] Test at both CASL extremes to confirm wght=800-900 renders correctly
- [ ] Check two-line-height sizing across viewport widths

#### Archive Links
**Design:** External links get a `<sup><a class="archive-link" href="{wayback_url}">(arch.)</a></sup>` appended, styled in `--ctp-subtext0`. On by default, suppressible per-post with `archive-links: false` in frontmatter.

**Implementation:**
- Build-time transformer (not client-side) — runs during `quartz build`
- Calls Wayback Availability API: `https://archive.org/wayback/available?url={url}`
- Cache file: `quartz/archive-cache.json` mapping `url → {snapshot_url, checked_at}` — committed to repo, checked before API calls, updated after fetches
- HEAD request to live URL alongside Wayback check: 4xx/5xx gets a `⚠` indicator alongside `(arch.)`
- archive.today: generate `https://archive.ph/newest/{url}` as fallback without verification (API-hostile, can't confirm)
- Skip: URLs that are already archive links (wayback, archive.ph, archive.org, perma.cc)
- Skip: same-domain links, anchor links, mailto links

**Cache structure:**
```json
{
  "https://example.com/some-post": {
    "snapshot_url": "https://web.archive.org/web/20240315120000/https://example.com/some-post",
    "live_status": 200,
    "checked_at": "2026-04-22"
  }
}
```

**Build performance:** With cache, only new/uncached links fetch. First build of link-roundup posts will be slow (450+ API calls); subsequent builds near-instant for unchanged posts.

Subtasks:
- [ ] Write `quartz/plugins/transformers/archiveLinks.ts`
- [ ] Implement Wayback Availability API fetch with error handling
- [ ] Implement cache read/write
- [ ] Implement HEAD request for live link health
- [ ] Write `(arch.)` superscript CSS in `subtext0`
- [ ] Add `archive-links` frontmatter flag support
- [ ] Test with tab-clearing draft posts (real link density stress test)
- [ ] Consider: archive.today fallback URL generation (no verification)

---

### Phase 5: ATProto + Comments

#### Giscus
Quartz has native `Component.Comments` support for Giscus. 

Subtasks:
- [ ] Enable GitHub Discussions on the lawn repo
- [ ] Install Giscus app on repo
- [ ] Get `repoId` and `categoryId` from giscus.app
- [ ] Add to `quartz.layout.ts` `afterBody`
- [ ] Write custom Giscus CSS theme matching Catppuccin (he documents how to do this — `themeUrl` pointing to CSS files in `quartz/static/giscus/`)

#### Sequoia
Subtasks:
- [ ] `npm install -g sequoia` (or however it's invoked)
- [ ] `sequoia init` in repo — authenticates with PDS, creates `site.standard.publication` record
- [ ] Wire `sequoia publish` into deploy script after `quartz build`
- [ ] Verify Standard.site `<link>` verification tags appear in built HTML

---

## Content Migration

### From Quarto drafts
The `.qmd` files are plain Markdown with YAML frontmatter — they'll port cleanly. The only Quarto-specific syntax to find-and-replace:
- `^[text]` inline footnotes → same syntax, works natively in Quartz ✓  
- `::: {.callout-note}` → Quartz uses `> [!note]` Obsidian callout syntax
- `::: {.column-margin}` → becomes `^[text]` inline margin note syntax
- `$math$` / `$$math$$` → Quartz supports KaTeX, same syntax ✓

### Tab-clearing roundup posts (3 drafts)
These are the best first content to publish — link-dense, good stress test for archive-links feature, and a "tabs clearing" post is a natural "here's what the blog is about" signal to early readers.

---

## Infrastructure Notes

- lawn.dawnfire.casa: Quartz site, Docker/nginx on K3s (same pattern as existing Quarto setup)
- garden.dawnfire.casa: new subdomain, Quarto Docker/nginx on K3s, only spin up when actually writing executable-code posts
- Existing Terraform for K3s workloads, existing Traefik + cert-manager — no infrastructure changes needed for Quartz vs Quarto, same containerized nginx pattern
- Cloudflare tunnel: add `garden.dawnfire.casa` ingress rule alongside `lawn.dawnfire.casa` when garden actually exists

---

## Features Not Carried Over From Quarto Setup

- **Remark42**: simplified out in favor of Giscus — zero self-hosted comment infrastructure, automatic email notifications, GitHub OAuth trust
- **Sketchy theme**: moving to clean CSS, Recursive's CASL axis carries the personality
- **Doto/JetBrains Mono/Space Mono stack**: consolidated to Recursive (one font file, all semantic roles)
- **Bootstrap SCSS**: Quartz doesn't use Bootstrap — direct CSS custom properties

---

## Reference

- Quartz docs: https://quartz.jzhao.xyz
- TurnTrout's repo (plugin reference): https://github.com/alexander-turner/TurnTrout.com
- TurnTrout's design post: https://turntrout.com/design
- Recursive font: https://www.recursive.design
- Wayback Availability API: https://archive.org/help/wayback_api.php
- Sequoia / Standard.site: https://sequoia.pub
- Catppuccin: https://github.com/catppuccin/catppuccin
