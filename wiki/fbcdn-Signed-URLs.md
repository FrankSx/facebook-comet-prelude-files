# fbcdn Signed URLs — Quick Reference

Distilled from [Part 8 — fbcdn Signed URL Internals](https://github.com/FrankSx/facebook-comet-prelude-files/blob/main/08-fbcdn-signed-url-internals.md). All evidence from live captures, September 2026.

## Signed URL parameters

| Param | Role |
|---|---|
| `_nc_ht` | **Host-claim token, bound into the signature.** Exact string must match the serving hostname. |
| `_nc_cat` | Content-category bucket (cache partition routing). |
| `_nc_ohc` | Object revision hash ("oh content"). |
| `sdl` | `0` = inline render. |
| `ccb` | Cookie/cache epoch (`14-4`) — global signed-URL invalidation lever. |
| `oh` | The HMAC signature. |
| `oe` | Expiry, hex epoch. 7-day TTL observed. |
| `_nc_sid` | Edge shard/region routing key. |

## Four-state edge table

| State | Trigger | Status | Body | `server:` header |
|---|---|---|---|---|
| Success | GET + valid signature | 200 | object | absent — Everstore tier |
| Signature mismatch | wrong `_nc_ht`/`oh`/`_nc_ohc` | 403 | 22 B `URL signature mismatch` | `proxygen-bolt` |
| Bad method | POST anywhere | 400 | 363 B generic "4xx Client Error" HTML | absent |
| Apex | GET `/` | 404-class | anonymous shell | absent |

## Tier fingerprints

- **200:** `x-everstore-unified-metadata: 1`, `x-needle-checksum` / `content-digest: adler32=…`, `x-fb-edge-debug`, `cache-control: max-age=1209600, no-transform` (14 days)
- **403:** `proxy-status: http_request_error; e_proxy=…`, `x-fb-connection-quality: …rtt=…tbw=…`, session-stable `x-fb-ptm-uuid`
- **400:** `proxy-status: proxy_internal_response; e_proxy=…; e_fb_twtaskhandle=…` — no telemetry, no server header

## Probing rules

1. Only GET exercises the signer. POST answers a different tier.
2. Vary exactly one parameter per request; `_nc_ht` mutations are the canonical probe.
3. No rate limiting observed on rapid mismatches; turnaround 60–200 ms (stateless signer).
4. Replay model: signature TTL (7d) < cache TTL (14d) — expired URLs re-validate at the signer, stale cache cannot serve them.

## Relation to application-layer signing

`fbcdn.net`/`fbsbx.com` are excluded from the `shouldAppendToken` allowlist (Part 2) — media URLs carry their own signature and must never get `fb_dtsg` appended (it would break the HMAC).
