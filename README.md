# R&D: Inline DOM SVG `feColorMatrix` Icon Tinting Filter

## Background

Our white-label CSS theming system (PW/ST2 client config CSS) recolors a shared
black/grayscale sprite-sheet icon set per client brand using a CSS `filter` chain:

```css
--icon-hover: brightness(0) saturate(100%) invert(19%) sepia(16%) saturate(5295%) hue-rotate(217deg) brightness(91%) contrast(103%);
```

```css
.sprite-icon { filter: var(--icon-hover); }
```

This lets one sprite sheet serve every client theme without maintaining separate colored assets —
the browser recolors black pixels to the brand hex live, driven entirely by a CSS custom property.

## The bug that triggered this R&D

A client's brand-primary color (`#282A73`, a deep saturated indigo) rendered correctly on
iOS 18 Safari but visibly shifted toward **purple** on iOS 26 Safari, using the identical filter
chain/value.

**Root cause:** reaching a deep, highly-saturated color (especially blue/indigo — low luma weight
in the underlying color matrices) from pure black requires an extreme `saturate()` boost
(1000–7000%+). At that magnitude, tiny per-engine differences in how Safari computes the
`invert()/sepia()/saturate()/hue-rotate()` chain (color-space handling, rounding, wide-gamut
display compositing, etc.) get massively amplified into a visibly different hue. Re-solving the
filter chain for better numeric accuracy did **not** fix it — the instability is structural, not a
tuning error.

**Short-term fix applied for the affected client:** recolor the sprite's native pixel value to `#282A73`
directly, so the hover state needs `filter: none` (pixel-perfect on every engine, no math at all),
and the other state (green) is solved from that new base color instead of from black, which also
lowers its required `saturate()` dramatically. This works, but it's a per-client workaround, not a
structural fix for the whole template system.

## What this R&D is for

Find a **structural fix** that keeps the current integration pattern (one shared black/grayscale
sprite sheet + a `filter: var(--icon-x)` CSS custom property per client theme) but eliminates the
saturate-amplification failure mode entirely, for any brand color, on any client, without needing
a per-color risk assessment every time a new client theme is onboarded.

### Leading candidate: inline DOM SVG `feColorMatrix` "constant fill" filter

Instead of an `invert/sepia/saturate/hue-rotate` chain, reference an SVG filter that forces a
**constant output color** regardless of input pixel value (only the alpha/shape channel is
preserved):

```html
<!-- somewhere in the live DOM, e.g. a shared layout partial — NOT a data: URI -->
<svg width="0" height="0" style="position:absolute">
  <filter id="icon-hover-fix" color-interpolation-filters="sRGB">
    <feColorMatrix type="matrix" values="
      0 0 0 0 0.156863
      0 0 0 0 0.164706
      0 0 0 0 0.450980
      0 0 0 1 0" />
  </filter>
</svg>
```

```css
--icon-hover: url(#icon-hover-fix);
```

Because the output RGB is hard-coded (no multiplicative saturate/hue-rotate math), there is nothing
for a rendering engine to compute inconsistently — it should be pixel-identical across every
browser and OS version.

### Known constraint (already discovered, must design around it)

We first tried this same `feColorMatrix` technique referenced via a `data:image/svg+xml,...` **data
URI** directly inside the CSS custom property (no separate DOM element). That version **silently
failed on Safari/iOS** — the filter did not apply at all (icons rendered as the raw, unfiltered
sprite). Safari/WebKit has a known limitation where `filter: url(#id)` only reliably works when the
referenced `<filter>` lives in an inline `<svg>` already present in the page's live DOM — not when
it's loaded from a `data:` URI or an external file.

**Implication:** the filter's `<svg><filter>...</filter></svg>` markup must be injected into the
actual page HTML (e.g. once per page/template, via a shared include/partial), not embedded inside
the CSS file via a data URI. This is the main architectural question for this R&D — where/how to
inject that shared SVG snippet across the template system without duplicating it per client.

## Questions to answer in this R&D

1. Does the inline-DOM version of `feColorMatrix` actually render correctly and consistently across
   Safari/iOS (all supported versions), Chrome, Firefox, and Edge? Confirm with real device/browser
   testing, not just emulation.
2. Where should the shared inline `<svg>` filter defs live in the template system so they're
   injected once per page without duplicating per-component or per-client? (e.g. a single shared
   layout partial included on every page.)
3. Can the `feColorMatrix` `values` matrix be generated per-client the same way the current
   `invert/sepia/...` chain is (i.e., computed from a brand hex), so onboarding a new client theme
   is still a simple "plug in the hex" step?
4. Do sprite icons that rely on partial alpha (anti-aliased edges) still look correct with a
   constant-fill matrix, or does edge quality differ from the current filter-chain approach?
5. Is there a performance difference (SVG filter vs. CSS filter chain) worth caring about given the
   icons are small and the filter is applied to many elements (tab bars, sidebars, etc.)?
6. Does this approach work when the same sprite sheet is used as a `background-image` sprite (not an
   `<img>`), which is our current setup?
7. Fallback behavior: what renders if a browser doesn't support SVG filters referenced this way at
   all (very old browsers)? Compare gracefulness vs. the current filter-chain approach's fallback.

## Current candidate and implementation handoff (pending testing)

The reusable branding candidate is inline SVG `feFlood` + `feComposite`, with colors
supplied by CSS custom properties. The hard-coded `feColorMatrix` is a comparison control,
not the proposed client-branding integration.

**Status:** real iOS 26 testing is still pending. Do not treat this as a verified global fix
or deploy it to production yet. The R&D does not modify the production main stylesheet.

After testing, prepare the exact template and CSS changes for the frontend team:

- Shared template: one inline SVG definitions block per document, with `icon-brand-default`
   and `icon-brand-hover` filters. Iframe documents using the filters need their own definitions.
- Shared CSS: `.icon-default-fill` and `.icon-hover-fill` rules using `flood-color` with
   `--icon-default-hex` and `--icon-hover-hex`; retain the non-displayed SVG sizing/position rules.
- Per-client CSS: define both hex variables on `:root`, set `--icon-color` to
   `url(#icon-brand-default)` and `--icon-hover` to `url(#icon-brand-hover)`.
- Keep existing icon markup, sprite assets, background positions, and default/hover selectors.
   The production filters do not require JavaScript or matrix-value generation.
- Include device/browser results and any integration limitations with the final handoff.

## Reference: current (working) per-client CSS file for comparison

The client-specific stylesheet and its local path are intentionally omitted from this public project.

## Reference: filter-chain solver script (zero npm dependencies, pure Node.js)

Used during the original bug investigation to compute/compare `invert/sepia/saturate/hue-rotate`
values and `saturate()` magnitude across candidate source colors (reimplementation of the public
"sosuke" CSS filter generator SPSA algorithm). Not included here — can be recreated from scratch in
the new R&D chat/workspace if needed for generating `feColorMatrix` values per target hex (the
constant-fill matrix is much simpler: it's just `target_r/255`, `target_g/255`, `target_b/255` as
the constant terms, no optimization/solver needed).
