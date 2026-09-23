# Inside Ghost Owl: Facebook's Anti-Tampering Immune System

*Part 1 of The Facebook Comet Prelude Files · frankSx · frankhacks.blogspot.com*

---

If you've ever installed a userscript on facebook.com, hooked `XMLHttpRequest` to watch GraphQL traffic, or run an ad blocker, there is a system running in the page whose entire job is to detect you, work around you, and in some cases silently repair the damage you did. Internally it logs under the FBLogger category **`ad_blocker_defense_ghost_owl`**. The module family is prefixed `GHL`, and it ships in the critical-path prelude bundle — it executes before virtually anything else on the page, before React, before the feed, before the module system has finished warming up.

This post is a complete breakdown of the GHL system as it exists in production, de-minified from a live 326 KB Comet prelude bundle (SHA-256 `57d60f40…e69ec4`). To our knowledge no public documentation of this system exists beyond scattered observations that "Facebook detects ad blockers." The reality is considerably more sophisticated.

## 1. The module family

Six modules make up the system, with this dependency structure:

```
GHLDetectionUtilsPreludeSafe   (deps: ExecutionEnvironment, FBLogger)
├── GHLDetectionUtils          (deps: GHLDetectionUtilsPreludeSafe, gkx)
│   └── GHLNetworkLayer        (deps: FBLogger, GHLDetectionUtils,
│                                 GHLDetectionUtilsPreludeSafe, err, gkx, justknobx)
│       └── getSameOriginTransport  (deps: ExecutionEnvironment, FBLogger,
│                                    GHLDetectionUtils, GHLDetectionUtilsPreludeSafe,
│                                    GHLNetworkLayer, err, getErrorSafe, gkx, justknobx)
├── GHLServerJSParse           (consumed by ServerJSPayloadListener)
└── GHLTypenameRestore         (consumed by GHLServerJSParse)
```

| Module | Role |
|---|---|
| `GHLDetectionUtilsPreludeSafe` | Low-level primitives: string normalization, clean-realm extraction, native restoration, behavioral shim detection. "PreludeSafe" = contains no dependencies that could themselves be late — it must run before the module system is fully operational. |
| `GHLDetectionUtils` | The detection matrix: six predicates answering "has this native been tampered with?" |
| `GHLNetworkLayer` | `getGHLXhr()` — obtains a trustworthy XHR constructor by prototype walking or clean-realm extraction. |
| `GHLServerJSParse` | Parses ServerJS payloads defensively when `JSON.parse` can't be trusted. |
| `GHLTypenameRestore` | Repairs `__typename` values in GraphQL payloads that content filters corrupt. |
| `getSameOriginTransport` | The choke point: every same-origin XHR Facebook makes goes through here, so every request benefits from GHL recovery. |

## 2. The detection matrix

The core of the system is a set of predicate functions in `GHLDetectionUtils`. Each checks whether a native function has been replaced or wrapped. The technique is identical across all of them — normalize `fn.toString()` and compare against the expected V8 native-code signature:

```js
// de-minified from GHLDetectionUtils
function isXHRModified(xhr) {
  return gkx("8869") ? true
    : typeof xhr == "function" && !(
        xhr.toString === xhr.toString.toString &&
        normalize(xhr.toString()) === "function XMLHttpRequest() { [native code] }" &&
        normalize(xhr.toString.toString()) === "function toString() { [native code] }"
      );
}
```

Three things are verified, and the second is the clever part:

1. **`fn.toString()` must serialize to the exact native signature.** If you replaced `XMLHttpRequest` with a wrapper, its source leaks.
2. **`fn.toString === fn.toString.toString`** — catches `Function.prototype.toString` spoofing. The classic evasion is overriding `toString` on your hook so it *returns* the native signature string. But that override is itself a function object, and the identity relationship between a function's `toString` and `toString.toString` only holds for the genuine `Function.prototype.toString`. Facebook checks the identity, not the output.
3. **`toString.toString()` must also be native.** Defense in depth against replacing `Function.prototype.toString` globally.

Normalization collapses whitespace so reformatted natives still match:

```js
// de-minified from GHLDetectionUtilsPreludeSafe
function normalizeString(s) {
  return typeof s.replace == "function"
    ? s.replace(/\n/g, " ").replace(/\s+/g, " ")
    : null;
}
```

### The complete check inventory

