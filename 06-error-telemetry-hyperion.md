# The Observation Deck: Facebook's Error, Telemetry, and Instrumentation Stack

*Part 6 of The Facebook Comet Prelude Files · frankSx · frankhacks.blogspot.com*

---

Everything described in Parts 1–5 is watched. The prelude carries a complete client-side observability stack: structured errors, sampled logging, performance tracing, network health monitoring, and — the centerpiece — **Hyperion**, Meta's property-descriptor interception engine that instruments the DOM itself. This post de-minifies the stack. If you research Facebook's client (or build your own telemetry), this is the layer that sees you.

## 1. fb-error: the structured error protocol

Facebook errors are not strings. The `fb-error` module defines a wire protocol for exceptions:

```js
// de-minified from fb-error / fb-error-lite
function err(format, ...params) {
  var e = new Error(format);
  e.messageFormat = format;                 // the template: "Loading %s failed for %s"
  e.messageParams = params.map(String);     // the values
  e.taalOpcodes = [TAALOpcode.PREVIOUS_FRAME];  // blame attribution
  return e;
}

TAALOpcode = { PREVIOUS_FILE: 1, PREVIOUS_FRAME: 2, PREVIOUS_DIR: 3, FORCED_KEY: 4 };
```

The displayed message is a template; the **format and params travel separately**, so server-side aggregation groups by template (stable across deploys) instead of by rendered string (unstable). `taalOpcodes` encode blame shifting — `PREVIOUS_FRAME` says "attribute this to my caller," which is how wrapper utilities avoid owning errors they re-throw. `FORCED_KEY` overrides the grouping hash entirely (used by the bootloader endpoint errors from Part 3: `forcedKey = module + ":" + errorMID`).

Additional structure visible in the module:

- **`RE_EXN_ID`** — thrown ReScript (OCaml-derived) exceptions are detected by a marker property and serialized with `JSON.stringify`, confirming ReScript code ships in Meta's client.
- **Non-Error thrown values are typed**: promises (`"Promise thrown: %s"`), non-extensible objects, objects with non-string `messageFormat` — each gets a distinct template, so throw-discipline violations are individually countable.
- **Rate limiting by hash**: `shouldLog` keeps a per-hash sliding window — **6 logs per 60 seconds**, entries evicted after 10 minutes, drops counted (`dropped`) and reported on the next accepted log. Identical errors can't flood the pipe; the flood itself becomes a number.
- **`ErrorXFBDebug.addFromXHR(xhr)`** — failed XHRs contribute their `X-FB-Debug` response headers to the error context, tying client errors to server-side request traces.

`ErrorGuard.applyWithGuard` is the execution wrapper used everywhere (module factories, event emitters, ServerJS instructions): run the function, catch, normalize via `ErrorNormalizeUtils`, report through `ErrorPubSub`, don't let it propagate. `ErrorSerializer` then flattens the structured error for transport.

## 2. FBLogger: categorized, leveled logging

`FBLogger(category)` returns a logger scoped to a dot-namespaced category — the categories themselves are a map of the codebase's subsystems. Observed in this bundle alone:

```
ad_blocker_defense_ghost_owl   bootloader_load_modules   client_consistency
comet_infra                    CSSPoller (via category)  ghl
memoize_instrumentation        serverjs_listener         web_session
binary_transparency            ...
```

Levels include `warn`, `debug`, `mustfix`, `mustfixThrow` — `mustfix` pages an on-call internally. When Ghost Owl's json5 fallback fails, that's a `mustfix`. The category strings are verbatim in the bundle and make excellent grep anchors for your own analysis.

## 3. QPL and UserTimingUtils: performance tracing

Quick Performance Log (QPL) support ships in the prelude as four ServerJS-consumed channels (`qplTimingsServerJS`, `qplAnnotationsIntServerJS`, `qplAnnotationsStringServerJS`, `qplTagServerJS`) plus client-side pieces:

- **`UserTimingUtils`** — wraps the User Timing API (`performance.mark`/`measure`), translating server timestamps into the client's time origin via `performanceAbsoluteNow`.
- **`qpl_active_flow_ids`** — Part 2 showed this riding on every async request: active trace flows are joined server-side, so a slow request can be decomposed into "which product flows were active."
- **`performanceAbsoluteNow`** — a monotonic-ish clock with drift correction: it compares `performance.now()` against `Date.now()` on window blur/focus, and when skew exceeds **500 ms** it adjusts and fires `performanceAbsoluteNowOnAdjust` hooks. Sleep/wake cycles don't corrupt traces.
- **`ResourceTimingsStore`** — per-resource-type circular buffers (1,000 entries each for js/css/xhr) with UID assignment, requestSent/responseReceived stamps, and per-request `TimingAnnotations` (string/set/vector props) that are prepared and attached at send time.

## 4. Banzai: the transport

Banzai (Meta's batched beacon transport) appears in the prelude as `BanzaiLazyQueue` — a pre-boot queue:

```js
// de-minified
var queue = [], onQueue = new SimpleHook();
function queuePost(route, data, options) { queue.push([route, data, options]); onQueue.call(...); }
function flushQueue() { var q = queue; queue = []; return q; }
```

Modules that run before Banzai proper loads (gate-exposure logging from `gkx`, QEX exposure from `qex`) post into this queue; the full transport drains it later. Two consequences for observers: (1) early events are buffered in page memory, readable before flush; (2) the `gk2_exposure` route means **Meta logs which feature gates your client evaluated** — your gate fingerprint is itself telemetry.

## 5. NetworkHeartbeat and NetworkStatus

`NetworkHeartbeat` (deps: `clearTimeout`, `getSameOriginTransport`, `setTimeout`) is a periodic same-origin probe — and note it goes through `getSameOriginTransport`, so even the health check gets Ghost Owl's clean-XHR treatment. `NetworkStatusImpl` consumes heartbeat results to maintain online/offline/degraded state (with `justknobx` gates), feeding the `NetworkStatus` facade; `NetworkStatusSham` is the SSR/no-op variant. Connection quality feeds back into requests as `ccg` (Part 2) — the loop is closed: the network you have changes the requests the client makes.

## 6. VisibilityListener: attention telemetry

The prelude records tab visibility as a time series:

```js
// de-minified from VisibilityListener
var startSkew = Date.now() - performanceNow();
var buffer = [], MAX = 10000, disabled = false;
// every visibilitychange → {key: eventTime + startSkew, value: isHidden}
```

`getHiddenTime(start, end)` reconstructs exactly how long the tab was hidden inside any window — so engagement metrics are corrected for background tabs, in both directions (hidden time is excluded; a *foreground* event flushes the buffer). Buffer overflow disables collection rather than degrading (`disabled = true`).

## 7. Hyperion: the interception engine

The deepest module in the telemetry stack. `Hyperion` loads only when `Env.loadHyperion === true`, in browser, not in a worker:

```js
// de-minified from Hyperion
if (ExecutionEnvironment.isInBrowser && !ExecutionEnvironment.isInWorker
    && Env.loadHyperion === true) {
  hyperionCore.intercept(global, hyperionDOM.IWindow.IWindowPrototype);
}
```

`hyperionCore` implements **prototype-level property interception**:

```js
// de-minified from hyperionCore
var EXT = "__ext", SPROTO = "__sproto";
var nextId = 0, filters = [];

function findDescriptor(obj, prop) {
  var d;
  while (obj && !d) {
    d = Object.getOwnPropertyDescriptor(obj, prop);
    if (d) d.container = obj;
    obj = Object.getPrototypeOf(obj);
  }
  return d;
}

function copyOwnProperties(src, dst) {          // clone a prototype, preserving descriptors
  for (var name of Object.getOwnPropertyNames(src)) {
    if (!(name in dst)) {
      var desc = Object.getOwnPropertyDescriptor(src, name);
      try { Object.defineProperty(dst, name, desc); } catch (e) {}
    }
  }
  dst.toString = function () { return src.toString() };   // look like the original
  if (hasOwn(src, "valueOf")) dst.valueOf = function () { return src.valueOf() };
  dst.prototype = src.prototype;
  try { Object.defineProperty(dst, "name", Object.getOwnPropertyDescriptor(src, "name")); } catch (e) {}
}

function extend(obj, skipFilter) {
  if (isObjectOrFunction(obj) && !hasOwn(obj, EXT)) {
    var shadow = null;
    for (var i = 0; !shadow && i < filters.length; ++i) shadow = filters[i](obj);
    if (!shadow) shadow = obj[SPROTO];
    if (shadow) {
      var ext = { virtualPropertyValues: {}, shadowPrototype: shadow, id: nextId++ };
      Object.defineProperty(obj, EXT, { value: ext });
      shadow.interceptObject(obj);
    }
  }
  return obj;
}
```

The mechanics:

- **`__sproto` (shadow prototype)**: an object can be assigned a *shadow* — a cloned prototype whose own properties mirror the original (via `copyOwnProperties`, including `toString`/`valueOf`/`name` camouflage) but whose methods are intercepted. `getShadowPrototype` reads it; `setShadowPrototype` installs it.
- **`__ext` (extension record)**: per-object metadata holding `virtualPropertyValues` — properties that exist only for instrumentation, invisible to normal enumeration — plus a unique `id` per intercepted object.
- **Filter chain**: `registerFilter(fn)` lets subsystems decide which objects get intercepted; first matching filter wins.
- **`getVirtualProperty`/`setVirtualProperty`** (`x`/`$` in the minified source): read/write instrumentation-only state on any object without polluting its real shape.

`hyperionDOM` supplies the DOM-specific shadows (`IWindow.IWindowPrototype` etc.), and `hyperionHook`/`hyperionGlobals` provide extension points and assertions. The net effect: **Meta can observe DOM API usage at the property-descriptor level** — attribute reads, method calls, event listener registration — without wrapping functions in a way Ghost Owl-style toString checks would catch. It's the same technique Ghost Owl defends against, turned inward for first-party instrumentation. The offense/defense symmetry is complete: they know exactly how hooking works because they built the best hooker in the stack.

For researchers: `__ext` and `__sproto` properties on DOM objects are the tell that Hyperion is active on your page, and `Env.loadHyperion` is the switch.

## 8. The supporting cast

- **`memoizeInstrumentation`** — samples 1% of memoize instances (gate `justknobx("1590")`), tracks cache growth against thresholds (1k/10k/100k/1M entries), and logs with a **captured creation stack** (top 10 frames) when caches grow unbounded. Memory leaks become attributed events.
- **`JSResourceEvents`** — rolling 50-event-per-module-per-type log of LOADED/PROMISE_RESOLVED/ACCESSED events, queryable by time window: the module-level "who touched what, when."
- **`BootloaderUsageLoggerUtils`** — wraps deferred module args in `Proxy` traps distinguishing `loaded` from `used` (Part 3), bumping ODS entity-key `454` counters per pkg_cohort.
- **`IntervalTrackingBoundedBuffer`** — a CircularBuffer whose evictions are reported to `ErrorPubSub`: even buffer loss is a monitored event.
- **`ClientConsistencyEventEmitter`** — the bus carrying `newEntry`/`softRefresh`/`hardRefresh` (Part 3); also telemetry.

## 9. The meta-lesson

Read together, the stack has a consistent philosophy: **structure everything, sample everything, attribute everything**. Errors carry templates and blame opcodes; logs carry categories; traces carry flow IDs that cross the client/server boundary; gates log their own exposure; even the anti-tamper system reports through the same pipes. When you poke Facebook's client, the interesting question is never "did they see it" — it's which of six systems saw it first, and what counter you just incremented.

---

*Next: the appendix — a complete annotated index of all 232 prelude modules.*
