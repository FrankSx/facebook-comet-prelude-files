# Appendix: The Complete Annotated Module Index

*Part 7 of The Facebook Comet Prelude Files · frankSx · frankhacks.blogspot.com*

---

This appendix is the reference table for the captured Comet prelude bundle (326,001 bytes, SHA-256 `57d60f40f636669c259ec6f945ae4264ac4b55cba01a5e767affaad900e69ec4`): **all 232 `__d` modules**, categorized, with dependency counts and one-line functional descriptions. Dependency counts are the length of each module's declared dependency array — a rough but honest complexity metric. `cr:NNNN` references are conditional-require placeholders resolved to real modules at build time; the numeric IDs are verbatim from production.

Use this as the map when doing your own analysis: every module name greps cleanly in the bundle, and the category groupings match the deep-dives in Parts 1–6.

### Loader kernel & module system (48)

| Module | Deps | Function |
|---|---|---|
| `$InternalEnum` | 0 | Frozen reverse-indexed enum factory (+Mirrored variant) |
| `Arbiter` | 7 | Legacy pub/sub with persistent state events and callback dependencies |
| `ArbiterToken` | 1 | Unsubscribe handle for Arbiter subscriptions |
| `BaseEventEmitter` | 5 | Event emitter with guarded dispatch and current-subscription tracking |
| `BitMap` | 0 | RLE-compressed bitmap (6-bit alphabet) for module-state reporting |
| `Bootloader` | 40 | The module fetcher (40 deps): tags, endpoint, tiers, Haste response handling |
| `BootloaderDocumentInserter` | 1 | Batch-inserts resource tags into <head> via DocumentFragment |
| `BootloaderEndpoint` | 15 | Batched module GET endpoint with error-mid handling |
| `BootloaderEvents` | 2 | Bootloader lifecycle event bus (bootload/defer_timeout/endpoint_error...) |
| `BootloaderEventsManager` | 2 | Named-dependency events for pipeline stages (t1/t2/t3, beDone, rsrcDone) |
| `BootloaderPreloader` | 6 | Preload/prefetch link insertion from Haste resource maps |
| `BootloaderRetryTracker` | 2 | Per-source retry with global circuit breaker (abortNum/abortTime) |
| `BootloaderUsageLoggerUtils` | 2 | Proxy traps logging module loaded-vs-used to ODS key 454 |
| `CallbackDependencyManager` | 1 | Register callbacks gated on named dependencies being satisfied |
| `CircularBuffer` | 1 | Ring buffer with onEvict callbacks |
| `ClientConsistency` | 4 | Stale-client detection: revision actions 2=soft, 3=hard refresh |
| `ClientConsistencyEventEmitter` | 1 | Bus for softRefresh/hardRefresh/newRevision |
| `CometResourceScheduler` | 2 | Bootloader work via Scheduler |
| `DeferredJSResourceScheduler` | 3 | Deferred module scheduling |
| `EmitterSubscription` | 1 | Subscription record binding listener+context |
| `EventEmitter` | 1 | BaseEventEmitter subclass |
| `EventEmitterWithHolding` | 0 | Emitter with event holding/replay for late subscribers |
| `EventHolder` | 1 | Held-event store for EventEmitterWithHolding |
| `EventSubscription` | 0 | remove() handle for emitter subscriptions |
| `EventSubscriptionVendor` | 1 | Per-event-type subscription registry |
| `HasteBitMap` | 1 | Named BitMap registry |
| `HasteBitMapName` | 0 | Bitmap name registry |
| `HasteResourceIndexUtil` | 1 | Parse server resource index lists |
| `HasteResponse` | 11 | Haste package consumer: defines, resources, ServerJS, consistency |
| `HasteSupportData` | 8 | Support-data bundle (ix/bx/gkx/qex/justknobx/Falco policy) |
| `JSResourceEvents` | 1 | Rolling 50-event per-module load/access log |
| `JSResourceReferenceImpl` | 8 | Module load promises; error-mid -> opes_mids failure mapping |
| `MakeHasteTranslations` | 12 | Locale fetching with retry, data-URI support, shape validation |
| `MakeHasteTranslationsMap` | 1 | Translation/virtual-module registry |
| `MetaConfigMap` | 0 | Server config map |
| `QPLHasteSupportDataStorage` | 0 | QPL storage for Haste support data |
| `RDRequireDeferredReference` | 1 | SSR-aware variant of RequireDeferredReference |
| `RequireDeferredFactoryEvent` | 1 | Enum: SUPPORT_DATA / CSS unblock events |
| `RequireDeferredReference` | 8 | Deferred-module handle: load/onReady/unblock, QPL load timing |
| `ResourceHasher` | 1 | Resource name hashing |
| `ResourceTimingsStore` | 5 | Per-type ring buffers (1000) of resource UIDs, timings, annotations |
| `ResourceTypes` | 0 | Enum: js/css/xhr |
| `SimpleHook` | 0 | Minimal multi-callback hook |
| `ifRequireable` | 1 | ifRequired wrapper |
| `ifRequired` | 0 | Conditional module access without forcing load |
| `requireCond` | 0 | Compile-time conditional require placeholder |
| `requireDeferred` | 1 | Factory/cache for RDRequireDeferredReference handles |
| `requireWeak` | 0 | Callback only if module already loaded; never fetches |

