# Facebook's Custom URL Scheme Allowlist: A 150-Entry Attack Surface Inventory

*Part 4 of The Facebook Comet Prelude Files · frankSx · frankhacks.blogspot.com*

---

Every URL Facebook's web code constructs, parses, or navigates to passes through the `URI`/`URIBase` layer — and that layer refuses to emit a URL whose scheme isn't on an explicit allowlist. The list lives in the `URISchemes` module as a single `Set` literal, and it is effectively a **map of every bridge between Facebook's web properties and its native apps, partner apps, payment rails, and internal tools**.

The production list, extracted verbatim from the prelude bundle, contains **150 schemes**. Meta publishes this nowhere. Below is the complete inventory, categorized, with enforcement mechanics and research notes. This is the first complete public enumeration.

## 1. The complete list, categorized

### Core Facebook / Messenger (28)

```
fb  fba  fbatwork  fb-ama  fb-internal  fb-workchat  fb-workchat-secure
fb-messenger  fb-messenger-public  fb-messenger-group-thread  fb-page-messages
fb-pma  fbagenthome  fbcf  fbconnect  fbinternal  fbmobilehome  mobilehome
fbrpc  fbstaging  fblite  fb-mk  fb-viewapp  fbboost  messenger  workchat
accountscenter  socialplatform
```

**Notes.** `fbrpc` is Facebook's RPC scheme, historically used for web→native calls with structured payloads — the highest-value target in the list. `fb-messenger-group-thread` and `fb-page-messages` deep-link into specific conversation contexts: a web page that can navigate to these drops a user into a specific native UI state, which is phishing-adjacent. `fbstaging` is a staging-environment handler — its presence in production allowlists is a recurring Meta quirk.

### App-ID schemes (13)

```
fb124024574287414  fb124024574287414rc  fb124024574287414master
fb1576585912599779  fb929757330408142  fbapi20130214  fb1196383223757595
fb1680871178595114  fb1543576032349914  fb1635404796768116  fb147781309031234
fb236786383180508  fb1775440806014337
```

The `fb<appid>` pattern is how apps registered with Facebook's platform get URL handlers. The `rc` and `master` suffixes on the first entry are release-candidate and master-branch dogfood builds of the same app — internal variants leaking through the production allowlist. `fbapi20130214` is the legacy API scheme dated to the 2013 platform era.

### Instagram / Threads / WhatsApp (6)

```
instagram  iglite  barcelona  whatsapp  whatsapp-consumer  whatsapp-smb
```

`barcelona` is Threads (internal codename). The `whatsapp` / `whatsapp-consumer` / `whatsapp-smb` split mirrors the consumer/business app split.

### Oculus / Quest / Horizon / spatial computing (25)

```
oculus  oculus.store  oculus.feed  oculusstore  com.oculus.rd  odh
oculus360photos  horizon  horizonlauncher  facebook-horizon  venues
meta-spatial-editor  spark-studio  spark-player  spark-simulator
cosmo-player  arstudio  hsr-asset-viewer  hsr-editor  moonstone
fb-nimble-vrsrecorder  fb-nimble-monohandtrackingvis  stella  together  togetherbl
```

**Notes.** The `fb-nimble-*` pair is striking: VR session recording and **monocular hand-tracking visualization** handlers, reachable from the web layer. `spark-*`/`arstudio` are the AR effects toolchain. `hsr-*` is the asset review/editor pipeline. `moonstone`, `stella`, `together`/`togetherbl` are codenames with no public product mapping. `com.oculus.rd` is a reverse-domain scheme — an Android-style package handler living in a web allowlist.

### AI products (5)

```
meta-ai  aidemos  aistudio  meta-bloks  vibes
```

`meta-bloks` is the Bloks server-driven native UI framework — Bloks screens are addressable by URL. `aidemos`/`aistudio` are the AI demo surfaces; `vibes` is the AI video feed.

### Developer / internal tooling (29)

```
fb-ide-opener  fb-vscode  fb-vscode-insiders  fb-vscode-dev  editor
flipper  origami-file  origami-internal  origami-public  manifold
munki  mattermost  logaggregator  pcoip  gizmo  aura  hatch  basel
lantern  fb-owl  glam  designpack  fbpixelcloud  cmms  q4bconfigurator
q4bnux  ctrl-hub  ctrl-launcher  assethub
```

The most revealing category for an external reader: `fb-vscode*` opens a file in an engineer's editor from the web (stack-trace → IDE bridges); `flipper` is the mobile debugging platform; `munki` is macOS fleet management; `mattermost` internal chat; `logaggregator` speaks for itself; `pcoip` is remote desktop (PC-over-IP). `manifold`, `gizmo`, `aura`, `hatch`, `basel`, `lantern`, `fb-owl`, `glam` are internal codenames — an OSINT gift; several have appeared in subsequent Meta job postings and open-source references, confirming they name real infrastructure.

### Payments — India UPI rails (6)

