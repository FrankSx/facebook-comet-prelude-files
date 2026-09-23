# The __d Machine: Facebook's Bootloader and Haste Module System

*Part 3 of The Facebook Comet Prelude Files · frankSx · frankhacks.blogspot.com*

---

View source on any Meta property and you won't find a pile of `<script src>` tags. You'll find inline `__d(...)` calls and a custom module runtime that predates ES modules, webpack, and most of the modern toolchain. This is **Haste** — Facebook's module system — and the prelude bundle contains its entire kernel: registry, resolver, network loader, retry logic, scheduler, and telemetry. This post documents all of it.

## 1. The registry: `__d` and the loader kernel

Every module registers with `__d`:

```js
__d("CSRFGuard", [], function(global, require, requireDynamic, requireLazy, module, exports) {
  // ...
  exports.clean = clean;
}, 66);
```

Signature: **name, dependency array, factory, kind bitmask**. The trailing number encodes module flags. Observed values in this bundle: `66`, `98`, `34`, `null`, `99` — bits covering ES-module semantics, whether the factory may be cleared after execution (`Env.clear_js_factory_after_used`), and deferred-definition behavior.

The loader kernel is an IIFE (installed before any `__d` call) that maintains the registry. From the de-minified source, its notable mechanics:

### Module state and reference counting

```js
// de-minified from the loader kernel
var MODULE_READY = 1, /* state bitfield: */ h=1, y=2, C=4, b=8, v=16, S=32, R=64, L=128, E=256;

function releaseModule(id) {
  var mod = registry[id];
  if (!mod || (mod.exports == null && !mod.factoryFinished)) initialize(id);
  mod = registry[id];
  if (mod && mod.refcount-- === 1) {
    reportUnused(mod);
    registry[id] = null;          // module entry is FREED
  }
}
```

Modules carry a `refcount`; at zero the entry is freed. The kernel tracks `wasUsed` per module and counts used-vs-unused (`m++` / `p++` / `_++` counters) — Facebook measures exactly how much shipped code never executes, per page load.

### Error-guarded factories

Every factory invocation goes through `ErrorGuard.applyWithGuard(initialize, null, [id])`. A throwing module doesn't take down the loader; the error is reported and the module marked failed, and dependent modules get a typed "waiting for" error.

### The dependency-wait debugger

The kernel can dump a full wait graph:

```js
// de-minified — produces e.g.:
// "ServerJS is waiting for ContextualComponent, HasteBitMap"
// "Bootloader is ready"
// "SomeModule is not defined"
```

This is the same machinery behind the "Loading" debug overlays Meta engineers see internally.

### Numbered invariants

Internal errors throw `invariant(0, 2506)`-style numbered failures, not strings. The decoder link points at `https://www.internalfb.com/intern/invariant/<id>/?args[0]=...` — the human-readable messages exist only inside Meta's network. Externally, the numbers are the documentation; this series maps several (2506, 2828, 1764, 1966–1971, 5809, 73819, 4494, 47458/47459, 62571, 88579, 77517, 1445, 2616, 2966, 3721) to their call sites.

## 2. Four require flavors

| API | Semantics |
|---|---|
| `require(name)` | Hard dependency. Throws if missing. |
| `requireWeak(name, cb)` | Callback only if the module is *already* loaded; never triggers a fetch. Used for optional telemetry (QPL flow IDs in `getAsyncParams`, memoize instrumentation). |
| `requireDeferred(name)` | Returns an `RDRequireDeferredReference` handle; loads on first `.load()` / `.onReady()`. Tracks per-handle load timing into `InteractionTracingMetrics` when available. |
| `ifRequired(name, cb, elseCb)` / `ifRequireable` | Conditional access without forcing a load. |

`RequireDeferredReference` also implements `unblock()` — a mechanism that registers a synthetic `rd:<module>` define gated on `RequireDeferredFactoryEvent` (`SUPPORT_DATA`/`CSS`), letting the server unblock deferred modules in batches tied to CSS/support-data arrival.

## 3. Bootloader: fetching what's missing

When `require` hits a module not in the page, **Bootloader** (40 declared dependencies — the most connected module in the prelude) fetches it. Two transport paths:

### 3.1 Resource tags (the normal path)

