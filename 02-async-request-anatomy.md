# Anatomy of a Facebook Async Request: DTSG, LSD, jazoest, and getAsyncParams

*Part 2 of The Facebook Comet Prelude Files · frankSx · frankhacks.blogspot.com*

---

Every authenticated request Facebook's web client makes — every GraphQL query, every like, every async navigation — carries a payload of parameters that must be exactly right or the server rejects it. The assembly logic lives in one prelude module: **`getAsyncParams`**, with 25 declared dependencies. De-minified, it is the most complete public map of Facebook's client-side request signing in existence.

This post walks the whole thing: the parameter map, the three-token system, the session machinery underneath, and the server-side enforcement signals visible from the client.

## 1. The parameter map

`getAsyncParams(method, skipJsmod)` builds the following, in order. (Parameter names marked *dynamic* have their **key name itself** supplied by server config via `StaticSiteData` — Meta can rename the parameter without shipping client code.)

| Parameter | Source | Notes |
|---|---|---|
| `__user` | `CurrentUserInitialData.USER_ID` | Numeric user ID. `0` when logged out. |
| `__a` | constant `1` | "This is an async request." Present on everything. |
| `__req` | `uniqueRequestID()` | Per-page-load incrementing counter (base-36). Server uses it for dedup/ordering. |
| *dynamic* (`hs_key`) | `SiteData.haste_session` | Haste session token. |
| *dynamic* (`dpr_key`) | `SiteData.pr` | Device pixel ratio. |
| *dynamic* (`connection_class_server_guess_key`) | `WebConnectionClassServerGuess.connectionClass` | Guessed connection quality (EXCELLENT/GOOD/MODERATE/POOR). |
| `__rev` | `SiteData.client_revision` | Client build revision. Drives `ClientConsistency` (Part 3). |
| `__s` | `WebSession.getId()` | Tab-session ID, format `xxxxxx:xxxxxx:n` (§3). |
| *dynamic* (`haste_session_id_key`) | `SiteData.hsi` | Haste session ID. |
| *dynamic* (`jsmod_key`) | `ServerJSDefine.getLoadedModuleHash()` | Compressed record of executed modules. Omitted when `skipJsmod` is set. |
| one per `HasteBitMapName` | `HasteBitMap.toCompressedString(name)` | Compressed module-state bitmaps (Part 3). |
| *dynamic* (`comet_key`) | `SiteData.comet_env` | Comet environment flag; omitted for wbloks, defaults to 1. |
| persisted params | `CometPersistQueryParams.relative` | URL query params that persist across async navigations. |
| URI-derived | `getAsyncParamsFromCurrentPageURI()` | Params extracted from `window.location`. |
| profiling | `getAsyncParamsForProfiling()` | Only when profiling is active. |
| `cquick`, `ctarget`, `cquick_token` | `Env.iframeKey` / `iframeTarget` / `iframeToken` | Only in CQuick iframe embedding contexts. |
| `__sp` | constant `1` | Only inside a social plugin iframe (`isSocialPlugin()`). |
| *dynamic* spin keys ×4 | `SiteData.spin_*` | Server-image (spin) builds only: revision, branch, time, mhenv. |
| *dynamic* error weight | `JSErrorLoggingConfig.sampleWeight` | When configured. |
| *dynamic* (`canonical_route_key`) | `CurrentCanonicalRoute.getForGlobalLoggingOnly()` | Logging attribution. |
| `qpl_active_flow_ids` | `QPLUserFlow.getActiveFlowIDs()` | Comma-joined sorted active perf-trace flow IDs, via `requireWeak` (absent if QPL isn't loaded). |
| PWA version | `MessengerPWAVersionForUserAgent` | Via `requireWeak`; Messenger PWA contexts only. |

Then the security tokens, which depend on HTTP method.

## 2. The three-token system

### 2.1 `fb_dtsg` (POST) — the CSRF token

For POST requests, the token comes from the `DTSG` module, hydrated server-side into `DTSGInitialData`:

```js
// de-minified from DTSG
var token = DTSGInitialData.token || null;
function getToken()      { return token; }
function setToken(t)     { token = t; }        // rotated by later responses
function refresh()       { invariant(0, 5809); }   // must never run in WWW
function setTokenConfig(){ invariant(0, 73819); }  // Comet-only path
```

Attached as `fb_dtsg`. The token is mutable — responses can rotate it mid-session via `setToken`, so a replaying client must re-read it from responses, not cache it from page load.

(There is also a `DTSG_ASYNC` module backed by `DTSGInitData` — a separate token channel for Comet's async tier, kept distinct from the WWW one.)

### 2.2 jazoest — the checksum

If `SprinkleConfig.param_name` is set (on facebook.com its value is **`jazoest`**), a second parameter is computed from the token:

```js
// de-minified from DTSGUtils
function getNumericValue(token) {
  var sum = 0;
  for (var i = 0; i < token.length; i++) sum += token.charCodeAt(i);
  var s = sum.toString();
  return SprinkleConfig.should_randomize ? s : SprinkleConfig.version + s;
}
```

**jazoest is the sum of the char codes of the DTSG token, prefixed with a config version digit** (historically `2`, producing values like `21098`). It is a checksum, not a secret — trivially computable from `fb_dtsg`, which is the point: the server can detect token corruption or truncation in transit without crypto. `should_randomize` exists to vary the format per-cohort, and the `jazoest` name itself is an obfuscation-by-rename artifact — "jazoest" is essentially a Caesar-shifted placeholder that stuck.

### 2.3 `fb_dtsg_ag` (GET) — the separate GET token

GET requests don't use `fb_dtsg`. They use a **different token** from a different conditional-require module (`cr:8960` vs POST's `cr:8959`), attached as `fb_dtsg_ag` ("ag" = async GET), with its own jazoest derived the same way:

```js
// de-minified from getAsyncParams
if (method == "POST") {
  var t = cr8959.getCachedToken ? cr8959.getCachedToken() : cr8959.getToken();
  if (t != null && t !== "") {
    d.fb_dtsg = t;
    if (SprinkleConfig.param_name) d[SprinkleConfig.param_name] = DTSGUtils.getNumericValue(t);
  }
  if (LSD.token != null && LSD.token !== "") {
    d.lsd = LSD.token;
    if (SprinkleConfig.param_name && (t == null || t === ""))
      d[SprinkleConfig.param_name] = DTSGUtils.getNumericValue(LSD.token);
  }
}
if (method == "GET") {
  var ag = cr8960.getCachedToken ? cr8960.getCachedToken() : cr8960.getToken();
  if (ag != null && ag !== "") {
    d.fb_dtsg_ag = ag;
    if (SprinkleConfig.param_name) d[SprinkleConfig.param_name] = DTSGUtils.getNumericValue(ag);
  }
}
```

Consequences:

- Stealing a POST token doesn't let you forge GETs and vice versa.
- Token rotation happens independently per method.
- `getCachedToken` is preferred over `getToken` — the client avoids forcing a token fetch on the hot path.

### 2.4 `lsd` — the pre-auth fallback

The `LSD` module holds a third token, attached as `lsd` when present. Notably, if there's no DTSG token but jazoest is configured, the checksum is computed from the **LSD token instead** — so logged-out and early-page-load requests still carry a valid checksum. LSD is also the token used by login and other pre-auth forms, which is why it appears in decades of scraping folklore.

## 3. WebSession: the `__s` parameter

The `__s` parameter deserves its own section because its construction is fully visible in the prelude:

```js
// de-minified from WebSession
var BASE = 36, ID_LEN = 6, SPACE = Math.pow(36, 6);   // ~2.18 billion

function generateId() {
  var n = Math.floor(Random.random() * SPACE);        // Random = Alea PRNG seeded by ServerNonce
  var s = n.toString(36);
  return "0".repeat(ID_LEN - s.length) + s;           // zero-padded 6-char base-36
}
```

- Session records are stored in **localStorage under the key `"Session"`** as `"<id>:<expiryTime>"`, with a sessionStorage fallback for the tab-scoped variant.
- IDs are validated strictly: exactly 6 chars, `/^[a-z0-9]+$/`, expiry must parse as an integer. Any deviation logs a `web_session` warning and discards the record.
- The PRNG is **Alea** (the `Alea` module ships in the prelude), seeded by a server-provided `ServerNonce` — so session IDs are server-seeded, not `Math.random()`-predictable in the naive sense.
- Expired sessions are regenerated transparently; `WebSessionDefaultTimeoutMs` controls the TTL.

The full `__s` format observed in requests (`xxxxxx:xxxxxx:n`) combines the persistent session ID, the tab ID, and a per-tab page counter.

## 4. When are tokens attached at all?

`DTSGUtils.shouldAppendToken(uri)` gates whether tokens go on the wire for a given target:

```js
// de-minified
function shouldAppendToken(uri) {
  return !isCdnURI(uri)
    && !uri.isSubdomainOfDomain("fbsbx.com")
    && (isFacebookURI(uri) || isInstagramURI(uri) || isMessengerDotComURI(uri)
        || isMetaDotComURI(uri) || isWorkplaceDotComURI(uri) || isOculusDotComURI(uri)
        || uri.isSubdomainOfDomain("freebasics.com")
        || uri.isSubdomainOfDomain("discoverapp.com"));
}
```

Two things worth noting:

1. **Explicit CDN and `fbsbx.com` exclusion.** `fbsbx.com` is Facebook's sandboxed domain for untrusted content (attachment hosting, iframe isolation). No tokens ever leak there — a deliberate anti-exfiltration control. If you're hunting for token-leak primitives, this function *is* the boundary: any gadget that makes `isCdnURI` false and the property checks true for an attacker-controlled host is a token exfiltration bug.
2. **The allowlist spans six Meta properties plus two zero-rating domains** (`freebasics.com`, `discoverapp.com` — the Free Basics program). Any subdomain of any of these receives your CSRF token. Subdomain takeover or XSS on *any* Meta-property subdomain is therefore a token-theft primitive.

## 5. The `for (;;);` guard

Responses to these requests are protected by the complementary mechanism in `CSRFGuard` — the entire module, de-minified:

```js
var PREFIX = "for (;;);";
var regex  = /^for ?\(;;\);/;
function exists(responseText) { return !!responseText.match(regex); }
function clean(responseText) {
  var m = responseText.match(regex);
  return m && responseText.substr(m[0].length);
}
exports = { regex, length: PREFIX.length, exists, clean };
```

Every JSON response is prefixed with `for (;;);` — an infinite loop. If a third-party site includes the endpoint via `<script src>` (the classic JSON-hijacking vector), the attacker's page hangs instead of executing the data. Facebook's XHR layer strips the prefix before parsing. Note the regex tolerates `for(;;);` without the space — both forms occur in the wild. A 20-year-old technique, still there, still mandatory.

## 6. Server-side enforcement signals

The client code reveals what the server checks:

- **`__rev` mismatch** → consistency actions, not errors. `ClientConsistency` distinguishes action 2 (`softRefresh` — reload resources in place) from action 3 (`hardRefresh` — full page reload), both with reason `multiple_revs`. A replaying client that ignores these slowly desynchronizes: responses arrive referencing modules it doesn't have.
- **`__req` reuse** → dedup. The counter exists so the server can drop duplicate deliveries from retry storms.
- **Missing jazoest with present token** → cheap rejection. It's the first thing a server can verify with zero state.
- **`hsi`/`haste_session` mismatch** → the server can't compute the module delta, so it over-sends — visible as bloated Haste responses.

## 7. Forging requests: the complete checklist

For tooling that replays Facebook async requests (archival, automation, research):

1. **Required:** `__user`, `__a=1`, `__req` (incrementing), `__rev` (must match a *live* client revision — extract from any fresh page load), `hsi`, `__s` (any well-formed `6char:6char:n` value survives basic validation), `fb_dtsg` + `jazoest` for POST **or** `fb_dtsg_ag` + jazoest for GET, `lsd`.
2. **Compute jazoest, never hardcode it:** `str(SPRINKLE_VERSION) + str(sum(map(ord, token)))`.
3. **Strip `for (;;);`** (both spellings) before parsing any response.
4. **Honor consistency actions** — on `softRefresh`/`hardRefresh` signals, re-pull the page and re-extract tokens/revision.
5. **Re-read `fb_dtsg` from responses** — mid-session rotation is normal.
6. **Stay inside the token-attachment allowlist** — requests to CDN or `fbsbx.com` origins will (correctly) carry no token, and adding one there is itself an anomaly.
7. **Expect per-cohort variation** — `SprinkleConfig.should_randomize`, dynamic key names, and gate-controlled behaviors mean two accounts can legitimately see different parameter shapes.

---

*Next: the module system that delivers all of this code — the __d registry and the bootloader.*