```
upi  phonepe  gpay  tez  paytmmp  bhim
```

The full Indian UPI ecosystem: Google Pay (`gpay` + legacy `tez`), PhonePe, Paytm, BHIM. Present for WhatsApp Pay and checkout flows. Notably absent: any Western payment scheme — no `venmo`, no `paypal`.

### Platform / OS / generic (38)

```
http  https  wss  ftp  sftp  svn+ssh  file  blob  data  about  content
mailto  tel  sms  callto  skype  webcal  itms  itms-apps  itms-services
market  ms-app  ms-windows-store  chrome-extension  x-safari-https
intent  apk  lasso  gtalk  pebblejs  moments  flash  home  systemux
cinema  aria  mwa  tbauth
```

**Notes.** `intent` (Android intent URLs) and `x-safari-https` (iOS Safari scheme-escape) are the platform-bridge schemes — both have rich exploitation histories ecosystem-wide. `apk` triggers package installs. `lasso` is Meta's defunct short-video app — a fossil. `flash` is a deeper fossil. `gtalk` and `pebblejs` are paleontology. `chrome-extension` being allowed means Facebook pages can legitimately reference extension URLs. `data` and `blob` are allowed but (see §3) constrained elsewhere in the parser.

## 2. Enforcement mechanics

`URISchemes` wraps the set in a `$InternalEnum` with membership options (`INCLUDE_DEFAULTS` etc.), and `URIAbstractBase.parse` rejects any URL whose scheme fails the check — unless parsing in lenient mode:

```js
// de-minified from URIAbstractBase.parse
if (!lenient && !URISchemes.isAllowed(parsed.scheme, this.options, this.subOptions))
  return false;
// ...
if (parsed.userinfo !== null) {
  if (lenient) throw err("URI.parse: invalid URI (userinfo is not allowed in a URI): " + raw);
  return false;
}
```

The `$InternalEnum` machinery itself is worth noting: it builds a **frozen, reverse-indexed enum object** (value→name `Map` cached in a WeakMap) with `isValid`, `cast`, `members`, `getName` — and a `Mirrored` variant for string=value enums. It's the same pattern used for `RequireDeferredFactoryEvent` and other protocol enums across the prelude.

## 3. Adjacent parser hardening

The same module layer contains hardening that has nothing to do with schemes but everything to do with URL-confusion attacks:

1. **Userinfo categorically rejected** — no `https://user:pass@host` URLs, killing a class of phishing/credential-leak constructions at the parser.
2. **Host character denylist** — the domain regex rejects `\x00-\x2c`, `\x2f`, `\x3b-\x40`, `\x5c`, `\x5e`, `\x60`, `\x7b-\x7f`, Unicode noncharacters `\uFDD0-\uFDEF` and `\uFFF0-\uFFFF`, bidi-adjacent `\u2047`/`⁈`, and the fullwidth lookalikes `\uFE56`/`﹖`/`\uFF03`/`＃`/`\uFF0F`/`／`/`\uFF1F`/`？`. That last group is an anti-spoofing list: fullwidth `？` and `／` in a hostname would render like `?` and `/` and misparses downstream.
3. **Backslash-path rejection** — a path containing `\` with no domain throws in lenient mode (the `http:\\evil` confusion).
4. **Control-character scheme splitting** — schemes are matched against `^(?:[^/]*:|[\x00-\x1f]*/[\x00-\x1f]*/)` to catch `java\tscript:`-style splits.

The `BaseDeserializePHPQueryData` module adds one more gem: query keys named `hasOwnProperty` or `__proto__` are rewritten to the private-use codepoint `\uD83D\uDF56` before assignment — **prototype-pollution defense in the query string parser**.

## 4. Why this list matters for research

- **Each scheme is a registered handler somewhere.** On a device with the apps installed, `scheme://` URLs navigate out of the browser into native code — whose URL parsing is a different, usually softer, trust boundary. The allowlist is the menu of reachable native parsers.
- **The list is a recon index.** Codenames here have preceded product launches; internal-tool schemes confirm infrastructure naming. Diffing this list across deploys (it's in every prelude) is a free early-warning system for Meta product movement.
- **Allowlist drift is a bug class.** A scheme removed from a native app's handlers but left in this list (or vice versa) creates a navigation primitive with no receiver — or a receiver nobody is auditing. `lasso` and `flash` prove entries outlive their products.
- **For defenders elsewhere:** the pattern to steal is the *combination* — scheme allowlist + userinfo ban + Unicode-lookalike host denylist + prototype-pollution-safe query parsing, all enforced at one choke point.

## 5. Verification

The `Set` literal is verbatim from the bundle (hash in the series index). Reproduce by searching any captured prelude for `"fbrpc"` or `"fb-nimble-vrsrecorder"` — the set is a single contiguous literal and extracts in one regex.

---

*Next: how Facebook actually renders — the ServerJS streaming pipeline and its data-sjs payloads.*
