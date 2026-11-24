# Symptom-driven detection

Each entry: what the user reports, what to look for, and how to tell a real cause from noise.
Every finding must end up as `file:line` plus a quoted line.

## open-lag

User says: "it takes a second before anything appears", "dark screen then the page".

Check in this order:

1. **Dead progressive guard.** A `useEffect` that does `if (visible && !prevVisible) { ...; return cleanup; }` and sets `prevVisible = visible` *after* the `if`. The `return` exits the effect, so the flag never flips and the reset branch is dead code. Symptom: heavy content mounts on the first frame of every reopen.
   Grep: `prevVisible`, `prevVisibleRef`, `setContentReady`.
2. **Synchronous work in the first render.** Computing data or tables, parsing large JSON, image `getSize`, or mapping a whole catalog inside the render body or a `useMemo` that runs on open.
3. **Everything mounted in one commit.** Count what the open frame mounts: a single screen can create hundreds of native views; frames of 150–450 ms are felt as a freeze.
4. **Native modal cost.** Window creation plus content mount. A window that paints before its content is the "dark screen".
5. **Boot-time dead code.** Components mounted at app start that parse big payloads even when never shown. `if (!open) return null` does not help when the expensive `useMemo` sits above it — hooks run anyway.

Prove it: temporary timestamps at tap → render → first commit → deferred commit. The gap between them names the culprit.

## scroll-jank

1. **Memo never bails out.** `React.memo` on a row or tile, but the parent passes `onPress={() => handler(item)}`. Fix: pass the stable handler and let the child call `handler(item)`.
2. **A gesture handler on a child steals the scroll.** Pan responders that answer `true` on touch start (`onStartShouldSetPanResponder: () => true`) claim the gesture before any movement. Worse when the same state also drives `scrollEnabled={!scrubbing}`.
3. **Images decoded at full size.** Large bitmaps rendered in 40–100 px cells without `resizeMethod="resize"` (Android) or equivalent downsampling.
4. **Prefetch storms.** Walking an entire animation frame set (hundreds of MB) at open, while the user scrolls.
5. **Layout animations on the JS driver.** `useNativeDriver: false` on `maxHeight`/`height`, or `LayoutAnimation.configureNext` in an app where every tab stays mounted: the layout pass covers the whole app.
6. **Per-frame state writes.** `onScroll` calling `setState` without throttling.
7. **Shadows on many scrolling cards.** `elevation`/`shadowRadius` per card inside a non-virtualized list.

## dark-screen

The user taps and gets a dimmed or black screen with no content, sometimes with dead input.

1. **Native modal inside a native modal.** On React Native + Android, a `Modal` rendered inside another `Modal` can stay visually empty and block input. Fix: render the inner sheet in-tree (`useModal={false}`-style flag) — check how sibling call sites already do it.
2. **Backdrop unconditional, content animated in.** If the backdrop renders first and content only after an animation or timer, a dropped frame leaves a pure dim.
3. **Transparent window with off-screen content.** A `transparent` modal kept visible while its content is translated off-screen swallows touches.
4. **Modals surviving a tab change.** When screens are hidden with `display: 'none'`, a native `Modal` inside them survives and covers the destination.
5. **Full-screen dim overlay at maximum opacity** (night mode) with `pointerEvents="none"`: taps land, nothing is visible.

## idle-fps

Measure first: 3 seconds untouched, count frames.

1. **Infinite repeat animations.** `withRepeat(..., -1, true)` / `Animated.loop` on icons, shimmers, glows. They force the compositor to redraw forever.
2. **Off-screen pages mounted.** A horizontal pager that mounts both pages: the hidden page keeps animating with zero visible pixels.
3. **Hidden screens.** Tabs kept mounted with `display: 'none'` keep subscriptions, timers and animations.
4. Distinguish "by design" from "waste": if pixels do not change, it is waste.

Fix order: stop animations when the surface is not visible → stop them after a short settle window (keep the entrance effect) → only then consider removing the effect.

## memory

1. **Unvirtualized lists.** A `ScrollView` with `map()` over every item mounts every row and image. At hundreds of items this is the top OOM cause on 1–2 GB devices.
2. **Base64 in JS strings.** Reading a file into base64 (tens of MB) doubles memory; prefer a `file://` URI.
3. **Unbounded caches** keyed by content that never evicts.
4. **Duplicate mounts** (both pager pages, hidden tabs) each holding their own images.

Prove with `dumpsys meminfo` PSS before and after: open → scroll → measure. Growth that follows data volume is the risk, not the absolute MB.

## boot

1. **Serial initializers.** Independent storage reads awaited one after another: cost = sum. Run them in parallel after the scope-dependent ones.
2. **Network awaited before the first paint.** Cap it, or let it confirm after render — but never remove the cap.
3. **Dead components parsing data.** Unreachable screens that still mount and parse.
4. Order matters only for real dependencies (account scope before user storage). Do not parallelize those.

## wrong-data (found by the same audit)

Not performance, but the same read: a screen opened with default props (`lat`/`lon`, ids) instead of the caller's values. It shows plausible but wrong data, and it also misses every cache keyed on those values.

Check: every prop a caller could pass vs the defaults in the component signature.

## Cross-platform map

| Concern | React Native | Compose | Flutter | SwiftUI | Qt/QML |
|---|---|---|---|---|---|
| Virtualized list | `FlatList`/`SectionList` | `LazyColumn` | `ListView.builder` | `List`/`LazyVStack` | `ListView`, `TableView` |
| Stable callbacks | `useCallback` + `React.memo` | `remember`, stable lambdas | `const` widgets | value types | `Q_PROPERTY`, `QML_ELEMENT` |
| Infinite animation | `withRepeat(-1)` | `infiniteRepeatable` | `repeat()` | `.repeatForever` | `SequentialAnimation` loops |
| Gesture vs scroll | `Gesture.Pan` + `failOffsetY` | nested scroll | `GestureDetector` arenas | `simultaneousGesture` | `Flickable` + `DragHandler` |
| Image downsampling | `resizeMethod` | sized `Image` + decoder | `cacheWidth` | `resizableImage` | `QQuickImage.sourceSize` |

These mappings are conceptual, not verified recipe by recipe.
