# Process pitfalls (learned the expensive way)

## Hooks

- A hook placed **after** a conditional `return` crashes with "Rendered more hooks than during the previous render". It appears only when the early-return branch is taken — often at boot, so the whole app dies.
- When adding a hook to an existing component, check every `return` above the insertion point.

## Verification discipline

- Never stack unverified changes. A batch of five fixes that breaks the app teaches nothing about which one broke it.
- Typecheck (`tsc --noEmit`) before every build; it catches hook and typing errors in seconds instead of a three-minute build.
- Test the flow you changed, not only that the app boots. A "dark screen" fix must be verified by opening the exact sheet that used to dim.
- When a measurement comes out *worse*, suspect the comparison before suspecting the code: cold cache vs warm cache, debug vs release, different device state.

## Device

- Before every capture, confirm the app is in front: `adb shell dumpsys activity activities | grep topResumedActivity`. Another app, or a picture-in-picture overlay, makes every number meaningless.
- If the device disappears from `adb devices`, do not assume a code problem — the phone may be charging on another cable or port.
- The user's phone may be in active use. Stop and ask rather than tapping around.

## Builds

- Large bundles (15 MB+) can OOM the Metro workers: `Zone Allocation failed`, exit `-1073740791`, `spawn UNKNOWN`, or a Gradle daemon killed by the system.
  Mitigate with `NODE_OPTIONS=--max-old-space-size=6144` and limit packaging workers (for example `extraPackagerArgs = ["--max-workers=2"]` in the `react { }` block). Too small a heap (3072) also fails; 4096–6144 works.
- Redirect build output straight to a file. A `Tee-Object` plus full-buffer pipeline can throw `OutOfMemoryException` in PowerShell on its own.
- Long C++ / Hermes release builds are normal; do not kill them at two minutes.
- Keystores live outside the repository. Never copy, read or log them; ask the user to sync secrets. A release build can be *measured* with a test keystore on an emulator, but never shipped.

## Reporting

- Separate "verified on device" from "code-only" in every report. The user makes release decisions on that distinction.
- State what was deliberately skipped and why (risk, low value at current data sizes, touching auth).
- If a fix introduces a regression, say it plainly and fix it in the same session.

## Maintaining this skill

- When an audit finds a cause that is **not** in `symptoms.md`, add it there in the same session, as a symptom → cause entry with the grep hint.
- When a fix recipe proves itself on a device, add it to `recipes.md` with the measured effect.
- When a measurement is misleading (cache state, build type, emulator GPU), add the caveat to `measure.md`.
- Keep entries short and evidence-based. A recipe without a measured effect is a hypothesis — label it as one.

## Desktop toolchains (Qt + Rust + MSVC)

- **`LNK1104: cannot open file '<app>.exe'`** means the app is still running. Do not kill the
  user's instance: build into a second build directory (`-BuildDir build-b`) and test there.
- **`fatal error C1060: compiler is out of heap space`**, `LNK1102: out of memory`, or rustc
  exiting with `0xC0000409` are **machine memory**, not code. Limit the parallelism
  (`CARGO_BUILD_JOBS=1`), close the audit's own leftovers, and retry before touching a line.
- A crashed compiler can leave a **zombie process at 0 MB that cannot be killed** (access
  denied) and poisons the target directory. Build into a separate target dir, or ask for a
  reboot; do not chase the error in the source.
- Installer builders (Inno `lzma2/max` + solid) can OOM on a busy machine: `lzma2/normal`,
  solid off. 60-90 s is normal; do not kill at one minute.
- The QML is compiled into the executable: a QML edit needs a rebuild (~10-30 s incremental).
  There is no hot reload unless the project sets one up - say so instead of "it should show".
