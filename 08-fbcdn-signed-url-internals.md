# fbcdn Signed URL Internals: `_nc_*` Parameters, Everstore, and the Proxygen Edge

*Part 8 of The Facebook Comet Prelude Files · frankSx · frankhacks.blogspot.com*

**Labels:** reverse engineering · CDN internals · URL signing · HMAC · Proxygen · Everstore · fbcdn · security research
**Permalink:** `/2026/09/fbcdn-signed-url-internals-nc-parameters.html`
**Search description (110 chars):** Reverse engineering Facebook's fbcdn signed image URLs: `_nc_*` HMAC parameters, Proxygen 403/400 error tiers, Everstore headers.

---

Parts 2 and 3 of this series documented how the Comet *client* signs its requests (`fb_dtsg`, `fb_dtsg_ag`, jazoest, `lsd`). But every image, video, and static asset Facebook serves travels under a **second, independent signature scheme** — one that lives entirely at Meta's CDN edge and never touches the application layer. Meta does not document it. This post reverse-engineers it from live traffic: two packet captures, one deliberately induced signature failure, and the response headers that leak the storage stack underneath.

All evidence below comes from captures taken September 23, 2026 against `scontent.xx.fbcdn.net`, cross-referenced with the prelude bundle (SHA-256 `57d60f40f636669c259ec6f945ae4264ac4b55cba01a5e767affaad900e69ec4`) analyzed in Parts 1–7.

![Banner: circuit-board padlock guarding glowing server gates — the fbcdn signed-URL edge](assets/post08-banner-cdn-edge.jpg)

## The four-state edge, at a glance

![Flowchart: the scontent.xx.fbcdn.net edge decision pipeline — POST gate, object-path gate, HMAC verification, and the Everstore/Needle success tier, with per-tier header fingerprints](assets/post08-diagram-four-state-edge.png)

## 1. Anatomy of a signed media URL

A Facebook image URL is not a path — it is a **signed capability**:

```
https://scontent.xx.fbcdn.net/v/t58.90637-1/822114087_1728385414904548_6228392286548835306_n.webp
    ?_nc_ht=scontent.xx.fbcdn.net
    &_nc_cat=104
    &_nc_ohc=j73yBykFlPcQ7kNvwHttUSA
    &sdl=0
    &ccb=14-4
    &oh=00_AQJMzkevC_fzjzreXHNkGdrYn3EKqmm-fI6bLbnKmS1FeQ
    &oe=6AB5EDCF
    &_nc_sid=a21977
```

Parameter by parameter, from what the edge enforces:

| Parameter | Role | Evidence |
|---|---|---|
| `_nc_ht` | **Host-claim token, bound into the signature.** Must exactly equal the serving hostname — a single missing dot fails verification (§3). | 403 `URL signature mismatch` when mismatched |
| `_nc_cat` | Content-category bucket (integer, e.g. `102`, `104`). Category of the owning object; likely feeds cache-partition routing. | constant across retries in capture |
| `_nc_ohc` | Object hash/checksum component — "ohc" = *oh content*. Rotates with the stored object revision; survives the object's lifetime. | unchanged across all 5 requests in capture 1 |
| `sdl` | Download flag (`0` = inline render, not forced download). | — |
| `ccb` | Cookie-consent/cache-busting epoch, format `14-4`. Bumped when Meta wants to globally invalidate cached signed URLs. | — |
| `oh` | **The signature itself** (opaque, `00_AQ…` base64url-ish). Verified against the exact tuple of path + `_nc_ht` + object revision. | 403 when `_nc_ht` tampered |
| `oe` | **Expiry**, hex epoch. `6AB5EDCF` = 2026-09-30 — a 7-day TTL on a URL minted ~2026-09-23. | decodes directly |
| `_nc_sid` | Edge session/shard routing key (`a21977` region/shard selector). | — |

