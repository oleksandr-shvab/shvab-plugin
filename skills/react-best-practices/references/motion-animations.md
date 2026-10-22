# Motion (formerly Framer Motion)

Read when the project uses Motion for layout or shared-element animations. Severity tags match [SKILL.md](../SKILL.md). API facts (import path, `layout`/`layoutId`/`LayoutGroup`, `layoutRoot`/`layoutScroll`, `MotionConfig` props, reduced-motion behavior, inline-element caveat) were checked against motion.dev docs; the remaining tips come from course notes.

- Import from `motion/react` (the library was renamed from Framer Motion; older code and tutorials import `framer-motion`)
- Use Motion where CSS can't animate: layout changes (`justify-content`, reordering) and DOM moves between parents. For plain property transitions, CSS is enough

## Basics (MEDIUM)

- Swap the element for its motion twin (`<div>` → `<motion.div>`) and drive it with `animate={{ y: open ? 0 : 60 }}`
- Springs (the default) feel livelier than easing curves — tune with `transition={{ type: 'spring', stiffness, damping }}` using a spring visualizer, not guesswork
- `initial={false}` skips the enter animation on mount
- Define a shared fallback with `<MotionConfig transition={...}>`

## Layout Animations (HIGH)

- Add `layout` to animate position and size changes caused by a re-render; `layout="position"` animates position only (right for text and images whose aspect ratio changes)
- Layout animation works through transforms, which distort children — give distorted children their own `layout` (`layout="position"` for text/images)
- Shrinkwrap elements that contain text: Motion only knows the bounding box, so an oversized box makes text snap or stretch
- Nested motion components don't inherit a parent's `transition` — repeat it or share it via `MotionConfig`; in a shared-layout animation the transition on the element being animated TO is used
- `display: inline` elements can't be transformed and won't animate — use `inline-block` or `block`
- Set `borderRadius` and `boxShadow` through `style` or an animation prop (`initial`/`animate`), not only CSS classes, so Motion can correct their distortion
- Inside a scrollable ancestor add `layoutScroll` to it; inside a fixed-position ancestor add `layoutRoot` to it — otherwise measurements are off

## Shared Layout — `layoutId` (HIGH)

- The same `layoutId` on two elements (even in different components, mounted at different times) animates one into the other: sliding nav highlights, an item moving between containers. If the old element stays mounted the two crossfade; wrap in `AnimatePresence` to animate exits
- Don't hardcode `layoutId`; derive it with `useId()` (`${id}-${item.id}`) so multiple instances of a component don't hijack each other. Exception: one deliberately shared id for the same state moving between two containers — then that component can't render twice on a page
- `layoutId` must be truthy and consistent with `key` — a changing `key` unmounts and remounts instead of animating
- Wrap related layout/`layoutId` components in `<LayoutGroup>` so a layout change in one triggers coordinated animation in the others — otherwise elements teleport or animate independently
- Track items that move between containers by a real, stable id from the data, not an index
- When several siblings can each render the shared element, raise the active one's `zIndex` for the duration of the transition
- Use different springs for different jobs: slower for moving between distinct containers, default for reshuffling within one; vary `damping` by index (`40 + index * 5`) for a cheap stagger
- Rapidly retriggered layout changes can glitch — use a faster spring, or briefly disable the trigger
- Pair every hover trigger (`onMouseEnter`) with `onFocus`, and reset on `onBlur` as well as `onMouseLeave`

## Reduced Motion (HIGH)

The OS "reduce motion" setting does nothing on the web unless you opt in.

- Wrap the app in `<MotionConfig reducedMotion="user">` — it disables transform and layout animations while keeping opacity and color animations. Values: `"user"`, `"always"`, `"never"` (default `"never"`)
- `useReducedMotion()` returns a boolean for finer control (e.g. swap movement for a fade); `MotionConfig` alone is enough for most apps
- Plain CSS animations: gate them with `@media (prefers-reduced-motion: no-preference)`
- Test by emulating `prefers-reduced-motion` in DevTools (command palette → "reduce")
- Next.js App Router: `MotionConfig` is client-only — render it from a tiny `'use client'` wrapper and import that into the Server `layout` (see [ssr-and-nextjs.md](ssr-and-nextjs.md))

## Troubleshooting

| Symptom | Fix |
|---|---|
| Nested text/content stretches or warps during a layout animation | Give it its own `layout` (`layout="position"` for text/images) |
| Text snaps to a new position at the start of a transition | Shrinkwrap the element so its box matches the content |
| Nested motion elements jiggle independently | Same `transition` on every nested motion component |
| Corners twitch | Set `borderRadius`/`boxShadow` via `style` or an animation prop |
| Shared-layout element teleports | Wrap the related elements in `<LayoutGroup>` |
| Shared-layout element blinks out mid-transition | Keep `layoutId` and `key` consistent; `layoutId` must be truthy |
| Nothing animates | Element is `display: inline`, or it didn't actually re-render |
| Layout is off inside a scroll or fixed container | `layoutScroll` on the scroller / `layoutRoot` on the fixed element |
