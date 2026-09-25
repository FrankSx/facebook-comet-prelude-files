# Assets — Reassembly Instructions

The four art files in this directory are stored as base64 text split into
`NN`-numbered parts (8,000 characters each) because of the tool channel's
content-length ceiling. Reassemble any asset with:

```sh
cat <asset-name>.b64.part?? | base64 -d > <asset-name>
```

For example:

```sh
cat post09-banner-route-envelope.jpg.b64.part?? | base64 -d > post09-banner-route-envelope.jpg
```

## SHA-256 checksums (verify after reassembly)

| File | Size (bytes) | SHA-256 |
|---|---|---|
| `post08-banner-cdn-edge.jpg` | 51,197 | `b65af47f96850b139153b1368f62c0834fca9ead425cfd47fb8381c1eae7f02b` |
| `post08-diagram-four-state-edge.png` | 18,706 | `6a80915ae29a0a3deae482aff688a086dc248f368226c2ccce0a90b66d159628` |
| `post09-banner-route-envelope.jpg` | 48,817 | `92aa7ca5d0531bf82d779d986aa5e8971163dec0e9a33958d96cf6d296f59a77` |
| `post09-diagram-route-envelope-pipeline.png` | 21,648 | `74313f819f5d437d44f736d7fd91aa905dabbb89278a73efb90dbd932601755a` |

Verify on macOS/Linux:

```sh
shasum -a 256 post09-banner-route-envelope.jpg
# or
sha256sum post09-banner-route-envelope.jpg
```

## Part inventory

- `post08-banner-cdn-edge.jpg.b64.part01` … `part09` (9 parts)
- `post08-diagram-four-state-edge.png.b64.part01` … `part04` (4 parts)
- `post09-banner-route-envelope.jpg.b64.part01` … `part09` (9 parts)
- `post09-diagram-route-envelope-pipeline.png.b64.part01` … `part04` (4 parts)

Each part file contains plain base64 text with no headers or footers — the
parts concatenate directly. The final part of each set carries the remainder
and is shorter than 8,000 characters.

## What the art depicts

- **post08 banner + four-state edge diagram** — visual pipeline of an
  `xx.fbcdn.net` signed-URL request through Proxygen: `_nc_ht` host-claim
  HMAC, `_nc_cat` / `_nc_ohc` object revision, `oh` signature, `oe`
  hex-epoch expiry (7-day TTL), and the four edge response states
  (200 Everstore / 403 proxygen-bolt / 400 anonymous HTML / apex 404).
- **post09 banner + route-envelope pipeline diagram** — anatomy of the Comet
  route payload envelope: `{payload:{payloads, sr_payload, log_roots}}`,
  `dtsgToken:null`, `canonicalRouteName`, the `csr:_9w_0_bq` RLE Haste
  bitmap, deferred requires, and ServerJS dispatch.

See Parts 8 and 9 of the blog series for the full write-ups.
