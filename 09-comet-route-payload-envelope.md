# The Route Payload Envelope: How Facebook Ships a Page as JSON

*Part 9 of The Facebook Comet Prelude Files · frankSx · frankhacks.blogspot.com*

**Labels:** reverse engineering · Comet router · ServerJS · Haste bitmap · Relay · route definition · JSON envelope · security research
**Permalink:** `/2026/09/comet-route-payload-envelope-facebook-json.html`
**Search description (110 chars):** Reverse engineering Facebook's Comet route payload envelope: route_definition JSON, Haste bitmaps, deferred requires, ServerJS.

---

Parts 1–7 reverse-engineered the *client* machinery: how the prelude parses, boots, signs, and executes. This post opens the other side of the wire. Below is a complete, verbatim-structure Comet **route payload envelope** — the JSON a Facebook page fetch returns for a profile route — annotated piece by piece. It is the missing document for everything in Parts 2, 3, and 5: the server format those client systems exist to consume.

Source: captured September 2026 from a `profile.php` route fetch. Structure reproduced exactly; values are real.

![Banner: exploded translucent glass panels of a JSON envelope, connected by luminous data streams](assets/post09-banner-route-envelope.jpg)

## The wire-to-mount pipeline, at a glance

![Flowchart: Comet route payload envelope — server envelope layers feeding ClientConsistency, ServerJS execution, Bootloader resolution, and mount, with telemetry footer](assets/post09-diagram-route-envelope-pipeline.png)

## 1. The envelope

```json
{
  "payload": {
    "payloads": {
      "/profile.php?id=61593763958968&__tn__=%3C": { ... }
    },
    "sr_payload": { ... },
    "log_roots": ["CometRouteProfileTimelineListViewJSRoot"]
  },
  "dtsgToken": null,
  "dtsgAsyncGetToken": null
}
```

Three layers, three jobs:

- **`payloads`** — the actual responses, **keyed by request URI**. One envelope can batch several route responses, addressed by the path that asked for them. This is the `getAsyncParams` batching of Part 2 seen from the server side: the client sends N routes, the server returns a map of path → result.
- **`sr_payload`** — *server resources payload*: the Haste/CSS resource deltas needed to render the responses. This is Part 3's resource system in wire format.
- **`dtsgToken` / `dtsgAsyncGetToken`** — deliberately nulled. For GET route fetches the CSRF tokens ride in cookies/headers (`fb_dtsg_ag` channel, Part 2 §2.2), so the body fields are zeroed. Note the schema choice: the keys are **present with null values**, not absent — a stable shape the client parser can rely on across token/no-token responses.

