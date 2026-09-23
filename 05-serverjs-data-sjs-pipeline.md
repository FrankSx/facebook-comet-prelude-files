# ServerJS and the data-sjs Pipeline: How Facebook Streams Its UI

*Part 5 of The Facebook Comet Prelude Files · frankSx · frankhacks.blogspot.com*

---

Facebook pages are not rendered. They are **streamed**. The HTML you receive is a skeleton plus a series of `<script data-sjs>` tags, each containing a JSON payload of instructions: define these modules, load these resources, require these modules with these arguments, put this markup in that container. A client-side engine — **ServerJS** — executes the payloads in dependency order. This post documents the pipeline as implemented in the prelude: wire format, the listener pair, integrity checks, defensive parsing, transport markers, and the scheduling layer.

## 1. The wire format

A data-sjs tag looks structurally like this (shape reconstructed from the parser and consumer code):

```html
<script data-sjs data-content-len="4821">
{"require":[["ModuleName","__cond_1",[],[{"__dr":"markup"}]]],
 "define":[["NewModule",[],null,66]],
 "hsrp":{"hblp":{"rsrcMap":{...}}},
 "qplTimingsServerJS":{...}}
</script>
```

The payload keys the prelude understands, from the modules that consume them:

| Key | Consumer | Meaning |
|---|---|---|
| `define` | `ServerJSDefine` | Register `__d` modules server-side |
| `require` | `ServerJS.handle` | Require-with-args instructions (conditional via `__cond_*` handles) |
| `hsrp` / `hblp` | `HasteResponse` / Bootloader | Haste resource map — which CSS/JS files exist and their metadata |
| `qplTimingsServerJS` | `qplTimingsServerJS` module | Server-side perf timings to merge into client traces |
| `qplAnnotationsIntServerJS` / `qplAnnotationsStringServerJS` / `qplTagServerJS` | QPL | Trace annotations and tags |
| `ix` | `ix` module | Static asset (image) manifest entries |
| `bx` | `bx` module | Bloks (server-driven UI) asset manifest entries |
| `cx` | `cx` module | CSS class-name obfuscation map entries |

`ix`/`bx`/`cx` are worth a sentence each: they're the three "manifest" channels by which the server ships lookup tables (images, Bloks assets, CSS names) that later modules consume by opaque key — which is why Facebook's markup is full of meaningless `x1a2b3c` class names that remap between deploys.

## 2. The listener: two generations, side by side

The prelude ships **two** payload listeners, selected by `window.Env.sjsListenerNew`:

```js
// de-minified from ServerJSPayloadListener (legacy path)
function processPayload(scriptEl) {
  if (!(scriptEl instanceof HTMLScriptElement)) return;
  var declaredLen = scriptEl.dataset.contentLen;
  if (scriptEl.dataset.processed) return;
  if (scriptEl.textContent.length.toString() !== declaredLen) return;  // integrity gate
  scriptEl.dataset.processed = "1";
  var payload = GHLServerJSParse.ghlParseServerJSPayload(scriptEl.textContent);
  if (payload == null)
    throw err("ServerJS payload marked with data-sjs was parsed as null");
  new ServerJS().handle(payload);
}

function process() {
  if (document == null) return;
  var scripts = document.querySelectorAll("script[data-sjs]:not([data-processed])");
  for (var el of scripts) processPayload(el);
}
```

The differences between the generations are exactly the interesting part:

1. **Legacy** parses through `GHLServerJSParse.ghlParseServerJSPayload` — the Ghost Owl defensive parser (Part 1) with shim detection and the json5 fallback.
2. **New** (`ServerJSPayloadListener_NEW`) uses plain `JSON.parse` — but only runs when `window.Env.sjsListenerNew` is set, i.e. when the server has decided this client/cohort doesn't need the GHL treatment.

Both share the **content-length integrity gate**: the server declares `data-content-len`, and the payload executes only if `textContent.length` matches exactly. This defeats in-transit injection and truncation by middleboxes — a payload modified by a proxy or extension fails the length check and is silently skipped. It stays unprocessed rather than erroring, so a later `process()` pass could still pick it up if the DOM is repaired.

