# URI Scheme Allowlist — Quick Reference

150 schemes in `URISchemes`, verbatim set in [Part 7 Appendix B](https://github.com/FrankSx/facebook-comet-prelude-files/blob/main/07-appendix-module-index.md). Categorized analysis: [Part 4](https://github.com/FrankSx/facebook-comet-prelude-files/blob/main/04-uri-scheme-allowlist.md).

| Category | Count | Highlights |
|---|---|---|
| Core FB/Messenger | 28 | `fbrpc`, `fb-messenger-group-thread`, `fbstaging` |
| App-ID | 13 | `fb<appid>` incl. `rc`/`master` dogfood variants |
| IG/Threads/WhatsApp | 6 | `barcelona` = Threads |
| Oculus/Horizon/spatial | 25 | `fb-nimble-vrsrecorder`, `fb-nimble-monohandtrackingvis` |
| AI | 5 | `meta-ai`, `meta-bloks`, `vibes` |
| Dev/internal | 29 | `fb-vscode*`, `flipper`, `munki`, `manifold`, `fb-owl` |
| UPI payments | 6 | `upi`, `gpay`, `tez`, `phonepe`, `paytmmp`, `bhim` |
| Platform/generic | 38 | `intent`, `x-safari-https`, `apk`, fossils: `lasso`, `flash`, `gtalk` |

Enforcement: scheme allowlist + userinfo ban + Unicode-lookalike host denylist + backslash-path rejection in `URIAbstractBase.parse`; `__proto__`/`hasOwnProperty` query keys rewritten to `\uD83D\uDF56` in `BaseDeserializePHPQueryData`.
