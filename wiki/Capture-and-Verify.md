# Capture and Verify

How to reproduce or extend this analysis against a live capture.

## Capture

1. Load facebook.com (any surface), open view-source or DevTools → Elements.
2. Find the inline `<script>` beginning `;/*FB_PKG_DELIM*/` in `<head>`.
3. Save the full script body to a file, UTF-8, no trimming.

## Verify against this series

```
sha256sum capture.txt
# series reference: 57d60f40f636669c259ec6f945ae4264ac4b55cba01a5e767affaad900e69ec4
```

Hashes will differ across deploys/cohorts — that's expected. Compare structurally instead:

```
grep -o '__d("[^"]*"' capture.txt | sort -u | wc -l     # module count (reference: 232)
grep -c 'ad_blocker_defense_ghost_owl' capture.txt      # Ghost Owl present
grep -o '"[a-z0-9]\{8\}"in t' capture.txt | sort -u   # GHL Env gate keys (rotated per deploy)
```

## Diff against the module index

Extract `__d` names from your capture and diff against [Part 7](https://github.com/FrankSx/facebook-comet-prelude-files/blob/main/07-appendix-module-index.md). New modules = new prelude capability; missing gates = cohort difference.

## Notes

* Gate arming (`gkx`/`justknobx`/Env keys) varies per cohort and deploy; code paths are constant.
* Env gate keys are 8-char randoms rotated per deploy — match by position/usage, not value.
