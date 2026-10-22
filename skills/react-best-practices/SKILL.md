---
name: react-best-practices
description: "Modern React best practices and anti-pattern catalog (2025-26). Use when writing, reviewing, or refactoring React components, hooks, and state management. Covers component design and component API design (props, composition, polymorphism, slots), state ownership and immutability, hooks misuse, forms, context, performance, data fetching, accessibility, and code organization. Reference files cover SSR/hydration/Next.js App Router and Motion (framer-motion) animations."
---

# React Best Practices & Anti-Patterns

Modern React conventions (2025-26). Covers what to do and what to avoid. For code examples, see [examples.md](examples.md). For SSR/Next.js and Motion, see [Reference Files](#reference-files).

## Severity Levels

Each rule is tagged with a severity for use by consuming agents:

- **CRITICAL** — Will cause bugs, broken reconciliation, or maintenance nightmares
- **HIGH** — Will cause performance issues or scaling problems
- **MEDIUM** — Will hurt maintainability or developer experience

---

## Component Design (CRITICAL)

- Components must be pure — same inputs = same outputs, no side effects during render
- Business logic in hooks/helpers, NOT in component bodies
- Container components fetch data; presentational components receive props and render UI
- Helper functions extracted OUTSIDE the component body
- Max 200 lines per component — split if larger
- Max 5-7 props — more suggests the component does too much
- One component per file (small colocated internal helpers are fine)
- Default props via destructuring (`{ size = 'md' }`) — not `||` (breaks on `0`/`''`) and not `defaultProps` (removed for function components in React 19)

### Composition Patterns

- Compose small focused components over monolithic ones
- "Lift Content Up" — move children to the parent when the wrapper doesn't use them for logic
- "Push State Down" — keep state in the component that actually needs it
- Prefer `children` prop and composition over deep prop drilling

## Derive, Don't Store (CRITICAL)

The #1 React anti-pattern. Look for it in every review.

- NEVER store derived values in `useState` — compute during render
- NEVER use `useState` + `useEffect` to sync a computed value — just compute it
- Use `useMemo` ONLY if the computation is expensive (measured, not assumed)
- If a value can be calculated from existing props/state, calculate it inline
- Store the minimal state shape (the `guesses` array) and derive the rest (rows, status, remaining count) — updating minimal state is a direct transform; updating pre-derived state means re-running the derivation
- Don't assume a derivation is too slow to run every render — typical string/array work is imperceptible; measure first, and memoize the computation rather than stashing the result in state
- Exception: a value that is a genuinely separate concern (a `status` that will grow new states) can be its own state — don't contort it into a derivation of unrelated data

## State Immutability (CRITICAL)

React detects changes by reference — mutating state in place gives stale UI and DOM nodes in the wrong place.

- NEVER mutate state or props: no `push`/`splice`/`sort`/`reverse`, no `arr[i] = x`, no `obj.field = x`
- Clone, modify the clone, then call the setter — spread, `map`, `filter`, `toSorted()`/`toReversed()`
- `sort()` and `reverse()` mutate in place — copy first (`[...arr].sort()`)
- Don't mutate "just briefly" before calling the setter
- Don't avoid cloning for performance — copying arrays/objects per update is effectively free
- Deeply nested state: use Immer (`produce`, or `useImmerReducer`) instead of spreading every level; log drafts with `current(draft)`

## State Management (HIGH)

### State Location
- Colocate state with the components that use it
- Don't lift state higher than necessary — it causes unnecessary re-renders
- Don't duplicate state across components
- Unmounting destroys state — if state must survive a component being hidden, lift it to a parent that stays mounted (or keep the component mounted and hide it)

### State Ownership
- Don't hand a raw setter (`setItems`) to descendants — expose narrow handlers from the owner (`handleAddItem(label)`); update logic stays in one place and children can't make arbitrary changes
- Each piece of data is either controlled (consumer passes `value` + `onChange`) or uncontrolled (the component owns it) — never both
- NEVER mirror a controlled prop into local state and sync it with an effect — two sources of truth; make the component fully controlled
- A prop that only seeds initial state should read like it (`initialValue`, `defaultValue`) — later changes don't resync
- When a parent must read or reset a child's value, make the child controlled — don't bolt external control onto an uncontrolled child

### Context API
- Context is for dependency injection (auth, theme), NOT global state management
- Context changes re-render ALL consumers — split contexts by concern
- Prefer hook return values over Context when only one subtree needs the data
- Reach for Context when a prop is threaded through components that don't use it; try "Lift Content Up" first, and keep passing plain props to direct children
- Memoize the Provider `value` (`useMemo`) — an inline `{ state, setState }` is a new object every render and re-renders every consumer, even `React.memo` ones. Memoizing the Provider component itself does not help
- Bundle state + actions into one `value` object; give each context its own Provider component (`createContext` + state + `<Provider>`) and compose providers at the root, not in `App`

### State Hygiene
- Not everything needs to be in `useState` — only values that change over time and affect the UI
- Combine related state with `useReducer` instead of multiple `useState` calls
- URL-dependent state (filters, pagination, search) belongs in URL search params, not component state
- Setters are scheduled — code after `setX(...)` in the same handler still sees the old value; compute `const next = ...` first and use it
- Expensive initial value → lazy initializer `useState(() => read())`; `useState(read())` runs `read` on every render
- Server data is not client state — use a data-fetching library (TanStack Query, SWR), not `useState`/Redux/Context; keep store libraries for genuinely client-side state

### Reducers
- Use `useReducer` when grouping update logic clarifies complex state — not by default for simple state
- Reducers must be pure: no `crypto.randomUUID()`, `Date.now()` or `Math.random()` inside — generate in the handler and pass it through the action
- Actions describe what happened (`{ type: 'item-added', item }`), not a precomputed result (`{ type: 'set-items', items }`) — the reducer decides how state responds
- Wrap each `case` body in `{}` and `return` from it; the reducer is a plain function, so unit-test it without rendering

## Hooks (HIGH)

### Rules of Hooks
- Call hooks unconditionally, at the top level, in the same order every render — never inside `if`/loops or after an early return
- Put EVERY hook above the early returns (`if (loading) return ...`)
- A hook needed for only part of the UI → extract that part into a child component that calls it, or branch in the parent to render one of two child components

### useEffect Rules
Before each `useEffect`, ask: **Is there an external system being synchronized?** If no, it's misused.

- NEVER use `useEffect` for derived state — compute during render
- NEVER use `useEffect` for event handling — put logic in the event handler
- NEVER chain `useEffect`s that trigger each other — usually means derived state
- ALWAYS declare all dependencies correctly
- ALWAYS clean up subscriptions, timers, and event listeners
- NEVER pass an `async` function to `useEffect` (it returns a Promise, not a cleanup) — define an inner async function, call it, and return cleanup from the outer function
- NEVER silence `react-hooks/exhaustive-deps` — a missing dep is a stale closure; restructure instead (functional updater, move logic into the handler, split the effect)
- Long-lived callbacks (intervals, listeners in a `[]` effect) see the state of the render that created them — read the latest through the functional updater (`setOn(on => !on)`); use it when a stale value actually shows up, not everywhere "just in case"
- Split unrelated concerns into separate effects (subscribe once with `[]`; react to a value with `[value]`)
- Refs are stable — they don't belong in the dependency array
- Cleanup must undo setup: `removeEventListener`, `observer.disconnect()`, `clearInterval`/`clearTimeout`; define handlers inside the effect so cleanup closes over them
- Keep `<StrictMode>` on — its dev-only extra mount/unmount of effects exposes missing cleanup; fix the cleanup, don't remove StrictMode
- Guard with an early `return` at the top of the effect instead of wrapping the body in `if`

### Refs
- Use `useRef` for imperative work with no declarative API: `focus()`, media `play()`/`pause()`/`playbackRate`, measuring, observers — not for values that should re-render the UI
- Move focus programmatically (`ref.current.focus()` in an effect) when UI opens (modal, menu, inline editor) and restore it on close
- Leave an input uncontrolled (read via ref or form data) when its value isn't shown or used elsewhere
- Pass the ref OBJECT to custom hooks, never `ref.current` — while the component body runs, `.current` is still `undefined`; the hook reads it inside its own effect

### Custom Hooks
- Extract a state + effect + cleanup combo that implements one behavior into a `useX` hook (`useTime`, `useToggle`, `useIsOnscreen`)
- A hook that needs a DOM node either takes the caller's ref or creates and returns its own — choose by whether the caller needs the ref too
- A hook meant as a drop-in for `useState` should accept the same lazy initializer
- Return stable functions (`useCallback`) from hooks whose consumers may pass them to memoized children
- Prefer a maintained utility-hook library over re-implementing generic hooks (debounce, local storage, previous value)

### Memoization Rules
Most `useMemo`/`useCallback` calls are unnecessary. Be skeptical:

- `useMemo` — only for actually expensive computations (sorting large datasets, complex transforms)
- `useCallback` — only when the function is passed to a `React.memo` child
- Simple string concatenation, arithmetic, or boolean checks do NOT need memoization
- A parent re-render re-renders ALL descendants whatever their props; `React.memo` skips that only when props are referentially stable — a fresh object/array/function literal each render defeats it
- `useMemo` also keeps an object/array prop stable for a memoized child; pair `useCallback` with functional updates (`setCount(c => c + 1)`) so deps stay minimal
- Try structural fixes first: move fast-changing state (ticking clock, input text) into its own small component beside the expensive subtree, or pass the expensive subtree in as `children`
- Rendering is not painting — React only touches the DOM where output changed, so most cascading re-renders are cheap; don't pre-memoize

## Render Factories (CRITICAL)

camelCase functions returning JSX are NOT React components. They break reconciliation, hooks, and dev tools.

- NEVER use `renderThing()` pattern — use `<Thing />` component syntax
- ALWAYS use PascalCase for anything that returns JSX
- Render factories lose component identity on every render, causing full unmount/remount

## Inline Creation in JSX (HIGH)

New arrays, objects, and functions created inline in JSX props break `React.memo` on children.

- Extract static arrays/objects to module-level constants
- Extract dynamic arrays/objects to `useMemo`
- Extract inline functions to `useCallback` (only when passed to memoized children)

## Over-Engineering (CRITICAL)

### Premature Abstraction
- Abstractions with only one consumer are premature — inline it
- "Reusable" hooks with hardcoded field names are not reusable — accept config as parameters
- Context storing state that could be computed locally is over-engineered

### Wrapper Components
- Components that only call a hook and return `null` are unnecessary — call the hook directly
- Wrappers that only pass props through add indirection without value
- Exception: a thin facade around a third-party dependency is worth having — it isolates the API, makes swapping cheap, and is the one place to add lazy loading

## Component API Design (HIGH)

A component's props are its public API — design for the consumer, and keep naming and patterns consistent across the codebase.

### Layering
- Split the generic shape from the business rule: a low-level `Banner` plus a thin `LoggedInBanner` that adds the rule — don't bake one use case into the reusable piece
- Don't over-parameterize a one-off component — hardcode what it never varies
- Two components with a near-identical visual pattern → extract one and expose the difference as a prop
- Generic UI primitives take plain props (`name`, `avatarUrl`), not a domain object (`user`); feature components may take domain objects
- Expose meaning, not styling, for fixed variants: `status="danger"` instead of `color`/`borderColor` props — the mapping lives in one place

### Prop Delegation
- Forward unknown props with rest/spread (`{ label, ...delegated }` → `<input {...delegated} />`); low-level wrappers (`TextInput`, `Button`) spread extra props onto the native element so every HTML attribute works. Name the rest object consistently (`delegated` or `rest`)
- Order decides who wins: attributes AFTER `{...delegated}` are locked (put `type="checkbox"` there); attributes BEFORE it are overridable defaults
- `className` and `style` must be MERGED, not overridden — `className={clsx('base', className)}`, `style={{ ...baseStyle, ...style }}`; spread order can't do that
- One generated value feeding two elements (input `id` + label `htmlFor`) → compute it once: `const appliedId = id ?? generatedId`
- A `className` pass-through is usually the realistic styling API: restricting to specific style props doesn't stop determined overrides and still misses edge cases

### Semantics & Polymorphism
- Choose tags by meaning, not looks: `<button>` for actions, `<a href>` for navigation — strip default styles with CSS instead of rebuilding native behavior on a `div`
- A component that may be a link or a button picks its tag from props: `const Tag = typeof href === 'string' ? 'a' : 'button'` — precise check, because `href=""` is valid but falsy
- Heading level follows document hierarchy, not desired size — accept `level` and render `h1`–`h6`; style size separately
- For an `as` prop restrict the allowed tags; with exactly two options a boolean prop (`ordered` → `ol`/`ul`) is simpler

### Slots
- Accept JSX as a prop (`actions`, `header`) when the wrapper owns the layout but not the content; prefer a named slot over `children` when position matters at the call site
- Accept a component reference (`icon={Award}`), not an element, when the wrapper must control the props it renders with — don't reach for `cloneElement`
- Don't give a reusable leaf a giant internal lookup (string → every icon) — every usage then bundles every option; pass the component in so only used ones ship

### Overlays & Complex Widgets
- Don't hand-roll accessibility-heavy widgets (modal, combobox, menu, tooltip, popover, accordion) — use a headless library (Radix Primitives, React Aria, Headless UI, Ariakit, Base UI) and apply your own styles; delete the hand-rolled equivalents when adopting one
- Avoid pre-styled UI kits (Material UI) when a distinct brand matters — their design system is hard to override later
- Check which major version a tutorial or snippet targets — headless library APIs change between majors
- Render overlays with `createPortal` (from `react-dom`) so ancestor `overflow`, `transform`, `filter`, `will-change` or stacking contexts can't clip or offset them; context and event bubbling still follow the React tree, and portaling into `document.body` is fine
- Tooltip describes a control; Popover is the general-purpose floating-content component

## Forms (HIGH)

- Wrap controls in a `<form>` and handle `onSubmit` (with `event.preventDefault()`), not `onClick` on the button — Enter-to-submit and native validation (`required`, `minLength`, `pattern`) come free. React 19 also offers form Actions (`<form action>`, `useActionState`, `useFormStatus`) — check the project's React version
- A controlled input needs BOTH `value` (or `checked`) and `onChange`; text state starts as `''`, never `undefined` (`value={x ?? ''}`) — an input can't switch between uncontrolled and controlled
- Intentionally uncontrolled inputs use `defaultValue`/`defaultChecked`, not `value`/`checked`
- `type="range"`/`type="number"` values arrive as strings — convert (`Number(...)`) before storing
- Generate `<option>`/radio/checkbox lists from one data source instead of hand-typing values that must match state; in the `.map`, don't name the callback parameter after the state variable (shadowing)
- Checkbox groups: state as an object map (`{ a: true, b: false }`), toggled immutably
- Link labels to inputs with one `useId()` per component (`${id}-email`) — never hardcoded `id`s, which collide when the component renders twice; `useId` is SSR-safe, `Math.random()` and counters are not
- Don't disable the submit button to enforce validity — it gives no feedback on why; keep it enabled and show the validation message in the UI (not `window.alert`)
- While a request is in flight, disable the controls and show progress; clear fields only after confirmed success

## Data Fetching (HIGH)

- ALL data fetching in custom hooks, never in component bodies
- Use the project's existing data-fetching layer (shared query/mutation hooks, or a library such as TanStack Query/SWR) — don't introduce a parallel one
- Handle loading, error, and empty states in the container component
- Use try-catch in async functions within hooks
- Model request state as one status (`'idle' | 'loading' | 'success' | 'error'`), not parallel booleans that can contradict each other
- Prefer a data-fetching library (TanStack Query, SWR) over hand-rolled `useEffect` + `fetch` for fetch-on-mount — caching, dedupe and revalidation are easy to get wrong. Plain `fetch` in the event handler is fine for user-triggered requests
- `fetch` does not reject on HTTP errors — check `response.ok` AND catch network failures; with SWR/TanStack Query the fetcher must `throw` for `error` to be populated
- Encode user input placed in URLs (`URLSearchParams` / `encodeURIComponent`)

## Styling (MEDIUM)

- Use the project's styling system consistently (Tailwind, CSS Modules, CSS-in-JS) — don't introduce a second one
- Reserve inline `style={}` objects for genuinely dynamic values (computed sizes, CSS variables) — not static styling
- Extract repeated style combinations into reusable components (Button, Card, Badge)
- Prefer the project's existing shared UI components over recreating common elements
- Join conditional classes with `clsx`/`cn(...)` — falsy values drop out; avoid hand-built template strings (stray `false`/`undefined`, whitespace typos)
- Prefer CSS breakpoints (`md:hidden` in Tailwind, media queries elsewhere) over JS media-query hooks for show/hide — no SSR mismatch, no extra renders

### Tailwind
- Use utility classes for all styling — no inline `style={}` objects
- Use responsive prefixes (`sm:`, `md:`, `lg:`) for responsive design

### CSS Modules
- Name stylesheets `*.module.css` — class names are scoped per file, so no BEM-style naming convention is needed
- Look up a class by a variable with bracket notation (`styles[variant]`); dot notation only works for a literal key
- Combine a base and a variant class with `clsx(styles.base, styles[variant])`; never `className={cond && styles.x}` — when false it yields `false`

## Error Boundaries (HIGH)

- Use `react-error-boundary` (or the framework's convention, e.g. Next.js `error.js`) for a function-component-friendly API
- Include `resetKeys` tied to the route (e.g. `resetKeys={[location.pathname]}`) so boundaries reset on navigation
- Provide a "Try again" button that calls `resetErrorBoundary` in fallback UI
- Error boundaries do NOT catch errors in event handlers, async code, or SSR — use try/catch there
- Place boundaries at the blast radius you want to contain (a flaky widget, a page section) — one catches everything below it; nest several for fault lines, the nearest wins
- Report from the boundary (`onError` → Sentry/LogRocket) so users get a fallback AND you get the error

## Key Prop Patterns (CRITICAL)

- NEVER use array index as `key` when lists can be reordered, filtered, or modified
- NEVER use `Math.random()` or other unstable values as keys — causes full unmount/remount
- When mapping Fragments, put `key` on `<React.Fragment key={id}>`, not on a child
- Generate IDs when the item is CREATED (`crypto.randomUUID()` in the handler) and store them on the item — not during render; `array.length` as an id duplicates after a delete
- Use `key` deliberately to reset: a changed key unmounts and remounts the subtree — `<ProfileForm key={userId} />` resets its state without an effect, and a changing key retriggers a one-shot CSS animation. Key off a value that changes only for the triggering action
- Instance identity is type + position (+ key): the same component at the same position keeps its state across conditional branches; changing its type or position remounts it and loses state

## Conditional Rendering (HIGH)

- NEVER use `{count && <Component />}` when `count` can be `0` — renders literal `0`
- Use `{count > 0 && <Component />}` or ternary instead
- Replace nested ternaries with early returns or extracted components
- For multiple UI states (loading/error/empty/success), use early returns pattern
- `undefined`/`null`/`false` as an attribute or child is omitted — use it to set props conditionally (`aria-describedby={error ? id : undefined}`)
- Prefer one ternary over two opposing `&&` checks
- Default to adding/removing elements over `display: none`; switch to CSS hiding only after measuring a real problem
- Gate a whole optional section on one check (`items.length > 0 && (...)`)

## Accessibility (HIGH)

- Add `aria-label` to icon-only buttons — without it, they're invisible to screen readers
- Link error messages to fields with `aria-describedby` and `aria-invalid`
- Use `aria-live="polite"` for dynamic content updates (search results, notifications, toasts)
- Trap focus inside modals; provide escape path (Escape key + visible Close button)
- Announce route changes for screen readers (SPA navigation is silent by default)
- Information conveyed only visually (a colored status dot) needs text for screen readers — use visually-hidden text (Tailwind's `sr-only` or a `VisuallyHidden` component), not `aria-label` on a `div`/`span` (ignored by screen readers, skipped by auto-translation)
- Modal: `role="dialog"`, `aria-modal="true"`, `aria-label`/`aria-labelledby`; move focus in on open, restore it to the trigger on close; close on Escape and backdrop click; lock background scroll. Prefer a headless Dialog over hand-rolling
- Different widgets have different correct initial focus (a drawer menu isn't a modal dialog) — follow the ARIA authoring pattern for the specific widget
- Anything revealed on hover (`onMouseEnter`) also needs `onFocus`, and must reset on `onBlur` as well as `onMouseLeave`
- Respect `prefers-reduced-motion` — the OS setting does nothing on the web unless you opt in (CSS media query; Motion's `MotionConfig`)

## Performance Beyond Memoization (MEDIUM)

- Use `React.lazy()` + `<Suspense>` for route-level code splitting (unless the framework already splits by route, as Next.js does)
- Split vendor bundles for better caching (Vite `manualChunks`, webpack `splitChunks`)
- Use top-level static paths in `lazy(() => import('./X'))` — dynamic paths break build analysis
- Group related async sections under ONE `<Suspense>` boundary so they appear together instead of one spinner per widget; wrap only the slow section, not the whole page
- Lazy-load heavy, rarely used dependencies; lazy loading helps a slow download, streaming helps a slow server — different problems
- Check a dependency's bundle cost (Bundle Phobia, bundle analyzer) before adding it

## HTTP Client Patterns (HIGH)

- Cancel in-flight requests in `useEffect` cleanup using `AbortController` (both `fetch` and Axios accept a `signal`)
- Use one centralized HTTP client (`baseURL`, default headers, interceptors/middleware) instead of ad-hoc calls scattered through components
- Attach auth tokens in a request interceptor/middleware and handle 401/403 in a response interceptor

## React 19 Patterns (MEDIUM)

- Accept `ref` as a regular prop instead of using `forwardRef` (React 19+)
- With React Compiler enabled, avoid adding `memo`/`useMemo`/`useCallback` unless measured
- Context: render `<ThemeContext value={...}>` directly (no `.Provider`); `use(ThemeContext)` can be called conditionally, unlike `useContext`
- On React 18 and earlier, ref forwarding needs `forwardRef`, and EVERY wrapper layer needs its own (`memo(forwardRef(...))` when combining)

## Code Organization (MEDIUM)

### Feature-Based Structure
- Colocate component + hook + helpers + tests per feature
- Shared utilities and UI primitives go in the project's shared directories (e.g. `utils/`, `components/ui/`)

### Component Directories
- Keep shared components in one flat directory — don't sort by abstraction level (`atoms/`, `molecules/`); components sit on a spectrum from generic to business-specific
- Colocate a private sub-component with its only consumer (`AccountPage/MyAccountInfo.tsx`), not in the shared directory
- Use an import alias (`@/`) over `../../../` chains

### File Quality
- Order: imports, constants, helpers, component, exports
- Reuse existing types and constants over creating new ones

## Reference Files

- [examples.md](examples.md) — good/bad code for the rules above
- [references/ssr-and-nextjs.md](references/ssr-and-nextjs.md) — read when the project uses SSR or the Next.js App Router: hydration safety, rendering strategy, Server vs Client Components, Suspense/streaming, lazy loading
- [references/motion-animations.md](references/motion-animations.md) — read when the project uses Motion (formerly Framer Motion): layout and shared-layout animations, reduced motion, troubleshooting