| Predicate | Target | Gate | Native signature checked |
|---|---|---|---|
| `isJSONParseShimmed()` | `JSON.parse` | `gkx("5415")` | *(via PreludeSafe)* |
| `isXHRModified(xhr)` | `XMLHttpRequest` | `gkx("8869")` | `function XMLHttpRequest() { [native code] }` |
| `isCanvasFillTextModified(ctx)` | `CanvasRenderingContext2D.prototype.fillText` | `gkx("9063")` | `function fillText() { [native code] }` |
| `isStringShimmed()` | `String` | — | `function String() { [native code] }` |
| `isCallShimmed()` | `Function.prototype.call` | — | `function call() { [native code] }` |
| `isNativeStackTampered()` | error stack infrastructure | — | *(PreludeSafe)* |
| `isXHRResponseGetterShimmed()` | XHR `response`/`responseText` property getters | — | *(PreludeSafe)* |

The canvas check deserves emphasis: `fillText` is the canonical canvas-fingerprinting primitive. Both fingerprinting scripts *and* anti-fingerprinting extensions (which wrap `fillText` to poison readings) trip this check. Facebook is watching both sides.

### Behavioral detection

Signature checks miss hooks that proxy through the real native. So the PreludeSafe module also ships behavioral variants:

- `isJSONParseBehaviorallyShimmed()` — probes `JSON.parse` with inputs whose observable side effects differ under a proxy (e.g. reviver invocation order, receiver identity).
- `isCallBehaviorallyShimmed()` — same idea for `Function.prototype.call`.
- `isStringBehaviorallyShimmed()` — same for `String`.
- `isBoxedParseEffective()` / `isWrappedParseEffective()` — test whether the *mitigation* (boxed/wrapped parsing, §5) actually neutralizes the shim, before relying on it.

Detection results feed a decision, not a log-and-die: the system picks a parsing/transport strategy that routes around the tampering.

## 3. Recovery tier 1: prototype-chain walking

`GHLNetworkLayer.getGHLXhr()` doesn't give up when it finds a hooked XHR. It walks the prototype chain (up to 5 levels) looking for an ancestor whose constructor still serializes as native:

```js
// de-minified from GHLNetworkLayer
function getGHLXhr() {
  try {
    if (gkx("8068") && isXHRModified(window.XMLHttpRequest)) {
      var MAX_DEPTH = 5;
      var proto = Object.getPrototypeOf(window.XMLHttpRequest);
      for (var depth = 0; proto != null && depth < MAX_DEPTH; depth++) {
        var s = normalize(proto.toString());
        if (s != null && s.includes("{ [native code] }")) break;
        proto = Object.getPrototypeOf(proto);
      }
      if (proto != null && !isXHRModified(proto)) return proto;
    }
    // tier 2: clean realm
    if (gkx("25201") && justknobx._("5685") && isXHRModified(window.XMLHttpRequest)) {
      return getCleanRealmXHRConstructor();
    }
    return null;
  } catch (e) {
    FBLogger("ghl").catching(e).mustfix("error while getGHLXhr");
    return null;
  }
}
```

This defeats the common hook pattern `class FakeXHR extends XMLHttpRequest` — the parent is still clean, so Facebook uses the parent. Only a hook that *replaces* the constructor without inheritance, or poisons the entire chain, forces tier 2.

## 4. Recovery tier 2: the hidden iframe armory

When the whole chain is dirty, `GHLDetectionUtilsPreludeSafe` falls back to **cross-realm recovery**: create a hidden iframe, pull pristine natives out of its `contentWindow`. A fresh realm has fresh natives no page-level hook has touched.

What makes this remarkable is *how* the iframe gets inserted. The prelude doesn't just call `document.body.appendChild(iframe)`. It checks **13 obfuscated gate keys in `window.Env`**, each unlocking a different DOM insertion technique:

| Env key | Technique unlocked |
|---|---|
| `h4npx7qw` | `iframe.src = "about:blank#g"` |
| `p9fk3wmn` | `srcdoc=""` variant |
| `f2yq8vnd` | `document.createElement` path |
| `t5nd8vqc` | `document.createComment` path |
| `m8r3kp6w` | `document.createRange` path |
| `b3xk8fqm` | `firstElementChild.before()` |
| `q4v7nx3k` | `firstElementChild.after()` |
| `r7c2m9xk` | `body.prepend()` |
| `z2ht6xqp` | `body.append()` |
| `k7q3nv9d` | *(flag — path selection)* |
| `w6jt4rnq` | *(flag — path selection)* |
| `w8kq3zmt` | `body.replaceChild()` |
| `b7xr2qnf` | `body.replaceChildren()` |

The insertion cascade, de-minified:

```js
// de-minified from GHLDetectionUtilsPreludeSafe
var span   = canCreateElement ? document.createElement("span") : null;
var div    = canReplaceChildren ? document.createElement("div") : null;
var span2  = canReplaceChild ? document.createElement("span") : null;
var comment = canCreateComment ? document.createComment("") : null;
var range  = canCreateRange ? document.createRange() : null;

if (span != null && "replaceWith" in span) {
  parent.appendChild(span);
  span.replaceWith(iframe);
} else if (div != null) {
  div.style.display = "none";
  parent.appendChild(div);
  div.replaceChildren(iframe);      // keep div as removable handle
  cleanupHandle = div;
} else if (span2 != null) {
  parent.appendChild(span2);
  parent.replaceChild(iframe, span2);
} else if (comment != null && "replaceWith" in comment) {
  // ...comment.replaceWith(iframe)
}
// ...range-based insertion as final fallback
```