`BootloaderDocumentInserter` + `BootloaderPreloader` insert `<link rel="preload">`, `<script>`, and `<link rel="stylesheet">` into a `DocumentFragment` batch-appended to `<head>`. Details from the source:

- URLs deduplicated in two sets (preload vs prefetch); a URL is never inserted twice.
- `crossOrigin="anonymous"` unless the resource is marked `nc` (no-CORS).
- `fetchpriority` is settable per resource.
- `d===1` resources skip preloading entirely (the `d` flag = "do not preload").
- CSS load detection: `CSSLoader` uses `link.onload` where supported (feature-detected by loading a `data:text/css;base64,` URL and watching whether `onload` fires), falling back to **`CSSPoller`** — polls `link.sheet.cssRules` every 20 ms; access throws on CORS-blocked sheets, which is exactly how it detects cross-origin failure, and gives up after `CSSLoaderConfig.timeout` with an FBLogger warning including the href and CORS setting.

### 3.2 The bootloader endpoint (the interesting path)

`BootloaderEndpoint` batches missing modules into a **single GET** against `BootloaderEndpointConfig.endpointURI`:

```js
// de-minified request construction
function buildURL(blocking, nonblocking) {
  var params = {};
  if (blocking.size)    params.modules    = joinKeys(blocking);     // "Name1,Name2,..."
  if (nonblocking.size) params.nb_modules = joinKeys(nonblocking);
  var query = Object.entries(Object.assign(params, getAsyncParams("GET")))
    .map(([k, v]) => encodeURIComponent(k) + "=" + encodeURIComponent(String(v)))
    .join("&");
  return endpointURI + (endpointURI.includes("?") ? "&" : "?") + query;
}
```

A request for *code* carries the full `getAsyncParams` payload from Part 2 — **`fb_dtsg_ag`, `__user`, `__rev`, session IDs, Haste bitmaps**. The response is a Haste package (the `;/*FB_PKG_DELIM*/` format this series analyzes), guarded by the `for (;;);` prefix (stripped via `CSRFGuard`), consumed by `HasteResponse`. Payload accounting is tracked per response:

```js
// de-minified payloadStats shape
{ hsdp: { entry, dup_entry },                    // define payloads
  hblp: { rsrc, dup_rsrc, comp, dup_comp },      // resource map
  sjsp: { define, dup_user_define, dup_system_define, require } }
```

Duplicates are counted, not just accepted — the server gets told when it re-sends things.

Failure handling is first-class: the response header **`error-mid`** (gated by `gkx("18719")`) carries a server-side error identifier; `BootloaderEvents.notifyBootloadEndpointError` broadcasts it per-module, and `JSResourceReferenceImpl` turns it into a thrown error tagged `opes_mids` with metadata entry `("OPES","MID",mid)` — so a broken deploy maps every failed module back to one server-side message ID.

### 3.3 Retry with a circuit breaker

`BootloaderRetryTracker`:

```js
// de-minified
maybeScheduleRetry(src, retryFn, giveUpFn) {
  var attempts = getNumRetriesForSource(src);
  if (!this.stillHealthy() || attempts >= config.retries.length) { giveUpFn(); return; }
  this.attemptTimestamps.push(now());
  this.perSource.set(src, attempts + 1);
  setTimeout(retryFn, config.retries[attempts]);   // per-attempt backoff from config
}

stillHealthy() {
  if (!this.enabled) return false;
  var n = this.attemptTimestamps.length;
  if (n < config.abortNum) return true;
  // if the last abortNum retries all landed within abortTime ms, the network is dead
  if (this.attemptTimestamps[n-1] - this.attemptTimestamps[n - config.abortNum] < config.abortTime) {
    this.enabled = false;
    config.abortCallback();   // e.g. FBLogger("binary_transparency").warn("Translations retry abort")
  }
  return this.enabled;
}
```

Thresholds (`jsRetries`, `jsRetryAbortNum`, `jsRetryAbortTime`, plus separate `translationRetries*` for translations) come from `BootloaderConfig` — server-tunable without a deploy. There's also `maybeRetryImmediately` for zero-delay single retries. The circuit breaker exists so a client on a dead connection fails fast instead of retry-storming Meta's edge.

### 3.4 Translations as a first-class resource