The critical insight from capture 1: **the same `oh` signature validates only when `_nc_ht` carries the exact serving hostname.** One missing dot — `scontent.xxfbcdn.net` — fails. So the HMAC input covers, at minimum, the path, the host-claim parameter, and the object revision (`_nc_ohc`). The edge is verifying not *where the request goes* but *what the signer claimed about where it would go* — a classic anti-rebinding binding.

## 2. The four-state edge

Capture 1 recorded five requests against one signed object, varying only `_nc_ht` and method. Capture 2 recorded one request to the bare apex. Together they give the complete decision table of the `scontent` edge:

```
                       ┌────────────────────────────────────────────┐
   request ──────────► │  Proxygen edge (server: proxygen-bolt)     │
                       └──────────────┬─────────────────────────────┘
                                      │
                 method == POST? ──yes─┼──► 400 "4xx Client Error"
                    │ no               │    proxy_internal_response
                    ▼                  │    (never reaches the signer)
              path == object?
                    │ no (apex "/")
                    ▼            ┌─ yes ─► signature verify (_nc_ht + oh + ohc)
            404 / connection      │           │
            handling at           │      match? ──no──► 403 text/plain
            vhost tier            │           │        "URL signature mismatch"
                    │             │           │        proxy-status: http_request_error
                    │             │        yes▼
                    │             │    ┌──────────────────────────────┐
                    └─────────────┴────┤ Everstore fetch via Needle   │
                                       │ 200 image/webp + integrity  │
                                       └──────────────────────────────┘
```

| State | Trigger | Status | Body | Server header | Debug headers |
|---|---|---|---|---|---|
| **Success** | valid signature, GET object | 200 | object bytes | (none — Everstore tier) | `x-everstore-unified-metadata`, `x-needle-checksum`, `content-digest: adler32=…` |
| **Signature mismatch** | `_nc_ht` (or `oh`/`_nc_ohc`) wrong | 403 | 22-byte `URL signature mismatch` text/plain | `proxygen-bolt` | `proxy-status: http_request_error; e_proxy=…` |
| **Bad method** | POST to object path or apex | 400 | 363-byte `4xx Client Error` HTML | **absent** | `proxy-status: proxy_internal_response; e_proxy=…; e_fb_twtaskhandle=…` |
| **Apex / nothing** | GET `/` on the vhost | 404-class page | generic 4xx shell | absent | same anonymous shell as POST |

### 2.1 The 403 tier

The signature-mismatch path is the *only* error state that identifies itself:

```
HTTP/2 403
content-type: text/plain
content-length: 22
server: proxygen-bolt
proxy-status: http_request_error; e_proxy="AcRaET-FxHCPxZv0K5zF7XAzVL4K611wibG-85IpDwRRuM_FGQE1saHTbuDdsJAMt6hDENH"
x-fb-connection-quality: GOOD; q=0.7, rtt=66, rtx=1, c=20, mss=1380, tbw=4569, ...
x-fb-ptm-uuid: E7F804BD85444EFE8875A4D97EACF67E
x-robots-tag: noarchive, noindex
alt-svc: h3=":443"; ma=86400
```