### Anti-tamper (Ghost Owl) (6)

| Module | Deps | Function |
|---|---|---|
| `GHLDetectionUtils` | 2 | Ghost Owl detection matrix (JSON.parse/XHR/String/call/fillText) |
| `GHLDetectionUtilsPreludeSafe` | 2 | Ghost Owl primitives: normalization, iframe clean-realm, native restore, decoy payload |
| `GHLNetworkLayer` | 6 | getGHLXhr: prototype walking + clean-realm XHR recovery |
| `GHLServerJSParse` | 5 | Defensive ServerJS payload parser (4-strategy fallback chain) |
| `GHLTypenameRestore` | 0 | Repairs __typename values stripped by content filters |
| `getSameOriginTransport` | 9 | Choke point for all same-origin XHR with GHL recovery |

### Tokens & request assembly (24)

| Module | Deps | Function |
|---|---|---|
| `Alea` | 0 | Alea PRNG |
| `CSRFGuard` | 0 | for (;;); prefix strip/check |
| `CookieConsent` | 1 | Initial cookie-consent state |
| `CurrentCanonicalRoute` | 2 | Canonical route for logging attribution |
| `DTSG` | 2 | fb_dtsg token holder (get/set/rotate) |
| `DTSGUtils` | 8 | jazoest derivation + token-attachment domain allowlist |
| `DTSG_ASYNC` | 1 | Comet async-tier DTSG channel |
| `Random` | 2 | ServerNonce-seeded Alea random |
| `StaticSiteData` | 0 | Dynamic parameter key names (hs_key, dpr_key, jsmod_key...) |
| `WebSession` | 4 | __s session IDs: 6-char base-36, localStorage 'Session', strict validation |
| `WebSessionDefaultTimeoutMs` | 0 | Session TTL constant |
| `WebStorage` | 5 | localStorage/sessionStorage safe accessors (consent-aware) |
| `asyncParams` | 0 | Extra async params registry |
| `bootstrapWebSession` | 3 | Session bootstrap at navigation start |
| `currentCometRouterInstance` | 0 | Router instance holder |
| `getAsyncParams` | 25 | The request-parameter assembler (Part 2) |
| `getAsyncParamsForProfiling` | 1 | Profiling params |
| `getAsyncParamsFromCurrentPageURI` | 0 | Params from window.location |
| `getTopMostRoute` | 1 | Top route |
| `getTopMostRouteInfo` | 1 | Top route info |
| `isQuotaExceededError` | 0 | Storage quota error detection |
| `isSocialPlugin` | 2 | Social-plugin iframe detection |
| `pushViewToRouteInfo` | 0 | Route info stack push |
| `uniqueRequestID` | 0 | __req incrementing counter |

### URI & query (21)

