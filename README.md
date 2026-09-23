# The Facebook Comet Prelude Files

**A frankSx research series** — [frankhacks.blogspot.com](https://frankhacks.blogspot.com)

A complete public reference for Facebook's Comet prelude bundle: the critical-path JavaScript that runs before anything else on facebook.com. De-minified from a live production capture (326,001 bytes, 232 `__d` modules).

**Source artifact SHA-256:** `57d60f40f636669c259ec6f945ae4264ac4b55cba01a5e767affaad900e69ec4`

## The series

| # | Post | Subject |
|---|---|---|
| 0 | [Series index](00-index.md) | Methodology, citation, provenance |
| 1 | [Inside Ghost Owl](01-ghost-owl-anti-tampering.md) | Facebook's anti-tampering immune system: hook detection, clean-realm recovery, the SponsoredData decoy |
| 2 | [Anatomy of a Facebook Async Request](02-async-request-anatomy.md) | DTSG / LSD / jazoest token plumbing and the `getAsyncParams` map |
| 3 | [The __d Machine](03-bootloader-haste-module-system.md) | Haste module system, bootloader endpoint, retry circuit breaker, Haste bitmaps |
| 4 | [The URL Scheme Allowlist](04-uri-scheme-allowlist.md) | All 150 custom URI schemes, verbatim and categorized |
| 5 | [ServerJS and the data-sjs Pipeline](05-serverjs-data-sjs-pipeline.md) | How Facebook streams its UI |
| 6 | [The Observation Deck](06-error-telemetry-hyperion.md) | Error protocol, telemetry, and the Hyperion interception engine |
| 7 | [Appendix: Complete Module Index](07-appendix-module-index.md) | All 232 modules annotated; verbatim scheme set; cr: IDs; gate inventory |

## Why this exists

Meta does not document these internals. What public analysis exists is years out of date. Every claim here is verifiable against the bundle hash above: capture the prelude from any facebook.com page load, hash, diff.

## Citation

Link this repo or the series index on frankhacks.blogspot.com, and cite the bundle SHA-256 — prelude contents shift with deploys; the hash pins the analysis to a specific artifact.
