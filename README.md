# The Facebook Comet Prelude Files

> **The first complete public de-minification and documentation of Facebook's Comet prelude bundle** — the critical-path JavaScript that executes before anything else on facebook.com: the Haste module system, the Bootloader, CSRF token plumbing, the ServerJS streaming renderer, the error/telemetry stack, and **Ghost Owl**, Facebook's in-page anti-tampering and ad-blocker defense system.

**A frankSx research series** · [frankhacks.blogspot.com](https://frankhacks.blogspot.com) · [Read the rendered series on GitHub Pages](https://franksx.github.io/facebook-comet-prelude-files/)

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Modules documented](https://img.shields.io/badge/modules%20documented-232-58d6c9)](#the-series)
[![URI schemes enumerated](https://img.shields.io/badge/URI%20schemes-150-58d6c9)](04-uri-scheme-allowlist.md)
[![Bundle SHA--256 pinned](https://img.shields.io/badge/bundle-SHA--256%20pinned-f97583)](#provenance--verification)

---

## Provenance & verification

Every claim in this series traces to a single captured artifact:

| | |
|---|---|
| **Artifact** | Facebook Comet prelude bundle (`;/*FB_PKG_DELIM*/` wrapper), inline critical-path JS from facebook.com |
| **Captured** | September 2026 |
| **Size** | 326,001 bytes · 310 lines · 232 `__d` modules |
| **SHA-256** | `57d60f40f636669c259ec6f945ae4264ac4b55cba01a5e767affaad900e69ec4` |
| **MD5** | `6f7a5cb0db878cdd9be1901cd5e2724b` |

**Reproduce:** load facebook.com → view source → extract the inline script beginning `;/*FB_PKG_DELIM*/` → hash → diff. Module names, gate IDs, and string literals quoted in this series are verbatim. Prelude contents shift with deploys; the hash pins this analysis to a specific artifact. Diff future captures against [Part 7's module index](07-appendix-module-index.md) to watch the system evolve.

## Headline findings

- 🦉 **Ghost Owl (GHL) — Facebook's anti-tampering immune system.** Detects hooked `JSON.parse`, `XMLHttpRequest`, `String`, `Function.prototype.call`, and canvas `fillText` via native-signature comparison with a **double-`toString` identity check** that defeats `toString` spoofing. Recovers pristine natives by walking the XHR prototype chain (5 deep) or extracting them from a **hidden clean-realm iframe inserted through 13 server-armed DOM techniques**. Ships a vendored **json5 parser** so a poisoned `JSON.parse` can't corrupt ServerJS. Actively **deceives content filters** with a synthetic `SponsoredData` GraphQL decoy. → [Part 1](01-ghost-owl-anti-tampering.md)
- 🔑 **The complete request-signing map.** POST vs GET CSRF token split (`fb_dtsg` / `fb_dtsg_ag`, independent rotation), the **jazoest** checksum derivation (char-code sum, version-prefixed), the `lsd` fallback, WebSession's Alea-seeded 6-char base-36 IDs, and the exact domain allowlist that decides when tokens are attached — including the deliberate `fbsbx.com` exclusion. → [Part 2](02-async-request-anatomy.md)
- ⚙️ **The Haste/Bootloader kernel.** The `__d` registry, module state bitmasks, reference counting with module freeing, four require flavors, the batched bootloader endpoint (code fetches carrying CSRF tokens), a retry **circuit breaker**, RLE-compressed Haste bitmaps, and ClientConsistency soft/hard refresh. → [Part 3](03-bootloader-haste-module-system.md)
- 🔗 **All 150 custom URL schemes, verbatim.** The `URISchemes` allowlist — web→native bridges for Messenger, Threads (`barcelona`), WhatsApp, Oculus/Horizon, India UPI payment rails, and 29 internal-tool codenames (`flipper`, `munki`, `manifold`, `fb-owl`…). The first complete public enumeration. → [Part 4](04-uri-scheme-allowlist.md)
- 📡 **ServerJS, the original streaming SSR.** `<script data-sjs>` payloads with a **content-length integrity gate**, dual-generation listeners, transport markers, and scheduled incremental execution — 15 years before React Server Components. → [Part 5](05-serverjs-data-sjs-pipeline.md)
- 👁 **Hyperion — the interception engine.** Meta instruments the DOM with `__ext`/`__sproto` shadow prototypes at the property-descriptor level — the same technique Ghost Owl defends against, turned inward. Plus the `fb-error` structured-error protocol (`messageFormat`/`taalOpcodes`), hash-based log rate limiting, and gate-exposure telemetry. → [Part 6](06-error-telemetry-hyperion.md)
- 📚 **All 232 modules annotated.** The reference appendix: every prelude module categorized with dependency counts and one-line descriptions, plus verbatim scheme set, `cr:` IDs, and the complete gate inventory. → [Part 7](07-appendix-module-index.md)
- 🖼 **fbcdn signed-URL internals, from live captures.** Facebook's second, independent signature scheme: the `_nc_ht` HMAC host-binding (proven by inducing `URL signature mismatch` 403s), the four-state Proxygen edge decision table, Everstore/Needle integrity headers on every 200, and the 7-day-signature / 14-day-cache replay model. → [Part 8](08-fbcdn-signed-url-internals.md)
- 📦 **The Comet route payload envelope, annotated from production traffic.** The full `route_definition` wire format: URI-keyed envelope, `__jsr`/`__dr` resource recipes, `entityKeyConfig` store-key recipes, Haste BitMap RLE (`":1,2,31"`) in `sr_payload.hsrp.hblp`, factory-wrapped `jsmods` requires, and the nulled `dtsgToken` on the GET route path. → [Part 9](09-comet-route-payload-envelope.md)

## The series

| # | Post | What you'll learn |
|---|---|---|
| 0 | [Series index & methodology](00-index.md) | Provenance, de-minification conventions, citation |
| 1 | [Inside Ghost Owl: Facebook's Anti-Tampering Immune System](01-ghost-owl-anti-tampering.md) | Hook detection matrix, clean-realm recovery, gate keys, evasion analysis |
| 2 | [Anatomy of a Facebook Async Request](02-async-request-anatomy.md) | DTSG/LSD/jazoest, `getAsyncParams` parameter map, replay checklist |
| 3 | [The __d Machine: Bootloader & Haste](03-bootloader-haste-module-system.md) | Module registry, endpoint protocol, retries, bitmaps, consistency |
| 4 | [The 150-Entry URI Scheme Allowlist](04-uri-scheme-allowlist.md) | Every web→native bridge, categorized, with parser hardening |
| 5 | [ServerJS and the data-sjs Pipeline](05-serverjs-data-sjs-pipeline.md) | Streaming UI architecture, integrity gates, defensive parsing |
| 6 | [The Observation Deck](06-error-telemetry-hyperion.md) | Error protocol, QPL/Banzai telemetry, Hyperion internals |
| 7 | [Appendix: Complete Module Index](07-appendix-module-index.md) | All 232 modules + verbatim data appendices |
| 8 | [fbcdn Signed URL Internals](08-fbcdn-signed-url-internals.md) | `_nc_*` HMAC binding, four-state edge, Everstore/Needle, capture evidence |
| 9 | [The Route Payload Envelope](09-comet-route-payload-envelope.md) | route_definition JSON, Haste bitmaps in `sr_payload`, ServerJS-by-JSON |

## By the numbers

| Metric | Value |
|---|---|
| Prelude modules documented | **232 / 232** |
| Custom URI schemes enumerated | **150** |
| Ghost Owl gate keys mapped | **21** `window.Env` keys + **16** gkx + **7** justknobx |
| Numbered invariants mapped to call sites | 18 |
| `cr:` conditional-require IDs catalogued | 22 |
| Edge states mapped (fbcdn four-state table) | **4** (200 / 403 / 400 / apex) |
| Envelope layers annotated (route payload) | **3** (payloads / sr_payload / tokens) |
| Words of analysis | ~16,800 |

## Who this is for

- **Security researchers & bug hunters** — the token-attachment boundary, scheme allowlist, and Ghost Owl detection surface are all live attack/defense terrain.
- **Web RE practitioners** — the module index and `getAsyncParams` map are the missing manual for reading any Meta bundle.
- **Instrumentation & tooling authors** — Part 1 §8 tells you exactly why your XHR hooks silently stop seeing Facebook traffic.
- **Frontend architects** — ServerJS and the Bootloader are a production-hardened masterclass in streaming, scheduling, and defense-in-depth.

## References & further reading

Foundational context and prior art for the systems documented here:

1. *Building the new facebook.com with React, GraphQL and Relay* — Facebook Engineering, May 2020 (the Comet architecture overview; the prelude is what boots it). `engineering.fb.com/2020/05/08/web/facebook-redesign/`
2. *BigPipe: Pipelining web pages for high performance* — Facebook Engineering, 2010 (the spiritual ancestor of ServerJS streaming).
3. *JSON Hijacking* — Haack/ Grossman-era write-ups on the attack class the `for (;;);` prefix defeats; see `CSRFGuard` in [Part 2](02-async-request-anatomy.md#5-the-for--guard).
4. JSON5 specification — `json5.org` (the vendored fallback parser in [Part 1](01-ghost-owl-anti-tampering.md#5-the-defensive-parse-chain)).
5. Alea PRNG — Johannes Baagøe, 2010 (the seeded PRNG behind `__s` session IDs, [Part 2 §3](02-async-request-anatomy.md)).
6. OWASP — *Prototype Pollution* (the `__proto__`/`hasOwnProperty` query-key defense in [Part 4 §3](04-uri-scheme-allowlist.md)).
7. RFC 3986 — *Uniform Resource Identifier: Generic Syntax* (the parser core in `URIRFC3986`).
8. uBlock Origin filter-list ecosystem — the adversary Ghost Owl's decoy payloads and insertion cascade are built to survive.

## Citation

```bibtex
@misc{franksx2026prelude,
  author = {frankSx},
  title  = {The Facebook Comet Prelude Files: A Complete De-minification of
            Facebook's Critical-Path JavaScript},
  year   = {2026},
  howpublished = {\url{https://github.com/FrankSx/facebook-comet-prelude-files}},
  note   = {Source artifact SHA-256:
            57d60f40f636669c259ec6f945ae4264ac4b55cba01a5e767affaad900e69ec4}
}
```

When citing, include the bundle SHA-256 — it pins every claim to a verifiable artifact.

## License

Analysis and prose: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Quoted Facebook code excerpts remain the property of Meta Platforms, Inc. and are reproduced here in de-minified form for security research, commentary, and education.

---

*frankSx — hardware/web reverse engineering, parser internals, anti-analysis systems. Previously: the SVG Jailbreak series, MESCALINE defense research, OMNIBUS hardware RE toolkit.*

`facebook` `reverse-engineering` `javascript-internals` `security-research` `haste` `bootloader` `ghost-owl` `anti-tampering` `adblock-detection` `graphql` `csrf` `serverjs` `hyperion` `web-recon` `osint` `meta` `fbcdn` `cdn-internals` `signed-urls` `proxygen` `everstore` `hmac` `comet-router` `relay` `route-definition` `request-signing` `web-reconnaissance`

---

*Series search descriptions (≤110 chars) for aggregators/search:*

- **Part 8:** `Reverse engineering Facebook's fbcdn signed image URLs: _nc_* HMAC parameters, Proxygen 403/400 error tiers, Everstore headers.` (110)
- **Part 9:** `Reverse engineering Facebook's Comet route payload envelope: route_definition JSON, Haste bitmaps, deferred requires, ServerJS.` (110)