| Module | Deps | Function |
|---|---|---|
| `BaseDeserializePHPQueryData` | 0 | PHP-style query parser with __proto__/hasOwnProperty pollution defense |
| `PHPQuerySerializer` | 2 | PHP query serialize/deserialize with [] passthrough |
| `PHPQuerySerializerNoEncoding` | 2 | Unencoded variant |
| `PHPStrictQuerySerializer` | 2 | Fully-encoded variant |
| `URI` | 14 | Full URI facade: navigation, qualified/unqualified, registered domain |
| `URIAbstractBase` | 8 | URI parser/core: scheme allowlist, userinfo ban, Unicode-lookalike host denylist |
| `URIBase` | 7 | URI with qualification, subdomain checks, raw-query handling |
| `URIRFC3986` | 0 | RFC 3986 URI regex parser |
| `URISchemes` | 1 | The 150-scheme allowlist (Part 4) |
| `UriNeedRawQuerySVChecker` | 3 | Domains needing unencoded query serialization |
| `flattenPHPQueryData` | 1 | Object -> PHP bracket-notation query flattening |
| `isCdnURI` | 0 | CDN domain test (token exclusion) |
| `isFacebookURI` | 0 | facebook.com/workplace domain test |
| `isInstagramURI` | 0 | instagram.com test |
| `isMessengerDotComURI` | 0 | messenger.com test |
| `isMetaDotComURI` | 0 | meta.com test |
| `isOculusDotComURI` | 0 | oculus.com test |
| `isSameOrigin` | 0 | Origin comparison |
| `isWorkplaceDotComURI` | 0 | workplace.com test |
| `setHostSubdomain` | 0 | Swap hostname subdomain |
| `unqualifyURI` | 0 | Strip protocol/domain/port |

### ServerJS pipeline (23)

| Module | Deps | Function |
|---|---|---|
| `CometPrelude` | 2 | Prelude entry: critical + runWhenReady |
| `CometPreludeCritical` | 22 | Prelude stage 1: bootloader + scheduler + DTSG wiring |
| `CometPreludeCriticalRequireConds` | 18 | Wires requireCond for the prelude critical path |
| `CometPreludeRunWhenReady` | 4 | Prelude stage 2: listeners + visibility |
| `ContextualComponent` | 1 | __cond_* container binding for streamed instructions |
| `Parent` | 1 | DOM parent binding for contextual components |
| `ReloadPage` | 2 | location.reload with CQuick iframe compat path |
| `Run` | 1 | Run behavior (cr:310) |
| `RunComet` | 8 | Comet onLeave/run behavior |
| `RunWWW` | 1 | WWW run behavior (cr:925100) |
| `ScheduledApplyEach` | 1 | Scheduled per-item application |
| `ScheduledServerJS` | 3 | ServerJS via JSScheduler |
| `ScheduledServerJSDefine` | 2 | ServerJSDefine via JSScheduler |
| `ScheduledServerJSWithCSS` | 3 | ServerJS coordinated with CSS arrival |
| `ServerJS` | 10 | The streaming-payload execution engine (Part 5) |
| `ServerJSDefine` | 2 | Server-side __d registration + loaded-module hash (jsmod) |
| `ServerJSPayloadListener` | 5 | Legacy data-sjs listener (GHL parse path) |
| `ServerJSPayloadListener_NEW` | 4 | New data-sjs listener (plain JSON.parse, gated) |
| `ServerJsRuntimeEnvironment` | 1 | Per-payload ServerJS runtime flags |
| `__debug` | 0 | Debug hooks surface |
| `ge` | 0 | Element-by-id helper |
| `nowServerJS` | 0 | Immediate ServerJS execution escape hatch |
| `replaceTransportMarkers` | 3 | Server marker substitution pre-parse, with exposure logging |

### Error infrastructure (15)