`MakeHasteTranslations` fetches locale data over the same machinery: XHR via `getSameOriginTransport` (so Ghost Owl recovery applies, Part 1), response shape-validated (`{translations: {...}, virtual_modules: [...]}`), with support for `data:application/json;base64` inline translation URIs to save a round trip. `BootloaderUsageLoggerUtils` wraps usage in `Proxy` traps that bump ODS counters (`bootloader_load_modules.<cohort>.module_loaded|module_used`) — Meta measures not just what loaded but what got *touched*.

## 4. Resource scheduling: tiers and deferred loading

Code doesn't just load — it's scheduled:

- **`BootloaderEventsManager`** names the pipeline stages: `rsrcDone:<url>`, `bl:<modules>`, `t1:<m>`, `t2s:<m>`, `t2:<m>`, `t3s:<m>`, `t3:<m>`, plus `*Log` variants and `beDone:<m>`. Tier 1/2/3 correspond to critical / needed-for-interaction / deferred. Other modules register callbacks against these named dependencies via `CallbackDependencyManager`.
- **`CometResourceScheduler`** and **`DeferredJSResourceScheduler`** route Bootloader work through `JSScheduler` (Facebook's Scheduler fork — `Scheduler-dev.classic` / `Scheduler-profiling.classic` ship in the prelude), so module execution yields to rendering.
- **`BootloaderEvents`** exposes the lifecycle as an Arbiter event bus: `bootloader/bootload_started`, `bootloader/bootload`, `bootloader/callback_timeout`, `bootloader/defer_timeout`, `hasteResponse/handle`, `bootloader/resource_in_longtail_bt_manifest`, `bootloader/bootload_error`, `bootloader/bootload_endpoint_error` — all persistent (late subscribers replay missed events).

## 5. Haste bitmaps: telling the server what you already have

The `BitMap`/`HasteBitMap` pair is how the client avoids re-downloading modules. Every executed module index is set in a bitmap; `toCompressedString()` run-length encodes it:

```js
// de-minified from BitMap
var ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ-_";
// first bit + binary run-length encoding of runs, packed 6 bits/char into ALPHABET
```

The result rides on async requests as the `jsmod`-family params (Part 2). The server diffs it against what the page *should* have and ships only the delta — which is why repeat Facebook page loads transfer almost no JS. `HasteResourceIndexUtil.parseResourceIndexes` parses the server-side index lists (`":1,2,3"` → `[1,2,3]`, with `UNKNOWN_RESOURCE_INDEX = 0`).

## 6. ClientConsistency: the stale-client killer

Pages stay open for days; Meta deploys constantly. `ClientConsistency` handles the drift: every response can declare a `rev` and an `actions` map. On revision mismatch against `SiteData.client_revision`, actions are consulted; action **2** emits `softRefresh`, action **3** emits `hardRefresh`, both with reason `multiple_revs`. Multiple revisions can be outstanding (`recordResponseRevision` accumulates them in a Set). If a long-lived facebook.com tab has ever reloaded itself under you — this module did it.

## 7. Prelude module census

The 232 modules in the captured bundle break down as:

| Category | Count | Examples |
|---|---|---|
| JS polyfills | ~45% of bytes | Array/Map/Set/String/Iterator/JSON/URL/Observers |
| Loader + Bootloader + Haste | ~25 | kernel, Bootloader, HasteResponse, endpoint, schedulers |
| Timing/telemetry | ~20 | performanceAbsoluteNow, ResourceTimingsStore, QPL, Hyperion, Banzai |
| Error infrastructure | ~15 | fb-error, ErrorGuard, ErrorPubSub, ErrorSerializer |
| URI/query machinery | ~12 | URI, URIBase, URISchemes, PHP serializers |
| GHL anti-tamper | ~10 | Part 1 |
| Tokens/request assembly | ~10 | Part 2 |
| Scheduling/timers | ~15 | JSScheduler, setImmediate*, TimeSlice shims |
| Network state | ~5 | NetworkStatus, NetworkHeartbeat |
| Misc | remainder | CookieConsent, WebStorage, Visibility, React DOM server glue |

The full annotated index of all 232 modules is Part 7 of this series.

---

*Next: the 150-entry custom URL scheme allowlist hiding in the URI layer.*
