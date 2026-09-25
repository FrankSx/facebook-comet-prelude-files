# The Facebook Comet Prelude Files — Series Index

**A frankSx research series · frankhacks.blogspot.com**

**Source artifact:** a 326,001-byte production Facebook "Comet prelude" bundle (`;/*FB_PKG_DELIM*/` wrapper), 232 `__d` modules, captured September 2026 from the critical-path inline JavaScript of facebook.com.

- SHA-256: `57d60f40f636669c259ec6f945ae4264ac4b55cba01a5e767affaad900e69ec4`
- MD5: `6f7a5cb0db878cdd9be1901cd5e2724b`

This is the code that runs before anything else on the page: polyfills, the Haste module system, the bootloader, CSRF plumbing, the error/telemetry stack, and Ghost Owl — Facebook's anti-tampering defense system. Meta does not document these internals publicly. What little exists in blog posts and CTF write-ups is years out of date. This series is intended to be the canonical public reference.

## The posts

1. **Inside Ghost Owl: Facebook's Anti-Tampering Immune System** — the `GHL*` module family: hook detection via native-signature comparison, double-`toString` anti-spoofing, prototype-chain walking, clean-realm recovery from hidden iframes, the 13 obfuscated `window.Env` gate keys, the `SponsoredData` decoy payload, and the vendored json5 fallback. With a complete gate-ID inventory and evasion analysis for instrumentation authors.

2. **Anatomy of a Facebook Async Request: DTSG, LSD, jazoest, and getAsyncParams** — the complete request-signing map: POST vs GET token split (`fb_dtsg` / `fb_dtsg_ag`), the jazoest checksum derivation, LSD fallback, WebSession ID format and storage, the token-attachment domain allowlist, and a replay checklist for tooling authors.

3. **The __d Machine: Facebook's Bootloader and Haste Module System** — the registry kernel, module state bitmasks, four require flavors, the bootloader endpoint request/response format, retry circuit breaker, Haste bitmaps, translation fetching, resource scheduling tiers, and ClientConsistency.

4. **Facebook's Custom URL Scheme Allowlist: A 150-Entry Attack Surface Inventory** — the complete `URISchemes` set, verbatim and categorized, with enforcement mechanics, adjacent parser hardening, and research notes per category.

5. **ServerJS and the data-sjs Pipeline: How Facebook Streams Its UI** — the streaming architecture: payload anatomy, the content-length integrity gate, dual-generation listeners, the defensive parse chain, transport markers, and scheduled execution.

6. **The Observation Deck: Facebook's Error, Telemetry, and Instrumentation Stack** — `fb-error`'s `messageFormat`/`taalOpcodes` protocol, ErrorGuard/ErrorPubSub, QPL, UserTimingUtils, Banzai, NetworkHeartbeat, and a full de-minification of **Hyperion** — Meta's property-descriptor interception engine with `__ext`/`__sproto` shadow prototypes.

7. **Appendix: The Complete Annotated Module Index** — all 232 prelude modules, categorized, with dependency counts and one-line functional descriptions. The reference table for anyone doing their own analysis.
8. **fbcdn Signed URL Internals: `_nc_*` Parameters, Everstore, and the Proxygen Edge** — Facebook's second, independent signature scheme, reverse-engineered from live captures: the `_nc_ht` HMAC binding (proven by deliberately breaking it), the four-state edge decision table (200/403/400/apex), Everstore/Needle integrity headers, and the 7-day signature vs 14-day cache TTL replay model. With capture-evidence flowchart. *(permalink `/2026/09/fbcdn-signed-url-internals-nc-parameters.html`)*

9. **The Route Payload Envelope: How Facebook Ships a Page as JSON** — a complete annotated `route_definition` from production traffic: the URI-keyed envelope, `__jsr`/`__dr` resource recipes, the `entityKeyConfig` store-key recipe, Haste BitMap RLE in `sr_payload.hsrp.hblp` (`":1,2,31"` decoded), factory-wrapped `jsmods` requires, and why `dtsgToken` is nulled on the GET route path. The wire format that Parts 2, 3, and 5 exist to consume. *(permalink `/2026/09/comet-route-payload-envelope-facebook-json.html`)*

## Methodology

All code excerpts were de-minified from the captured bundle. Minified identifiers (`e`, `t`, `n`, `r`) are preserved or renamed only where the intent is unambiguous from context (exported names, dependency arrays, FBLogger category strings, invariant messages). Module names in `__d("...")` declarations, gate IDs, and string literals are verbatim from production. Every claim in this series can be verified against the bundle hash above: capture the prelude from any facebook.com page load (`view-source:` or devtools → the first inline `<script>` blocks), hash it, and diff.

## Citation

If you reference this work, link the series index on frankhacks.blogspot.com and cite the bundle SHA-256 above — prelude contents shift with deploys, and the hash pins the analysis to a specific artifact.

*frankSx — hardware/web reverse engineering, parser internals, anti-analysis systems. Previously: the SVG Jailbreak series, MESCALINE defense research, OMNIBUS hardware RE toolkit.*