| Module | Deps | Function |
|---|---|---|
| `ErrorGuard` | 1 | Guarded execution wrapper (catch-normalize-report, never propagate) |
| `ErrorGuardState` | 1 | ErrorGuard configuration state |
| `ErrorMetadata` | 1 | Structured error metadata entries (e.g. OPES/MID) |
| `ErrorNormalizeUtils` | 1 | Thrown-value normalization (fb-error re-export) |
| `ErrorPubSub` | 1 | Error report bus (fb-error re-export) |
| `ErrorSerializer` | 1 | Structured-error flattening for transport (fb-error re-export) |
| `ErrorUtils` | 6 | ErrorUtils setup/config surface |
| `FBLogger` | 1 | Categorized leveled logging (warn/mustfix/mustfixThrow) |
| `IntervalTrackingBoundedBuffer` | 3 | CircularBuffer with eviction error reporting |
| `err` | 1 | fb-error err() re-export |
| `fb-error` | 2 | Full structured-error stack: templates, blame opcodes, rate limiting, XFBDebug |
| `fb-error-lite` | 0 | Minimal structured-error factory (messageFormat/messageParams/taalOpcodes) |
| `getErrorSafe` | 1 | Coerce thrown values to Error (fb-error re-export) |
| `invariant` | 3 | Numbered internal assertions with internalfb.com decoder links |
| `sprintf` | 0 | %s string formatting |

### Telemetry & performance (31)

| Module | Deps | Function |
|---|---|---|
| `BanzaiLazyQueue` | 1 | Pre-boot Banzai beacon queue |
| `Hyperion` | 4 | Loader: intercepts window when Env.loadHyperion |
| `NetworkHeartbeat` | 3 | Periodic same-origin health probe via getSameOriginTransport |
| `NetworkStatus` | 3 | Network status facade |
| `NetworkStatusImpl` | 3 | Online/degraded state from heartbeat results |
| `NetworkStatusSham` | 0 | SSR no-op variant |
| `TimingAnnotations` | 0 | Per-request annotation collector (string/set/vector props) |
| `UserTimingUtils` | 3 | performance.mark/measure wrapper with time-origin mapping |
| `UserTimingUtils.shared` | 2 | Shared User Timing helpers |
| `Visibility` | 3 | Document visibility facade |
| `VisibilityListener` | 2 | Visibility time-series recorder (10k buffer) |
| `bx` | 1 | Bloks asset manifest |
| `cx` | 0 | CSS class-name obfuscation map |
| `getFalcoLogPolicy_DO_NOT_USE` | 1 | Falco logging policy |
| `gkx` | 4 | Feature-gate reader with gk2_exposure logging |
| `hyperionCore` | 4 | Property-descriptor interception engine (__ext/__sproto shadow prototypes) |
| `hyperionDOM` | 2 | DOM shadow prototypes (IWindow etc.) |
| `hyperionHook` | 0 | Hyperion extension points |
| `ix` | 1 | Image asset manifest |
| `justknobx` | 1 | Runtime knob reader (server-tunable booleans/ints) |
| `memoizeInstrumentation` | 4 | 1%-sampled memoize cache-growth logging with creation stacks |
| `performance` | 0 | window.performance shim |
| `performanceAbsoluteNow` | 2 | Wall-clock mapping of performance.now() with blur/focus drift correction (500ms) |
| `performanceAbsoluteNowOnAdjust` | 1 | Hook fired on clock-skew adjustment |
| `performanceNavigationStart` | 1 | navigationStart binding |
| `performanceNow` | 1 | Monotonic-ish now() with Date.now() fallback and backward-skew correction |
| `qex` | 2 | Quick-experiment reader with exposure logging |
| `qplAnnotationsIntServerJS` | 0 | QPL int annotations channel |
| `qplAnnotationsStringServerJS` | 0 | QPL string annotations channel |
| `qplTagServerJS` | 0 | QPL tags channel |
| `qplTimingsServerJS` | 3 | Server timing merge into client traces |

### Scheduling & timers (44)

