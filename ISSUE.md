# Title

Turbopack emits invalid VLQ column deltas in source maps for already-minified vendor sources (Firefox rejects with `has invalid "mappings"`)

---

## Link to the code that reproduces this issue

<!--
TODO: push the contents of /tmp/nextjs/vlq to a public GitHub repo and paste the URL here.
The repo only needs:
  - package.json
  - tsconfig.json
  - next.config.js
  - app/layout.tsx
  - app/page.tsx
The dependency that triggers the bug is `simple-peer@9.11.1`, but the same bug occurs for any pre-minified file under node_modules — see "Additional context" for a second example (`react-jsx-dev-runtime.development.js`) that shows the same symptom.
-->

`<paste public repro URL here>`

## To Reproduce

1. `pnpm install` (uses `next@16.3.0-canary.9` and `simple-peer@9.11.1`)
2. `pnpm dev` (runs `next dev --turbopack -p 3099`)
3. Open `http://localhost:3099/` in **Firefox**
4. Open the browser DevTools console

The page is a single `"use client"` component that does:

```tsx
import SimplePeer from "simple-peer/simplepeer.min.js";
export default function Page() {
  return <div>SimplePeer: {typeof SimplePeer}</div>;
}
```

## Current vs. Expected behavior

### Current

Firefox prints in the console:

```
Source Map "http://localhost:3099/_next/static/chunks/node_modules__pnpm_<hash>._.js.map" has invalid "mappings"
```

The map fetches successfully (HTTP 200, valid JSON, `version: 3`, indexed format with `sections`). The problem is the contents of the `mappings` string inside the section that targets `simplepeer.min.js`. Several VLQ-encoded segments contain column deltas in the **billions**, which the Firefox source-map decoder rejects.

Concrete numbers from the generated `.map`:

- The section for `.../simple-peer/simplepeer.min.js` contains 26,036 VLQ tokens; **656** of them are longer than 6 base64 characters (i.e. encode values that don't fit in 30 bits).
- Two representative tokens, decoded:
  - `IAKw/67//H` → `(genCol=+4, srcIdx=+0, srcLine=+5, srcCol=+4,294,899,192)`
  - `AAL5367//H` → `(genCol=+0, srcIdx=+0, srcLine=-5, srcCol=-4,294,899,068)`
- The magnitudes (~4.295 × 10⁹) are within ~68 KB of `2^32`, which strongly suggests a **32-bit signed/unsigned wraparound** in the encoder — small negative deltas reinterpreted as huge positives (and vice versa).
- The mapped file (`simplepeer.min.js`) is ~50 KB on a single line, so a real `srcCol` cannot exceed ~50,000. The delta values are not just unusual — they are **structurally impossible**.

### Expected

VLQ column deltas in the source map should reflect actual source positions and stay within the size of the original line. Firefox should not reject the map, and the map should be usable for symbolicating frames in the minified vendor source.

## Provide environment information

```bash
Operating System:
  Platform: linux
  Arch: x64
  Version: #19-Ubuntu SMP PREEMPT_DYNAMIC Fri Mar  6 14:02:58 UTC 2026
  Available memory (MB): 62553
  Available CPU cores: 32
Binaries:
  Node: 22.22.0
  npm: 10.9.4
  Yarn: N/A
  pnpm: 10.29.3
Relevant Packages:
  next: 16.3.0-canary.9 // Latest available version is detected (16.3.0-canary.9).
  eslint-config-next: N/A
  react: 19.0.0
  react-dom: 19.0.0
  typescript: 6.0.3
Next.js Config:
  output: N/A
```

Also reproduced on the stable release `next@16.2.4` with identical token signatures.

## Which area(s) are affected? (Select all that apply)

- Turbopack

## Which stage(s) are affected? (Select all that apply)

- next dev (local)

`next build` (with `productionBrowserSourceMaps: true`) is **not** affected: the production map for the same chunk is a flat (non-sectioned) source map, all VLQs decode to sane values, no column deltas anywhere near 2³², and no negative cumulative line/column or out-of-range name indices. So the bug appears specific to the indexed/sectioned source-map path used by `next dev --turbopack`.

## Additional context

**The bug is not specific to `simple-peer`.** Any pre-minified file under `node_modules` produces the same symptom. In the same repro, the source map for the chunk that contains `simplepeer.min.js` is an indexed (`sections`-style) map with three sections; **section 0** maps `next/dist/compiled/react/cjs/react-jsx-dev-runtime.development.js` and contains **96** oversized VLQ tokens with the same near-2³² magnitudes. So the bug surfaces wherever Turbopack source-maps a long-line / already-minified input, not just when a user imports `simplepeer.min.js` directly.

**Browsers:** Chrome appears to tolerate the bad VLQ values silently; Firefox surfaces the warning. Either way, the encoded data is invalid against the source-map v3 spec semantics (column deltas cannot reasonably exceed source line length).

**Likely root cause:** the magnitudes of all the bad tokens cluster within a few KB of `2^32`, with the corresponding small offsets visible as `2^32 − n` (for small `n`). That is the fingerprint of a 32-bit signed↔unsigned conversion bug in the VLQ encoder, where small negative `srcCol` deltas (when the encoder steps backward to an earlier column on a long line) get reinterpreted as huge positive `u32` values before being VLQ-encoded.

The fact that `next build` produces a clean flat source map for the *same* input file — no billion-magnitude deltas, no negative cumulative columns — supports this: the dev-only indexed/sectioned encoder path is where the wraparound is happening, not the underlying line/column tracking.

**Related but distinct existing issues** (none cover this exact symptom):
- #77670 / #73384 — Firefox source-map warnings under Turbopack on Windows, but those are `Invalid URL: file://C%3A/...` (Windows path encoding), a different decoder error.
- #53253 — "Inaccurate sourcemaps for sources under node_modules" — long-running issue about low-fidelity (column-0) maps for vendor code; symptom is *poor* mappings, not *invalid* ones.
- PR #76627 — overhauled the indexed/sectioned source-map format Turbopack emits (which is the format used here). The bad VLQ data appears inside that format's per-section `mappings`.
