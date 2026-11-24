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

## Desktop (Qt / native engine)

The protocol is the same: same machine, same build type, same gesture, before and after.
The tools change.

### Cold start

There is no `am start -W`. Start the process and poll for the first *usable* signal - a
readiness socket, a log line after the first frame, or a window-ready callback:

```powershell
$sw = [Diagnostics.Stopwatch]::StartNew()
$p  = Start-Process -FilePath '.\app.exe' -PassThru
while ($sw.Elapsed.TotalSeconds -lt 20) {
  if (Ready($p)) { break }        # socket handshake or a marker in the log
  Start-Sleep -Milliseconds 50
}
$sw.Elapsed.TotalSeconds, (Get-Process -Id $p.Id).WorkingSet / 1MB
```

Measured example (Qt/QML app, Windows): 1.7 s from cold file cache, 0.58-0.60 s warm,
~200 MB idle. Report both - the first number is what a user sees after a reboot.

### Stage profiling

Add a flag (`CS_PROFILE=1`) that prints each pipeline stage in ms, then read it twice in the
same process. **The second render is 5-10x faster than the first** because the first one
pages in every f32 plane (33 MB each at 4K). Comparing a cold first render with a warm
second one invents a regression that does not exist; measure each in isolation.

Verified example, 4K frame, per stage (cold / warm):

```
cosmetic        1708 ms / 202 ms   -> parallelised over rows: 300 / 33 ms
wavelet NR       767 ms /  64 ms   -> parallelising it made cold 14% better and warm 18% worse
stretch + sat    155 ms /  17 ms
stars            197 ms /  16 ms
```

Conclusion that saved a day: the cost was page faults, not arithmetic.

### Memory

```powershell
(Get-Process -Id $p.Id).WorkingSet / 1MB        # what the OS sees
(Get-Process -Id $p.Id).PrivateMemorySize64 / 1MB
```

Sample after each step (idle -> opened the heavy screen -> applied the heavy operation).
A jump that scales with the data size is the OOM risk on user machines.

### Driving the real UI (the adb-input equivalent)

A tiny localhost JSON socket in the app (only when a debug/automation flag is on) beats
guessing: `state`, `screenshot`, `click x= y=`, `type`, `key`, `drag`, `zoom`, `open`.
It is how "cannot test it myself" stops being an excuse. Keep it bound to 127.0.0.1 and
allow it to be switched off; never expose it on the network.