Why carry a dozen insertion strategies? Because **ad blockers and content scripts block specific DOM APIs**. Filter lists neuter `appendChild` for iframes; anti-fingerprint extensions strip iframes via MutationObserver; some extensions break `replaceWith` specifically. By cascading through `appendChild`+`replaceWith`, `replaceChildren`, `replaceChild`, comment replacement, and Range insertion, Ghost Owl ensures at least one path survives whatever the user's extension stack throws at it. The server selects which paths are armed per-deploy via the Env keys — the keys are 8-character random identifiers, rotated to prevent signatures.

Once the clean realm exists, the module extracts:

- `getCleanRealmXHRConstructor()` — pristine `XMLHttpRequest`
- `getCleanJSONParse()` — pristine `JSON.parse`
- `restoreNativeCall()` — reinstalls real `Function.prototype.call` in the main realm
- `restoreNativeString()` — reinstalls real `String`
- `restoreNativeXHRGetters()` — restores XHR response property getters
- `isCallShimmedCrossRealm()` — cross-realm comparison check (gate `justknobx("2694")` + `gkx("5023")`, with the newer `justknobx("5589")` + `gkx("23984")` path)

## 5. The defensive parse chain

`GHLServerJSParse.ghlParseServerJSPayload` is where detection becomes action. Given a ServerJS payload string, the full fallback chain (de-minified):

```js
function ghlParseServerJSPayload(raw) {
  var env = window.Env;
  var armPreTransform  = env && "v9k2mt7q" in env;
  var armBehavioral    = env && "d3hf9km2" in env;
  var armTransportMark = env && "k8pq2mnb" in env;
  var armTypenameFix   = env && "n5tq2wjb" in env;

  var payload = raw;
  if (armTransportMark && payload != null) payload = replaceTransportMarkers(payload);

  var shimmed = (armPreTransform && isJSONParseShimmed())
             || (armBehavioral && isJSONParseBehaviorallyShimmed());

  var result = null, ok = false;

  // attempt 1: boxed parse — isolate payload inside an object literal
  if (env && "c6mw9qtk" in env && shimmed && payload != null
      && (!("j6dw4ztx" in env) || isBoxedParseEffective())) {
    try {
      var boxed = JSON.parse('{"q7z":' + payload + "}");
      if (boxed != null && boxed.q7z != null) { result = boxed.q7z; ok = true; }
    } catch (e) { ok = false; }
  }

  // attempt 2: wrapped parse — isolate inside an array literal
  if (!ok && shimmed && env && "x8kf2pw6" in env && payload != null
      && (!("j6dw4ztx" in env) || isWrappedParseEffective())) {
    try {
      var wrapped = JSON.parse("[" + payload + "]");
      if (Array.isArray(wrapped) && wrapped.length === 1) { result = wrapped[0]; ok = true; }
    } catch (e) { ok = false; }
  }

  // attempt 3: clean-realm parse, optionally restoring String first
  if (!ok && shimmed) {
    try {
      if (env && "r4wt7kmj" in env && isStringBehaviorallyShimmed()) restoreNativeString();
      var cleanParse = getCleanJSONParse();
      var cleanOk = false;
      if (cleanParse != null) {
        try { result = cleanParse(payload); cleanOk = true; } catch (e) {}
      }
      // attempt 4: vendored json5 — no dependency on JSON.parse at all
      if (!cleanOk) result = json5.parse(payload + " ");
    } catch (e) {
      FBLogger("ad_blocker_defense_ghost_owl").catching(getErrorSafe(e))
        .mustfix("Failed to parse ServerJS payload using json5");
      result = JSON.parse(payload);   // last resort
    }
  } else if (!ok) {
    result = JSON.parse(payload);
  }

  // post-parse: repair __typename values stripped by content filters
  if (armTypenameFix && result != null && payload.indexOf(typenameMarker) !== -1)
    GHLTypenameRestore.restoreTypenameValues(result, typenameMarker, restoreValue);
  if (result != null) GHLTypenameRestore.restoreAllTypenames(result, payload);
  return result;
}
```

Design notes:

