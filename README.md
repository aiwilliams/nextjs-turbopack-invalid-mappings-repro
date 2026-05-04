# Turbopack invalid VLQ mappings — minimal reproduction

Reproduces a Turbopack source-map bug in `next dev` where the indexed/sectioned
map for chunks containing pre-minified vendor sources (e.g. `simplepeer.min.js`)
contains VLQ-encoded column deltas with magnitudes around 2³². Firefox rejects
the map with `Source Map "..." has invalid "mappings"`.

See [`ISSUE.md`](./ISSUE.md) for the full report (template-formatted for
`vercel/next.js`).

## Run

```bash
pnpm install
pnpm dev
```

Then open `http://localhost:3099/` in **Firefox** and watch the DevTools
console.

## Verifying the bad VLQ values without a browser

After visiting the page once so Turbopack compiles the chunk:

```bash
# locate the chunk that contains simplepeer.min.js
grep -laE "simplepeer\.min\.js" .next/dev/static/chunks/*.js

# decode the corresponding .map and look at section column deltas
python3 - <<'PY'
import json, re
m = json.load(open('<the .map you found>'))
for s in m['sections']:
    src = s['map'].get('sources', [])
    if any('simplepeer' in x for x in src):
        toks = re.split('[,;]', s['map']['mappings'])
        big = [t for t in toks if len(t) > 6]
        print(f'oversized VLQ tokens: {len(big)} / {len(toks)}')
        print('samples:', big[:3])
PY
```

`next build` (with `productionBrowserSourceMaps: true`) is **not** affected —
the bug is specific to the dev-server's sectioned source-map encoder path.
