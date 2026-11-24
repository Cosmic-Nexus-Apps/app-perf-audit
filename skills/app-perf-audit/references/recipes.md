# Verified recipes

Each pattern below was applied and measured on a real device. Snippets are React Native; see the platform map in `symptoms.md` for equivalents.

## 1. `wasVisible` guard (dead progressive-render fix)

Wrong — the `return` exits before the flag is updated, so the close branch never runs:

```tsx
useEffect(() => {
  if (visible && !prevVisibleRef.current) {
    // ...schedule the heavy content
    return () => { /* cleanup */ };
  } else if (!visible && prevVisibleRef.current) {
    setContentReady(false);           // dead code
  }
  prevVisibleRef.current = visible;   // never reached on open
}, [visible]);
```

Right:

```tsx
useEffect(() => {
  const wasVisible = prevVisibleRef.current;
  prevVisibleRef.current = visible;   // always updated, before any return
  if (visible && !wasVisible) {
    // ...schedule
    return () => { /* cleanup */ };
  }
  if (!visible && wasVisible) {
    setContentReady(false);
    stopAllLoops();
  }
}, [visible]);
```

Effect: heavy subtrees stop mounting on the first frame of every reopen.

## 2. Staged mounting ("waves")

Mount the shell first, then the sections in separate commits, so no single commit creates the whole tree.

```tsx
useEffect(() => {
  if (!visible) return;
  const timers = [
    setTimeout(() => { if (mounted) setPhasesReady(true); }, 150),
    setTimeout(() => { if (mounted) setEclipsesReady(true); }, 250),
    setTimeout(() => { if (mounted) setConjunctionsReady(true); }, 500),
    setTimeout(() => { if (mounted) setCalendarReady(true); }, 750),
  ];
  return () => timers.forEach(clearTimeout);
}, [visible]);
```

Rules:
- header and hero render immediately, ungated;
- each gated section is below the fold, or its absence does not shift what is visible;
- do **not** build the deferred state from a `useMemo` that runs on the open render — that defeats the purpose.

## 3. Animation budget with settle

Stop looping animations when the surface is not visible, and after a short settle window.

```tsx
const SETTLE_MS = 8000;
const AnimationsActiveContext = createContext(true);

export const AnimationsActiveProvider = ({ active, children }) => {
  const [settled, setSettled] = useState(false);
  useEffect(() => {
    if (!active) { setSettled(false); return; }
    const t = setTimeout(() => setSettled(true), SETTLE_MS);
    return () => clearTimeout(t);
  }, [active]);
  return <Provider value={active && !settled}>{children}</Provider>;
};
```

`active` must be false when another tab covers the screen, a modal is open, or the screen is unmounted. Consumers:

```tsx
const animationsActive = useAnimationsActive();
useEffect(() => {
  cancelAnimation(scale);
  scale.value = 1;
  if (!animationsActive) return;          // resting pose
  scale.value = withRepeat(/* ... */, -1, true);
  return () => cancelAnimation(scale);
}, [animationsActive]);
```

Measured effect: idle frames 50 fps → ~18 fps; a hidden screen went from 154 to 2 frames per 3 s.

## 4. Stable callbacks so `React.memo` works

Wrong:

```tsx
<GridTile photo={p} onPress={() => handleOpen(p)} />   // new closure every render
```

Right:

```tsx
// tile prop: onPress: (photo: Photo) => void  → calls onPress(photo) internally
<GridTile photo={p} onPress={handleOpen} />
```

Also hoist inline style arrays/objects passed to memoized children, and keep handlers in `useCallback` with stable deps.

## 5. Stable image `source` objects

A new `{ uri }` literal per frame makes the image module reload the bitmap.

```ts
const cache = new Map<string, { uri: string }>();
export function sourceFor(uri: string) {
  let s = cache.get(uri);
  if (!s) { s = { uri }; cache.set(uri, s); }
  return s;
}
```

## 6. Debounce plus deferred index

Search that rebuilds an index of thousands of items per keystroke:

- debounce the input (160–250 ms) before it feeds the filter;
- build the index **once, deferred after the screen settles** (~800 ms) — not on the first keystroke (a cold build can take seconds) and not in the first 10 ms (it blocks the opening frames);
- never rebuild synchronously inside the results `useMemo` as a fallback — return `[]` until ready.

Counter-example from a real session: moving the build to the first keystroke made search wait ~6 s. Reverted the same session.

## 7. Coalesce subscription bursts

```tsx
useEffect(() => {
  let timer: ReturnType<typeof setTimeout> | null = null;
  const unsub = subscribe(() => {
    if (timer) return;
    timer = setTimeout(() => { timer = null; bump(); }, 400);
  });
  return () => { unsub(); if (timer) clearTimeout(timer); };
}, []);
```

Also filter notifications: compare the keys the screen actually uses instead of re-rendering on every storage write.

## 8. Partial virtualization

When a screen has several layouts, virtualize only the uniform one (grid) and keep the complex one (masonry) on the existing scroller. Anything less than a root `FlatList` is not virtualization — a list nested inside a `ScrollView` does not virtualize.

Checklist when converting:
- the header and search live in `ListHeaderComponent` **as a memoized element** (`ListHeaderComponent={headerElement}`, not an inline arrow), otherwise the input remounts on every keystroke and loses focus;
- month/section headers become list items so they scroll virtually too;
- keep tile props identical so visuals do not change.

## 9. Cheap wins that hold up

- `resizeMethod="resize"` on Android thumbnails.
- Replace `LayoutAnimation.configureNext` with nothing, or with a native-driver transform; never keep a global layout animation in an app where all tabs stay mounted.
- Prefetch a neighbourhood of frames, not the whole set.
- `removeClippedSubviews` on a long `ScrollView` as a stopgap — not a replacement for virtualization.

## 8. Progressive preview (desktop, measured)

A 4K render on every slider move is a freeze; a stale preview is a lie. Split by intent:

- debounce ~40 ms while the value keeps changing -> render at ~1/3 width;
- settle timer ~300 ms after the last change -> render full resolution.

Measured on one 4K frame: 0.32 s preview vs 4.5 s full; the settle render lands 0.3 s after
the hand leaves the slider, so the user never sees a half-resolution frame while judging.

```cpp
// setter path
m_adjusting = true;  m_settle.start();  m_debounce.start(40);
// debounce -> render() with preview = m_adjusting
// settle   -> m_adjusting = false; render()
```

Keep the preview flag out of exports: an export must always be full resolution.

## 9. Soft threshold for a fill: match the grain (desktop, measured)

An inpainting/retouch pass that writes a smooth patch into a grainy frame reads as "erased"
at 100% even when the average level is right. Measure the grain (median |high-pass| / 0.6745)
on the untouched pixels and add the same sigma to the patch, per channel. Users describe the
smooth version as a bug ("it looks wiped"), not as a missing texture.

## 10. Software renderer fallback (desktop)

`--software` that sets `QT_QUICK_BACKEND=software` before the QGuiApplication exists turns a
"does not start on this machine" ticket into a usable app. Pair it with the log line that
names the D3D11 error, so the next report is one screenshot away from a diagnosis.
