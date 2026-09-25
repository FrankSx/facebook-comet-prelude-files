# Route Payload Envelope — Quick Reference

Distilled from [Part 9 — The Route Payload Envelope](https://github.com/FrankSx/facebook-comet-prelude-files/blob/main/09-comet-route-payload-envelope.md). Structure reproduced verbatim from a production `profile.php` route fetch, September 2026.

## Envelope anatomy

```json
{ "payload": { "payloads": { "<request URI>": { ... } },
               "sr_payload": { ... },
               "log_roots": [ ... ] },
  "dtsgToken": null, "dtsgAsyncGetToken": null }
```

- `payloads` — keyed by **request URI** (incl. tracking params); one envelope batches multiple routes.
- `sr_payload` — server resource payload (Haste/CSS deltas).
- Token fields **present but nulled** on the GET route path — CSRF rides in headers (`fb_dtsg_ag`), so body scraping for tokens fails by design.

## route_definition exports (per payload)

| Export | Function |
|---|---|
| `canonicalRouteName` | Stable route ID → client route table |
| `rootView` | `__jsr`/`__dr` resource refs (names, never code) + server-filtered props |
| `entityKeyConfig` | Store-key recipe: components are `constant` or `prop`-sourced |
| `stripParams` | Query params erased from URL after resolution (`modal`, `modal_param`) |
| `tracePolicy` | QPL/Banzai logging namespace |
| `prefetchable` | Router may fetch before navigation |
| `timeSpentConfig` | Time-spent beacon session ID |

## sr_payload.hsrp.hblp

- `consistency.rev` — Haste manifest revision → ClientConsistency soft/hard refresh check.
- `rsrcMap.<id>` — `{ "type": "csr", "src": ":1,2,31", "c": 1 }` — **RLE bitmap**: `c:1` = compressed; `:1,2,31` = chunks {1,2,31} (leading `:` = chunk 0 absent).
- `tieredResources` — `r` regular / `rdfds` / `rds` scheduling tiers.

## jsmods require table

Deferred modules arrive as factories: `emptyFunction@<md5>.thatReturns(["JSResourceReferenceImpl" | "RequireDeferredReference"], …)` — registration is instant, execution waits for mount. `@` suffix = content hash for Haste consistency verification.

## Schema-stability fingerprint

Null slots are deliberate (`dtsgToken: null`, `trackingCode: null`, `meta.title: null`). Present-but-null beats absent. Watch for nulls disappearing — that's the diff signal when the format changes.