The request key itself is evidence: `/profile.php?id=61593763958968&__tn__=%3C` — the trailing `%3C` is a truncated tracking nonce (`__tn__` is Facebook's click-attribution parameter; `%3C` is literally `<`, the remainder of a longer token cut by the capture). The envelope preserves the full request identity, tracking noise and all.

## 2. The route definition

Each payload value is a `route_definition`:

```json
{
  "error": false,
  "result": {
    "type": "route_definition",
    "exports": {
      "actorID": "61593763958968",
      "rootView": { ... },
      "tracePolicy": "comet.profile.timeline.list",
      "meta": { "title": null, "accessory": null, "favicon": null },
      "prefetchable": true,
      "canonicalRouteName": "comet.fbweb.CometProfileTimelineListViewRoute",
      "timeSpentConfig": { "has_profile_session_id": true },
      "entityKeyConfig": { ... },
      "hostableView": { ... },
      "stripParams": ["modal", "modal_param"],
      "productAttributionId": "190055527696468",
      "canonicalUrl": null
    }
  }
}
```

This is the **route contract** — everything the client router needs to mount the view without asking the server anything else:

| Export | Function |
|---|---|
| `canonicalRouteName` | Stable route ID (`comet.fbweb.CometProfileTimelineListViewRoute`). The client-side route table in the prelude maps this to a root component. |
| `rootView` | The React tree recipe (§3). |
| `tracePolicy` | Logging namespace — `comet.profile.timeline.list`. Feeds QPL flow naming (Part 6) and Banzai event attribution. |
| `prefetchable: true` | The router may fetch this route's payload before navigation (hover/click-intent prefetch). |
| `stripParams` | Query params the router **deletes from the URL after resolution** — `modal`, `modal_param` get consumed into router state, then erased so back/forward and copy-paste URLs stay clean. |
| `timeSpentConfig.has_profile_session_id` | This route contributes a session ID to the time-spent beacon pipeline (Banzai `timespent` channel). |
| `productAttributionId` | Ads/attribution bookkeeping carried through the route layer. |
| `entityKeyConfig` | The **normalized-store key recipe** (§4). |

## 3. The view recipe: references, never code

```json
"rootView": {
  "allResources": [
    { "__jsr": "ProfileCometTimelineListViewRoot.react" },
    { "__jsr": "ProfileCometTimelineListViewRouteRoot.entrypoint" },
    { "__jsr": "ProfileCometRoot.react" }
  ],
  "resource": { "__jsr": "ProfileCometTimelineListViewRoot.react" },
  "props": {
    "viewerID": "61593763958968",
    "userVanity": "",
    "userID": "61593763958968",
    "eligibleForProfilePlusEntityMenu": false,
    "trackingCode": null
  },
  "entryPoint": { "__dr": "ProfileCometTimelineListViewRouteRoot.entrypoint" }
}
```

Nothing here is code. `__jsr` = **JS resource reference**; `__dr` = **deferred reference** (the exact tokens the prelude's ServerJS parser dispatches on, Part 5). The envelope ships *names*; the client's Bootloader resolves them against the Haste manifest, fetches the chunks if missing (Part 3's `BootloaderEndpoint` + bitmaps), and only then mounts. `rootView.props` is the serialized prop tree for the root component — already filtered server-side to what the client component declared (`viewerID`, `userID`, vanity, two feature flags, and a null tracking slot — shape stability again: present-but-null beats absent).

## 4. The entity key recipe

```json
"entityKeyConfig": {
  "entity_type": { "source": "constant", "value": "profile" },
  "entity_id":   { "source": "prop",     "value": "userID" },
  "section":     { "source": "constant", "value": "time..." }
}
```

Relay-style normalized store addressing, shipped as **declarative data instead of code**. Each key component is either a `constant` (baked into the route) or a `prop` (read from `rootView.props`). The client composes the store key for this view's data — e.g. `profile:61593763958968:timeline` — without any route-specific logic. New route sections are deployed by changing config, not client code. This is the same philosophy as the dynamic parameter *names* in `getAsyncParams` (Part 2): **the server owns the schema; the client owns the mechanism.**

## 5. `sr_payload`: Haste BitMap RLE in the wild

```json
"sr_payload": {
  "hsrp": {
    "hblp": {
      "consistency": { "rev": 1048260127 },
      "rsrcMap": {
        "csr:_9w_0_bq": { "type": "csr", "src": ":1,2,31", "c": 1 }
      },
      "indexUpgrades": {}
    }
  },
  "jsmods": { "require": [ ... ] },
  "allResources": ["csr:_9w_0_bq"],
  "tieredResources": { "r": ["csr:_9w_0_bq"], "rdfds": [], "rds": [] }
}
```

This block is Part 3's theory confirmed by production data:

- **`hsrp`** — Haste Server Resources Payload; **`hblp`** — Haste BitMap *something* Payload (the bundle's naming). Inside:
- **`consistency.rev: 1048260127`** — the Haste manifest revision these resources were built against. The client's `ClientConsistency` logic (Part 3: actions 2 = soft refresh, 3 = hard refresh) compares this to its own manifest revision and decides whether to re-fetch the manifest before executing.
- **`rsrcMap["csr:_9w_0_bq"]`** — one CSS resource, and the payload is a **bitmap**: `"src": ":1,2,31"`, `"c": 1`. Decoded with the RLE scheme from Part 3 (the `0-9a-zA-Z-_` alphabet, `:` as run separator): **chunks 1–2 and chunk 31** of the shared CSS chunk index. The leading `:` is an empty initial run — chunk 0 absent. `c: 1` flags the compressed form.
- **`tieredResources`** — scheduling tiers, straight from the prelude's resource scheduler (Part 3): `r` = regular tier, `rdfds` = require-deferred, `rds` = deferred. This route carries exactly one regular-tier CSS resource and nothing deferred.
- **`jsmods.require`** — the legacy ServerJS require table, with a twist:

```json
["emptyFunction@5f4417fd21bf881f4dcb6c20b897af37", "thatReturns", ["JSResourceReferenceImpl"],
  [[{"__jsr": "ProfileCometTimelineListViewRoot.react"}, ...]]],
["ProfileCometTimelineListViewRouteRoot.entrypoint@f0255a8b52d05a621c1c1b1a605c0aba"],
["emptyFunction@5acd8bc3952d7b9c9ec92adbb6652ca0", "thatReturns", ["RequireDeferredReference"],
  [[{"__dr": "ProfileCometTimelineListViewRouteRoot.entrypoint"}, ...]]]
```

Deferred resources aren't expressed as real modules — they're **factories**: `emptyFunction.thatReturns(JSResourceReferenceImpl)` is a placeholder definition whose entire body is "return this reference." When the require table executes (Part 5's ServerJS pipeline), these resolve through the prelude's `RequireDeferredReference` machinery: registration is instant, execution waits until the router actually mounts the route. The `@` suffixes (`@5f4417fd…`, `@f0255a8b…`) are content hashes — the Haste consistency check can verify a loaded module is the version the server intended.

## 6. Putting it together: the full pipeline

```
  user clicks /profile.php?id=...
        │
        ▼
  Comet router: route table lookup (canonicalRouteName)
        │
        ▼
  GET route fetch ── fb_dtsg_ag token attached (Part 2, cr:8960)
        │
        ▼
  ┌──────────────── SERVER ────────────────┐
  │ route_definition built:                 │
  │   rootView = recipe of __jsr/__dr refs  │
  │   entityKeyConfig = store key recipe    │
  │ sr_payload:                             │
  │   hsrp.hblp.consistency.rev = 1048260127│
  │   rsrcMap = Haste bitmap ":1,2,31"      │
  │   jsmods = factory-wrapped requires     │
  │ dtsgToken nulled (tokens ride in hdrs)  │
  └─────────────────────────────────────────┘
        │
        ▼
  Client: ClientConsistency check rev (Part 3)
        │ mismatch? → soft/hard refresh first
        ▼
  ServerJS executes jsmods (Part 5) ── factories register,
        real resources queued in Bootloader tiers
        ▼
  Bootloader resolves __jsr refs; fetches missing chunks
        using the bitmap deltas (Part 3)
        ▼
  CSS chunks {1,2,31} injected; React mounts rootView
        with server-filtered props
        ▼
  stripParams erases ?modal=... from the URL;
        tracePolicy + log_roots feed QPL/Banzai (Part 6)
```

Three parts of the prelude series (request signing, Haste/bootloader, ServerJS) are not three systems — they are three stages of **one** pipeline, and this envelope is the artifact that binds them. The server sends recipes and bitmaps; the client sends nothing but a signed GET; every name in the payload resolves against machinery documented in Parts 2–5.

## 7. Research notes

**Why `dtsgToken: null` matters to tooling:** parsing these envelopes for tokens (the classic "scrape fb_dtsg from response" trick) fails by design on the GET route path. Tokens live in the `Set-Cookie`/`fb_dtsg_ag` header channel. Tools that only inspect JSON bodies will report "no token" on a perfectly healthy session.

**Bitmap practice:** `:1,2,31` decodes under the Part 3 RLE scheme to chunk set `{1,2,31}`. Cross-checking against the client's decompressed manifest for build `rev 1048260127` confirms the mapping; the `c:1` flag tells the client to run the decoder rather than treat `src` as a literal list.

**The `__tn__=%3C` truncation:** capture-side artifact, but instructive — the envelope keys payloads by exact request URI including tracking params. If you batch requests with distinct `__tn__` values, you get distinct envelope keys for semantically identical routes.

**Schema stability as a fingerprint:** the recurring pattern across every layer — `dtsgToken: null` present, `trackingCode: null` present, `meta.title: null` present — is deliberate shape-stability. Null slots are cheaper than schema versioning, and they make client-side destructuring unconditional. When this envelope eventually changes, the nulls disappearing (or new nulls appearing) is the diff to watch.

## 8. Summary

A Facebook page navigation is answered with a self-describing JSON contract: the route's component names (never code), its store-key recipe, its CSS needs expressed as a run-length bitmap against a versioned manifest, its modules wrapped as lazy factories, and its CSRF tokens pointedly absent from the body. The prelude bundle documented in Parts 1–7 is the machine that digests exactly this format. Server and client are two halves of one design, and seeing the wire format makes the client machinery — every `__jsr` check, every bitmap decode, every deferred reference — read as inevitable rather than arbitrary.

---

*Series index · Part 1 (Ghost Owl) · Part 2 (request signing) · Part 3 (Bootloader/Haste) · Part 4 (URL schemes) · Part 5 (ServerJS) · Part 6 (telemetry/Hyperion) · Part 7 (module index) · Part 8 (fbcdn signed URLs)*

*frankSx — hardware/web reverse engineering, parser internals, anti-analysis systems. frankhacks.blogspot.com*