Body, all 22 bytes: `URL signature mismatch`. Not JSON, not HTML — a deliberate minimal response, cheap to generate at the edge and useless to fingerprint beyond `server: proxygen-bolt` (Meta's open-source C++ HTTP library, deployed as the edge tier). The `e_proxy` token is an encrypted correlation handle: same mismatch requested twice (`x-fb-ptm-uuid` constant `E7F804BD…` across 40 minutes) — the edge traces sessions, not just requests.

Note that the 403 *does* carry the FB edge telemetry (`x-fb-connection-quality`: RTT, retransmit count, MSS, throughput estimate `tbw`). The 400 path carries **none** of it — the internal-response short-circuit happens before the telemetry layer stamps the response.

### 2.2 The 400 tier

POST to a signed object — or to the bare apex — never reaches signature validation:

```
HTTP/2 400
content-type: text/html; charset=utf-8
content-length: 363
access-control-allow-origin: *
proxy-status: proxy_internal_response; e_proxy="…"; e_fb_twtaskhandle="AcQ3uBSB0lzw…"
```

Body — byte-for-byte identical across three independent requests in two captures:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <title>4xx Client Error</title>
  ...
  <h1>4xx Client Error</h1>
  <p>The request could not be processed.</p>
```

This is the strange part that draws attention: **a document-grade HTML error page served to top-level navigation with no `server` header, no FB telemetry, and a wildcard CORS policy** — while carrying an `e_fb_twtaskhandle` correlation token we see on no other response class. Most CDNs redirect the apex, brand their errors, or send 403. Meta's edge treats the apex exactly like a malformed object request: anonymous, unbranded, untraceable except for `proxy-status`.

One capture artifact worth flagging: the apex 400 arrived in response to an `Origin: null`, `Sec-Fetch-Site: cross-site` POST with an empty body — not something an address bar produces. The generic shell title "4xx Client Error" is easily misread as a 404; the true `GET /` 404 header set remains uncaptured.

## 3. Experiment: walking `_nc_ht`

Capture 1 is, functionally, a controlled experiment. Five requests, one object, identical signature tokens — only the host-claim parameter and method varied:

| # | Method | `_nc_ht` | Result |
|---|---|---|---|
| 1 | GET | `google.com` | **403** — `URL signature mismatch` |
| 2 | GET | (favicon request — control) | 200 |
| 3 | POST | `google.com` | **400** — method rejected before signing |
| 4 | POST | `scontent.xxfbcdn.net` (missing dot) | **400** |
| 5 | GET | `scontent.xxfbcdn.net` (missing dot) | **403** — `URL signature mismatch` |
| 6 | GET | `scontent.xx.fbcdn.net` (correct) | **200** — 369,724-byte WebP |

Reading the table:

1. **`_nc_ht` is cryptographically bound.** An absurd value (`google.com`) and a one-character-off value (`xxfbcdn` for `xx.fbcdn`) both fail the same way. The check is exact-string equality of the HMAC input, not a suffix or host-allowlist match — otherwise request 5 would have passed.
2. **Method gates before signature.** Both POSTs returned 400 regardless of `_nc_ht`. The CDN object store is read-only; POST is a protocol-level dead end answered by the internal-response tier, one level *above* the signer in the pipeline.
3. **GET + wrong `_nc_ht` is the only path that exercises the signer.** If you're probing a signature scheme, vary exactly one parameter over GET; everything else answers a different question.
4. **The signer is stateless and fast.** Wrong-signature turnaround was 204ms, 60ms across attempts — no rate limiting observed on five rapid mismatches, and no progressive penalty (no `Retry-After`, no escalating status).

## 4. What a 200 leaks: Everstore and Needle

The success response is a full fingerprint of Meta's internal storage stack:

```
HTTP/2 200
x-everstore-unified-metadata: 1
last-modified: Wed, 23 Sep 2026 16:29:15 GMT
content-type: image/webp
x-needle-checksum: 1443449978
content-digest: adler32=1443449978
x-crypto-project: 0
cross-origin-resource-policy: cross-origin
timing-allow-origin: *
access-control-allow-origin: *
cache-control: max-age=1209600, no-transform
accept-ranges: bytes
content-length: 369724
x-fb-edge-debug: ysEUxP1jK6tBpE2MnOW0iVb7M2RGZLdcnKVdjgmHLGmDSK-EmrBfj5TaeDlRCIeJtj5Fxmi2dSzlpd4oB5KfJXD1amjL4sNakfjA
```

- **`x-everstore-unified-metadata: 1`** — the object was served from **Everstore**, Meta's internal fault-tolerant blob-storage system (publicly described in Meta engineering talks as the backend for photos). The header marks the unified-metadata serving path.
- **`x-needle-checksum` / `content-digest: adler32=1443449978`** — **Needle**, the Haystack-derived storage layer's integrity checksum (adler32 over the stored blob), exposed verbatim in a response header. The CDN will happily tell you the internal checksum of any object you can sign for.
- **`x-crypto-project: 0`** — encryption-context flag; `0` = plaintext-at-rest tier for this object class.
- **`cache-control: max-age=1209600, no-transform`** — 14-day edge cache TTL, *longer* than the 7-day `oe` signature expiry. Implication: a URL outlives its own signature. After expiry the edge must re-validate with the signer rather than serve from cache — the TTL skew is deliberate defense against stale-signed-URL replay from caches.
- **`x-fb-edge-debug`** — per-response encrypted edge-debug blob; decodes only inside Meta's tooling. Presence on success, absence on 4xx, marks which tier stamped the response.
- **`timing-allow-origin: *`** — the edge permits Resource Timing reads cross-origin, so any web page can measure fbcdn latency for its embedded Facebook assets.

## 5. Connecting to the prelude

Why does the Comet client carry an entire `shouldAppendToken` domain allowlist (Part 2) that *excludes* `fbcdn.net` and `fbsbx.com`? Because these URLs sign themselves. From `URI` / `getAsyncParams` in the bundle:

- Application-layer tokens (`fb_dtsg`, jazoest) authenticate **state-changing application requests** to `facebook.com` / `*.facebook.com`.
- Media-layer tokens (`oh`, `oe`, `_nc_*`) authorize **read-only fetches** of specific objects at the edge.

The two schemes never mix: the client must not append CSRF tokens to CDN URLs (they'd break the signature), and CDN URLs carry no session state (a leaked signed URL is a capability, valid for 7 days, for exactly one object — the design assumption being that the HMAC input includes enough entropy that guessing is infeasible, which §3 confirms: partial knowledge of the scheme buys nothing without the key).

This also explains the `URISchemes` and resource-host checks in the prelude: the client treats any URL whose host resolves to an fbcdn shard as a *resource*, not a *route*, and routes it through the Bootloader/Banzai download path (Part 3) instead of async navigation.

## 6. Research notes

**Reproducing the `_nc_ht` walk:** any signed image URL from a Facebook page load works. Issue GETs varying only `_nc_ht`: an unrelated hostname, a one-character mutation, and the correct value. Expect 403 / 403 / 200. Do not POST — it answers a different tier.

**Expiry arithmetic:** `oe` is a plain hex epoch. Deltas observed across objects minted the same week cluster at exactly 7 days (± minutes). `ccb` epochs correlate with client build pushes, not object uploads.

**Checksum cross-check:** `x-needle-checksum` (adler32) can be recomputed over any fetched object; it matches `content-digest`. For archived objects this gives a third integrity anchor alongside the transport checksum and the file's own metadata — useful when validating that a re-fetched object is bit-identical to a previously captured one.

**The uncaptured state:** the true `GET /` 404 on the apex vhost. The captured apex response was a POST-induced 400. If you capture the apex GET 404 headers, the four-state table in §2 completes — send them and this post will be updated.

## 7. Summary

Facebook's CDN edge is a three-tier pipeline: an HTTP front (Proxygen) that rejects malformed methods with an anonymous 400 shell, a stateless HMAC signer that answers wrong signatures with a 22-byte 403, and an Everstore/Needle storage backend that stamps every success with internal integrity metadata. The `_nc_*` parameter family binds host claim, object revision, expiry, and shard into one signature — and the 7-day signature / 14-day cache TTL skew shows the replay model was thought through. None of this is documented by Meta. All of it is observable from a browser, a HAR recorder, and five GETs.

---

*Next: Part 9 — the Comet route payload envelope: route definitions, Haste bitmaps in `sr_payload`, and ServerJS-by-JSON.*

*Series index · Part 1 (Ghost Owl) · Part 2 (request signing) · Part 3 (Bootloader/Haste) · Part 4 (URL schemes) · Part 5 (ServerJS) · Part 6 (telemetry/Hyperion) · Part 7 (module index)*

*frankSx — hardware/web reverse engineering, parser internals, anti-analysis systems. frankhacks.blogspot.com*
