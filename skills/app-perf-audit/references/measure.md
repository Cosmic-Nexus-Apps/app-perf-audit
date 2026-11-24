# Measuring (Android / adb)

Rules: same device, same build type, same gesture geometry, both runs. Numbers across different builds or devices prove nothing.

## Cold start

```sh
adb -s <serial> shell am force-stop <pkg>
adb -s <serial> logcat -c
adb -s <serial> shell am start -W -n <pkg>/.MainActivity     # TotalTime = first frame
```

`TotalTime` measures until the first frame of the activity, not until your data is rendered. To measure that, instrument the app (below).

## Frame jank

```sh
adb -s <serial> shell dumpsys gfxinfo <pkg> reset
# ...the action: one tap, or N identical swipes...
adb -s <serial> shell dumpsys gfxinfo <pkg>
```

Read the summary line (`Janky frames`, 50th/90th/95th/99th). Notes:

- Some devices emit no `framestats` section; the histogram counters still work.
- A 99th percentile of 150–450 ms is the number a human feels as a freeze.
- Scroll noise is large: run the same action twice and compare both.
- Measure idle too: 3 s untouched, count frames. A screen rendering 150 frames while producing identical pixels is wasting battery.

## Memory

```sh
adb -s <serial> shell dumpsys meminfo <pkg> | grep -E 'TOTAL PSS|TOTAL RSS|Native Heap|Java Heap'
```

Protocol: measure on the screen → open the heavy screen → scroll → measure again. Growth that follows data volume (not time) is the release risk. PSS around 180–270 MB is normal for React Native; a +80 MB jump from one navigation is not "normal" — it is what OOMs a 2 GB device.

## Weak-device profile

```sh
"$ANDROID_SDK/emulator/emulator" -avd <name> -no-window -no-audio -no-boot-anim \
    -gpu swiftshader_indirect -memory 2048 -cores 2
```

Verify with `adb shell nproc` and `adb shell "grep MemTotal /proc/meminfo"`.

Caveat: the emulator GPU is software, so frame numbers are pessimistic. Use the emulator for **memory, crashes and gross regressions**, and a physical device for jank.

## Instrumenting an open

```tsx
// tap handler
(globalThis as any).__tap = Date.now();
// component body, top
console.log('[PERF] render', Date.now() - __tap);
// after the expensive computation
console.log('[PERF] compute', ms);
// first useEffect after commit
console.log('[PERF] first commit', Date.now() - __tap);
// deferred callback
console.log('[PERF] content committed', Date.now() - __tap);
```

Read with `adb logcat -d | grep '\[PERF\]'`. Remove the logs in the final build — they are synchronous in debug.

## Comparing

Put the numbers in a table: stage | before | after. Also report what could not be measured, and why. A fix with no measurable delta is not a win; say so.

## Release vs debug

Debug bundles are larger, unminified and slower. Test the target build.

Release needs a signing config. Signing with a throwaway debug keystore is acceptable **for measurement on an emulator only**:

```sh
-Pandroid.injected.signing.store.file=<debug.keystore> \
-Pandroid.injected.signing.store.password=android \
-Pandroid.injected.signing.key.alias=androiddebugkey \
-Pandroid.injected.signing.key.password=android
```

Never ship a build signed with a test keystore.
