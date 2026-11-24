---
name: app-perf-audit
description: Use when an app feels laggy, takes long to open, shows a dark or blank screen instead of content, drains battery, grows memory or crashes on other users' devices, or before a release. Device-measured performance audit with symptom-driven detection, verified fix recipes and an evidence protocol. Covers React Native/Android primarily, with conceptual mappings to Compose, Flutter, SwiftUI and Qt/QML.
---

# App performance audit (device-measured)

Do not optimize from intuition. Every finding needs a `file:line` or a measurement; every fix needs the same measurement repeated.

## When to use

- "the app lags / scroll stutters / opening is slow"
- "the screen just darkens and nothing appears"
- "it's fine on my phone, but users complain"
- before a release, or after adding a heavy screen
- battery or heat complaints

## Hard rules

1. **No claim without a line reference** (`file:line` + quoted code) or a number.
2. **No fix declared done without a measurement on a device**, before and after, same build type.
3. **One batch at a time.** Never stack unverified changes; a broken batch hides which edit broke it.
4. **Reproduce first.** If you cannot reproduce, say so and treat the fix as unverified.
5. **Do not undo a deliberate trade-off** made after user feedback (for example a spinner that replaced a jumping list). Read the comments around the code.
6. **Measure the real target.** Debug builds are 2–3× slower than release; debug jank numbers are relative, not absolute.

## Workflow

1. **Map the app**: every screen, every full-screen modal, every list, plus the shell (tabs, boot).
2. **Split the audit by area** and run read-only analysts in parallel — one per screen group, plus one for the shell and overlays. Use the same checklist for all so findings are comparable.
3. **Verify the reports yourself** in the code. Reject speculation. Keep only findings with a line reference.
4. **Instrument the worst symptom** before fixing it: temporary timestamps at tap, render start, first commit and deferred commit. One build.
5. **Fix the cause** with the smallest complete change, then rebuild and repeat the identical measurement.
6. **Report what is proven vs code-only.** Never present a code fix as a measured win.

## Symptom → start here

| Symptom | First suspects | Details |
|---|---|---|
| Slow open, blank then content | sync work in the first render, everything mounted in one commit, dead progressive guard | `references/symptoms.md` — open-lag |
| Stuttering scroll | memo killed by inline props, a gesture handler stealing the scroll, oversized images | `references/symptoms.md` — scroll-jank |
| Screen dims, nothing appears | native modal inside a native modal, backdrop without content | `references/symptoms.md` — dark-screen |
| Battery/heat, app never idle | infinite repeat animations, hidden screens still mounted | `references/symptoms.md` — idle-fps |
| Grows over time, crashes for some users | unvirtualized lists, base64 in memory, unbounded caches | `references/symptoms.md` — memory |
| Slow cold start | serial initializers, dead code parsing large payloads at boot | `references/symptoms.md` — boot |

## Recipes (verified)

`references/recipes.md` — copy-paste patterns: `wasVisible` guard, staged mounting waves, animation budget with settle, stable callbacks for memo, stable image sources, debounce plus deferred index, coalescing subscriptions, partial virtualization.

## Measuring

`references/measure.md` — adb commands (`am start -W`, `dumpsys gfxinfo`, `dumpsys meminfo`), a weak-device emulator profile, the before/after protocol, and what each number can and cannot prove.

## Process pitfalls

`references/process.md` — hook order, build OOM, device focus guard, logging without blowing memory, and the mistakes that cost hours in real sessions.

## Keeping it current

If an audit finds a cause that is not in `symptoms.md`, add it **in the same session** (symptom → cause → grep hint). If a recipe proves itself on a device, add the measured effect to `recipes.md`. If a measurement misleads you once, write the caveat into `measure.md`. A recipe with no measured effect is a hypothesis — label it.
