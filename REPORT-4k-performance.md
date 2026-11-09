# ZenzaWatch 4K Performance Fix — Final Report

Branch: `playlist-deploy` (all changes uncommitted working-tree edits)

## 1. Root Cause of 4K Slowness

The comment layer was built for a fixed **384px-tall virtual comment area** and then scaled up to the real video height via CSS `transform: scale()` on `#commentLayerOuter`:

```
scale = targetHeight / 384
```

At 1080p that is ×2.8; at 4K it is **×5.625**. On top of the CSS scale, the actual rasterized surface is multiplied by `devicePixelRatio`:

- A comment is rasterized at its DOM size (roughly 320×32px) and then composited at `CSS scale × DPR` (at 4K ≈ 11.25×).
- Each `.nicoChat` has `will-change: transform` (required for per-comment compositor animation — removal was tested previously and caused flicker), so every comment holds a **persistent GPU texture** at raster resolution: ≈ **1.2 MiB each at 4K vs ~94 KiB at 1080p**.
- With the up-to-40-comment cap, that is **~48 MiB of comment textures** plus ~300 MiB of backing-store churn across the scaled compositor surface every frame — consistent with the reported 24–34ms per-frame latency at 4K.

Additionally, full-viewport persistent compositor layers made it worse:

- `.videoPlayer`, `.commentLayerFrame`, `.resizeObserver` in `src/NicoVideoPlayerDialog.js` each had `will-change: transform, opacity` while covering the whole viewport → at 4K+DPR each is a persistent full-screen texture (**~31.6 MiB each**, ~95 MiB total) even when idle.
- `.chatControlBoxInner` (`src/NicoChatControlBox.js`) had `will-change: transform,opacity`, but it is an 854×56 element (~2.2 MiB), so it was left untouched.

## 2. What Changed and Why

### `packages/zenza/src/commentLayer/NicoCommentCss3PlayerView.js`
- `onResize()` now computes `effectiveScale = cssScale × win.devicePixelRatio` and toggles class `is-largeScale` on `#commentLayerOuter` when `effectiveScale >= LARGE_SCALE_THRESHOLD` (5). The threshold triggers at ~1536p@DPR2, 4K@DPR1, 2K@DPR3 — i.e. whenever the raster multiplier gets large, regardless of which factor contributes.
- New CSS: `#commentLayerOuter.is-largeScale .nicoChat { will-change: auto !important; }`. The `!important` is required because `will-change` is set inside the `.is-animatable … .nicoChat` descendant rule that carries the comment-animation setup. `will-change: auto` is the initial value the app itself uses for non-animated comments (`.is-ready` branch), so it is a proven-safe state; comments still animate through keyframes, just without a persistent per-comment texture — the layer is promoted only during animation instead of always.
- `_getMaxDisplayComment()` reduces the simultaneous-comment cap from **40 → 24** when `is-largeScale` is set (mirroring the existing `_isSmallScreen` precedent of adapting the cap to viewport). ~19 MiB of GPU textures saved at 4K.

### `src/NicoVideoPlayerDialog.js`
- Removed `will-change: transform, opacity` from `.commentLayerFrame` and `.resizeObserver` (they are purely positional wrappers — no transitions/animations) and from the visible-state rule of `.videoPlayer`.
- Safety check: `updateLayout()` switches visibility with `style.display` only; no CSS opacity transitions exist for the dialog, so removing the persistent promotion cannot alter fade behavior.

### `dist/ZenzaWatch-dev.user.js`
- Rebuilt with `npm run build:monkey` so the userscript is in sync with the sources.

## 3. Verification Performed

| Check | Result |
|---|---|
| `ClassList.toggle(name, force)` two-arg form | `ClassListWrapper` returns a native `DOMTokenList` (`Element.js:411`); supports the force flag ✔ |
| `#commentLayerOuter` exists in HTML (non-SVG) rendering mode | `__TPL__` template contains the div; the `<g>` variant only appears in the SVG-export string ✔ |
| CSS selector `#commentLayerOuter.is-largeScale .nicoChat` resolves | Confirmed against the compiled stylesheet, including the `!important` override ✔ |
| `devicePixelRatio` source | Uses `win.devicePixelRatio` from the same iframe window already used for `win.innerWidth`, keeping scale/DPR consistent ✔ |
| `onResize()` ordering | Called from constructor after class fields are initialized (`_lastMaxDisplayComment = 0` exists before the listener fires); all call sites safe ✔ |
| Syntax | `node --check` passes for all 3 changed files ✔ |
| Build | `npm run build:monkey` succeeds; dist contains `is-largeScale`/`LARGE_SCALE_THRESHOLD`/DPR logic ✔ |
| Test suite | `npm test` fails only on a **pre-existing** issue: `NicoScripterTest.js` imports `../../src/NicoScripter`, which does not exist in HEAD either (unrelated to this change) |
| Lint | ESLint fails parsing class properties in `NicoVideoPlayerDialog.js` — pre-existing config limitation (`ecmaVersion: 2019` vs modern syntax), not caused by the change |

Estimated effect at 4K: **~67 MiB** less persistent GPU memory (48→29 MiB comment textures + ~95 MiB viewport layers removed), and far less per-frame texture-raster churn on the scaled comment layer.

## 4. Not Verified (needs a real browser)

1. **Live frame-time measurement at 4K** — no browser/GPU environment here; the numbers above are computed from layer sizes, not measured FPS. Needs on-device profiling (DevTools Performance / GPU memory).
2. **Visual check of the `will-change: auto` comment mode** — the pattern is used elsewhere in the app and is considered safe, but 4K comment smoothness (especially `colorShade`/animation-heavy comments) should be eyeballed.
3. **DPR change between monitors** — if the window moves between displays with different DPR, `onResize()` only fires on size change; a pure DPR change without a size change would not re-evaluate `is-largeScale` until the next resize.
4. **Interaction with the pre-existing small-screen cap** — on a 4K window smaller than 500px wide (rare), the small-screen cap (20) wins over the large-scale cap (24); behavior is intentional but untested.

## Files Touched

- `packages/zenza/src/commentLayer/NicoCommentCss3PlayerView.js` — large-scale detection + will-change removal + comment cap
- `src/NicoVideoPlayerDialog.js` — removed full-viewport `will-change` layers
- `dist/ZenzaWatch-dev.user.js` — rebuilt bundle