| Module | Deps | Function |
|---|---|---|
| `ImmediateImplementation` | 1 | setImmediate implementation selection |
| `JSScheduler` | 1 | Facebook Scheduler facade |
| `Promise` | 1 | Promise + allSettled/finally polyfills over native or cr:6640 |
| `PromiseAnnotate` | 0 | displayName attach/read for promise debugging |
| `Scheduler-dev.classic` | 1 | Scheduler dev build |
| `Scheduler-profiling.classic` | 1 | Scheduler profiling build |
| `SchedulerFb-Internals_DO_NOT_USE` | 4 | Scheduler internals bridge |
| `SchedulerFeatureFlags` | 0 | Scheduler flags |
| `TimeSlice` | 1 | Execution-context tracking (cr:1126) |
| `TimeSliceSham` | 3 | WWW TimeSlice variant |
| `TimerStorage` | 0 | Timer handle storage |
| `asyncToGeneratorRuntime` | 1 | async/await runtime helper |
| `cancelAnimationFrame` | 1 | cAF export |
| `cancelAnimationFramePolyfill` | 0 | cAF polyfill |
| `clearImmediate` | 1 | clearImmediate export |
| `clearImmediatePolyfill` | 1 | clearImmediate polyfill |
| `clearInterval` | 1 | clearInterval binding |
| `clearIntervalComet` | 1 | Comet clearInterval |
| `clearIntervalWWW` | 1 | WWW clearInterval (cr:1003267) |
| `clearTimeout` | 1 | clearTimeout binding (cr:3725) |
| `clearTimeoutComet` | 1 | Comet clearTimeout |
| `clearTimeoutWWW` | 1 | WWW clearTimeout (cr:806696) |
| `clearTimeoutWWWOrMobile` | 1 | WWW/mobile clearTimeout (cr:7386) |
| `createCancelableFunction` | 1 | Cancelable callback wrapper |
| `nativeRequestAnimationFrame` | 0 | Native rAF binding |
| `promiseDone` | 3 | Terminate promise chains with error reporting |
| `replaceNativeTimer` | 7 | Native timer replacement helper |
| `requestAnimationFrame` | 3 | rAF facade |
| `requestAnimationFrameAcrossTransitions` | 2 | rAF surviving transitions |
| `requestAnimationFramePolyfill` | 3 | rAF polyfill |
| `setImmediateAcrossTransitions` | 2 | setImmediate surviving transitions |
| `setImmediatePolyfill` | 3 | setImmediate polyfill |
| `setInterval` | 1 | setInterval binding |
| `setIntervalAcrossTransitions` | 1 | setInterval surviving page transitions (cr:7389) |
| `setIntervalAcrossTransitionsWWW` | 1 | WWW interval across transitions (cr:896462) |
| `setIntervalComet` | 2 | Comet setInterval |
| `setIntervalWWW` | 1 | WWW setInterval (cr:896461) |
| `setTimeout` | 1 | setTimeout binding (cr:4344) |
| `setTimeoutAcrossTransitions` | 1 | setTimeout surviving transitions (cr:7391) |
| `setTimeoutAcrossTransitionsWWW` | 1 | WWW timeout across transitions (cr:986633) |
| `setTimeoutComet` | 2 | Comet setTimeout |
| `setTimeoutCometInternals` | 1 | Comet timer internals via JSScheduler |
| `setTimeoutWWW` | 1 | WWW setTimeout (cr:807042) |
| `setTimeoutWWWOrMobile` | 1 | WWW/mobile setTimeout (cr:7390) |

### Misc utilities (20)

| Module | Deps | Function |
|---|---|---|
| `CSSCore` | 1 | className manipulation |
| `CSSLoader` | 5 | Stylesheet loading with onload detection + CSSPoller fallback |
| `CSSPoller` | 6 | 20ms cssRules polling fallback for CSS load detection |
| `Env` | 0 | Server-hydrated environment flags (ajaxpipe_token, loadHyperion, sjsListenerNew, GHL gate keys) |
| `ExecutionEnvironment` | 0 | Environment capability sniffing (canUseDOM, isInWorker, isInBrowser) |
| `ReactDOMServerExternalRuntime` | 2 | React DOM server runtime glue |
| `emptyFunction` | 0 | Shared no-op and constant-returning functions |
| `hyperionGlobals` | 0 | Hyperion assertions/globals |
| `isEmpty` | 1 | Emptiness check for objects/arrays/iterables |
| `json5` | 0 | Vendored JSON5 parser (Ghost Owl fallback) |
| `maybeDisableAnimations` | 2 | Apply animation disabling |
| `memoize` | 1 | One-shot lazy initializer |
| `memoizeStringOnly` | 1 | String-keyed memoize with instrumentation hooks |
| `nullthrows` | 0 | Throw on null/undefined |
| `objectKeys` | 0 | Object.keys alias |
| `objectValues` | 0 | Object.values alias |
| `removeFromArray` | 0 | splice-by-value helper |
| `shouldDisableAnimations` | 0 | Animation-disable check |
| `unexpectedUseInComet` | 2 | WWW-only API misuse logger |
| `unstable_server-external-runtime-exp` | 1 | React server external runtime experiment |