- **Boxed and wrapped parsing** work because many `JSON.parse` shims only behave correctly at the top level (they post-process the result object). Nesting the payload inside `{"q7z":...}` or `[...]` changes what the shim sees and often bypasses naive result-poisoning.
- **The trailing space** in `json5.parse(payload + " ")` works around a json5 end-of-input edge case.
- **Four independent parse strategies** mean a hook author must defeat all of them — including a parser that doesn't touch `JSON.parse` at all.
- Additional Env gates in this module beyond the 13 iframe keys: `v9k2mt7q`, `d3hf9km2`, `k8pq2mnb`, `n5tq2wjb`, `c6mw9qtk`, `j6dw4ztx`, `x8kf2pw6`, `r4wt7kmj` — 8 more, same 8-char random format.

## 6. The SponsoredData decoy

Buried in `GHLDetectionUtilsPreludeSafe` is one of the strangest artifacts in the bundle:

```js
// de-minified
function makeSponsoredData() {
  var parts = ["Spon", "sored", "Data"];   // split to dodge static string matching
  var t = "";
  for (var i = 0; i < parts.length; i++) t += parts[i];
  return '{"node":{"s":{"__typename":"' + t + '"}}}';
}

function repeatHex(len) {
  var s = "3f0a7c1b9e42d685";
  while (s.length < len) s += s;
  return s.slice(0, len);
}

var sponsored = makeSponsoredData();
var decoyPayload =
  '{"data":' + sponsored +
  ',"edges":[' + sponsored +
  '],"require":[' + sponsored +
  '],"p":"' + repeatHex(4095) +
  '","extensions":{"is_final":true}}';
```

A **synthetic GraphQL response**: fake `SponsoredData` nodes in `data`, `edges`, and `require` positions, 4,095 characters of hex padding, and the `extensions.is_final` terminator that marks the end of a real streamed response. The `__typename` string is assembled from fragments so static scanners don't find `"SponsoredData"` in the source.

Combined with `GHLTypenameRestore` — which repairs `__typename` values that filter lists strip from *real* payloads — the picture is clear: Ghost Owl doesn't just defend against tampering, it actively **deceives content filters**, feeding them synthetic sponsored-content structures while restoring the markers they remove. This is an arms race conducted entirely inside the JavaScript layer, and Facebook has industrialized it.

## 7. The gate inventory

Every GHL behavior is remotely switchable. The complete gate list observed in this bundle:

**GK (gatekeeper) IDs:** `999`, `1174`, `5023`, `5163`, `5415`, `7742`, `8068`, `8869`, `9063`, `11588`, `15745`, `18719`, `20935`, `20936`, `23984`, `25201`

**JustKnobx IDs:** `340`, `1590`, `2694`, `3323`, `4216`, `5589`, `5685`

GKs are boolean feature gates (exposed via `gkx`, with exposure logging through `BanzaiLazyQueue` to the `gk2_exposure` endpoint — Meta logs *which gates your client evaluated*). JustKnobx are runtime-tunable knobs. Not all 23 are GHL-specific (some gate bootloader and scheduling behavior), but the GHL cluster is identifiable: `5415`, `8869`, `9063` (detection predicates), `8068`, `25201`+`5685` (recovery tiers), `5023`/`23984`/`2694`/`5589` (call-shim handling), `340`+`999` (XHR getter restoration), `3323` (transport selection).

**Practical consequence:** a hook that works today may be detected tomorrow without any code shipping — Meta flips a gate server-side, per-cohort, per-percentage.

## 8. What this means for instrumentation authors

1. **Naive XHR/fetch hooks are detected and bypassed.** Your hook stays in place, but Facebook's traffic routes around it via a prototype ancestor or an iframe-realm constructor. You see *nothing* while believing you see everything — the worst outcome for instrumentation.
2. **toString spoofing is checked twice deep.** Returning the native signature from a custom `toString` fails the `toString === toString.toString` identity check.
3. **Prototype-inheriting hooks don't help.** Tier 1 walks up to 5 ancestors looking for a clean one.
4. **Hooking JSON.parse is a losing game.** Four parse strategies, one of which (json5) ignores `JSON.parse` entirely.
5. **The one thing Ghost Owl cannot do** is remove your hook from *your* realm or observe below the JS layer. Instrumentation at the network layer — Chrome DevTools Protocol, a service worker you control, a local proxy — is structurally immune to everything described here. Ghost Owl is a same-realm defense; don't fight it in the same realm.
6. **For defenders**, the design lessons are worth stealing: identity checks over output checks, behavioral probes over signatures, recovery over blocking, and gate-everything so the defense can evolve without deploys.

## 9. Verification

Every excerpt above de-minifies cleanly from the captured bundle (hash in the series index). To reproduce: load facebook.com, view source, extract the inline script beginning `;/*FB_PKG_DELIM*/`, and search for `ad_blocker_defense_ghost_owl`, `getCleanRealmXHRConstructor`, or any Env gate key from §4. Gate arming varies by cohort and deploy; the code paths are constant.

---

*Next: the token plumbing behind every authenticated Facebook request.*
