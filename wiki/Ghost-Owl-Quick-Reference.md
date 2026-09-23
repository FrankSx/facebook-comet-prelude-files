# Ghost Owl — Quick Reference

FBLogger category: `ad_blocker_defense_ghost_owl`. Module prefix: `GHL`. Full analysis: [Part 1](https://github.com/FrankSx/facebook-comet-prelude-files/blob/main/01-ghost-owl-anti-tampering.md).

## Detection matrix

| Predicate | Target | Gate |
|---|---|---|
| `isJSONParseShimmed()` | `JSON.parse` | gkx 5415 |
| `isXHRModified()` | `XMLHttpRequest` | gkx 8869 |
| `isCanvasFillTextModified()` | canvas `fillText` | gkx 9063 |
| `isStringShimmed()` | `String` | — |
| `isCallShimmed()` | `Function.prototype.call` | — |
| `isNativeStackTampered()` | error stacks | — |
| `isXHRResponseGetterShimmed()` | XHR getters | — |

Core check: `fn.toString === fn.toString.toString` plus exact native-signature string match. Defeats toString spoofing.

## Recovery

1. **Tier 1** — walk XHR prototype chain (max depth 5) for a clean ancestor (gkx 8068).
2. **Tier 2** — hidden iframe clean realm; pull pristine `XMLHttpRequest` / `JSON.parse` (gkx 25201 + jk 5685).

## window.Env gate keys

Iframe insertion paths: `h4npx7qw p9fk3wmn f2yq8vnd t5nd8vqc m8r3kp6w b3xk8fqm q4v7nx3k r7c2m9xk z2ht6xqp k7q3nv9d w6jt4rnq w8kq3zmt b7xr2qnf`

Parse-strategy arming: `v9k2mt7q d3hf9km2 k8pq2mnb n5tq2wjb c6mw9qtk j6dw4ztx x8kf2pw6 r4wt7kmj`

## Parse fallback chain

boxed `{"q7z":…}` → wrapped `[…]` → clean-realm JSON.parse → vendored json5 (`payload + " "`) → native JSON.parse (mustfix log).

## Evasion notes

* Same-realm hooks are detected and bypassed; instrument at network layer (CDP, service worker, proxy).
* Hyperion activity tell: `__ext` / `__sproto` own-properties on DOM objects; switch is `Env.loadHyperion`.