Note the re-entrancy design: `process()` is idempotent (the `data-processed="1"` marker) and is meant to be called repeatedly as the streamed document grows — each pass scans the whole document but only touches new tags. There is no MutationObserver here; the streaming parser calls `process()` as chunks arrive.

## 3. Transport markers

The legacy parse path optionally pre-transforms the payload via `replaceTransportMarkers` (gate `k8pq2mnb`). That module:

- Maintains a `Set` of registered marker strings.
- Rewrites occurrences of each marker in the raw payload text before parsing.
- Queues an exposure log through `BanzaiLazyQueue` when markers actually fire.

Transport markers are how the server smuggles values that would be mangled by intermediaries (or by the GHL decoy/filter ecosystem) through the wire format: the server writes a placeholder, the client substitutes the real value post-receipt, pre-parse. It's a checksum-free integrity side-channel — and the exposure logging means Meta knows when intermediaries touched the payload.

## 4. CSRFGuard on the response side

Payloads arriving via XHR (bootloader endpoint, async navigations) rather than inline tags carry the `for (;;);` prefix, stripped by `CSRFGuard.clean` before the same ServerJS machinery handles them (Part 2, §5). One parser, two transports, two different injection defenses: length-gated inline tags, prefix-guarded XHR.

## 5. The execution model: scheduled, guarded, incremental

`ServerJS.handle` doesn't execute instructions synchronously. The prelude wraps it in scheduling layers:

- **`ScheduledServerJS`** — runs `ServerJS.handle` through `JSScheduler` via `ScheduledApplyEach`, so payload execution interleaves with rendering instead of blocking the main thread.
- **`ScheduledServerJSDefine`** — same treatment for `ServerJSDefine` registration.
- **`ScheduledServerJSWithCSS`** — coordinates ServerJS execution with CSS arrival through Bootloader, so markup doesn't execute before its styles exist (no FOUC-by-design).
- **`nowServerJS`** — immediate-execution escape hatch for critical instructions.
- **`ErrorGuard`** — every instruction runs guarded; a throwing `require` instruction is reported without aborting the rest of the payload.
- **`HasteBitMap`** — every executed define/require is recorded in the bitmap that later rides on async requests (Part 3, §5), closing the loop so the server never re-sends what already ran.
- **`ServerJsRuntimeEnvironment`** — carries per-payload runtime flags (e.g. whether usage stats are reported).

`ServerJSDefine` itself does more than register modules: it maintains the **loaded-module hash** (`getLoadedModuleHash()` — the `jsmod` parameter from Part 2) and handles duplicate-definition accounting (`dup_user_define` vs `dup_system_define` in the payload stats), distinguishing user-land redefines from system redefines.

## 6. Observable streaming behavior (for researchers)

Incremental execution has a side effect useful for analysis: **the page is functional in stages**, and each stage's capabilities tell you which payloads have arrived. Practical probes:

- `document.querySelectorAll("script[data-sjs]")` growing over time, correlated with `data-processed` markers, reveals the server's streaming strategy — what it prioritizes (chrome, then feed units, then below-fold), what it defers, and how it reacts to connection class (`ccg` from Part 2 feeds those decisions).
- `data-content-len` vs actual length is a free middlebox detector for your own connection.
- The `__cond_*` handles in `require` instructions resolve through `ContextualComponent`/`Parent` — the mechanism that binds streamed instructions to DOM containers without global selectors.

## 7. Why the design is worth studying

ServerJS is a 15-year-old answer to the problem the industry later re-solved as React Server Components and streaming SSR: ship *instructions*, not markup, and let the client assemble the UI as data arrives. The prelude implementation shows the production scars the new hotness will eventually grow: transport integrity checks, parser-distrust fallbacks, anti-tamper repair passes, scheduling to keep the main thread alive, manifest channels for every asset class, and a feedback loop (Haste bitmaps + loaded-module hash) so the server knows exactly what the client has executed.

---

*Next: the observation deck — Facebook's error, telemetry, and instrumentation stack, including the Hyperion interception engine.*