## Appendix B: The verbatim URISchemes set

The complete 150-entry set literal from `URISchemes`, in production order:

```
about accountscenter aidemos aistudio apk blob barcelona cmms fb fba fbatwork
fb-ama fb-internal fb-workchat fb-workchat-secure fb-messenger fb-messenger-public
fb-messenger-group-thread fb-page-messages fb-pma fbagenthome fbcf fbconnect
fbinternal fbmobilehome mobilehome fbrpc file flipper ftp gtalk http https mailto
wss ms-app intent itms itms-apps itms-services lasso market svn+ssh fbstaging tel
sms pebblejs sftp whatsapp moments flash fblite chrome-extension webcal instagram
iglite fb124024574287414 fb124024574287414rc fb124024574287414master
fb1576585912599779 fb929757330408142 designpack fbpixelcloud fbapi20130214
fb1196383223757595 tbauth oculus oculus.store oculus.feed fb1680871178595114
fb1543576032349914 fb1635404796768116 fb147781309031234 oculusstore socialplatform
odh com.oculus.rd aria skype ms-windows-store callto messenger workchat
fb236786383180508 fb1775440806014337 data fb-mk munki origami-file
fb-nimble-vrsrecorder fb-nimble-monohandtrackingvis together togetherbl
horizonlauncher horizon venues whatsapp-consumer whatsapp-smb fb-ide-opener
fb-vscode fb-vscode-insiders fb-vscode-dev editor spark-studio spark-player
spark-simulator meta-spatial-editor cosmo-player arstudio manifold origami-internal
origami-public stella mwa mattermost logaggregator pcoip cinema home
oculus360photos systemux content moonstone hsr-asset-viewer upi phonepe gpay tez
paytmmp bhim q4bconfigurator q4bnux fb-viewapp meta-ai vibes x-safari-https
meta-bloks facebook-horizon fbboost ctrl-hub ctrl-launcher hsr-editor assethub
gizmo aura hatch basel lantern fb-owl glam
```

## Appendix C: Conditional-require (cr:) IDs observed

```
cr:310      cr:1078     cr:1080     cr:1126     cr:3725     cr:4344
cr:6640     cr:7385     cr:7386     cr:7388     cr:7389     cr:7390
cr:7391     cr:8959     cr:8960     cr:3725     cr:806696   cr:807042
cr:896461   cr:896462   cr:925100   cr:986633   cr:1003267  cr:696703
```

Of particular interest: `cr:8959` = POST DTSG token provider, `cr:8960` = GET (`fb_dtsg_ag`) token provider (Part 2, §2).

## Appendix D: Gate inventory

**gkx (feature gates):** 999, 1174, 5023, 5163, 5415, 7742, 8068, 8869, 9063, 11588, 15745, 18719, 20935, 20936, 23984, 25201

**justknobx (runtime knobs):** 340, 1590, 2694, 3323, 4216, 5589, 5685

**window.Env GHL gate keys (8-char, rotated per deploy):** h4npx7qw, p9fk3wmn, f2yq8vnd, t5nd8vqc, m8r3kp6w, b3xk8fqm, q4v7nx3k, r7c2m9xk, z2ht6xqp, k7q3nv9d, w6jt4rnq, w8kq3zmt, b7xr2qnf (iframe insertion paths); v9k2mt7q, d3hf9km2, k8pq2mnb, n5tq2wjb, c6mw9qtk, j6dw4ztx, x8kf2pw6, r4wt7kmj (parse-strategy arming).

---

*End of series. Verify everything against the bundle hash; diff future captures against this index to watch the system evolve. — frankSx*
