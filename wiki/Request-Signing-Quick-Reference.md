# Request Signing — Quick Reference

Full analysis: [Part 2](https://github.com/FrankSx/facebook-comet-prelude-files/blob/main/02-async-request-anatomy.md).

## Tokens

| Param | Method | Source |
|---|---|---|
| `fb_dtsg` | POST | `DTSG` module (`cr:8959`), rotatable mid-session |
| `fb_dtsg_ag` | GET | separate provider (`cr:8960`) |
| `lsd` | both | pre-auth fallback token |
| `jazoest` | both | `SprinkleConfig.version + sum(charCodes(token))` |

## Core params

`__user` · `__a=1` · `__req` (incrementing) · `__rev` (client revision) · `__s` (WebSession `xxxxxx:xxxxxx:n`) · `hsi` · dynamic keys via `StaticSiteData` (haste session, dpr, ccg, jsmod) · Haste bitmaps · `qpl_active_flow_ids`

## Token-attachment boundary

Attached only to: facebook.com, instagram.com, messenger.com, meta.com, workplace.com, oculus.com (+ subdomains), freebasics.com, discoverapp.com. **Excluded:** CDN URIs, all `fbsbx.com`.

## Responses

* Prefixed `for (;;);` (regex `/^for ?\(;;\);/`) — strip before parsing.
* `__rev` mismatch → ClientConsistency actions: 2 = softRefresh, 3 = hardRefresh.

## WebSession

6-char zero-padded base-36 ID (Alea PRNG, ServerNonce seed), stored in localStorage key `Session` as `id:expiry`.
