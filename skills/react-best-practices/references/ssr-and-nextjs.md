# SSR, Hydration & Next.js (App Router)

Read when the project renders on the server (Next.js, Remix, any SSR). Severity tags match [SKILL.md](../SKILL.md).

Next.js changes quickly. Items marked *(version-sensitive)* can differ between releases — confirm against the installed version's docs. Only the `ssr: false` and segment-config (`dynamic`) notes were checked against the Next.js docs (Next 16 era); the rest comes from course notes and general knowledge.

---

## Hydration Safety (CRITICAL)

The first client render must produce exactly the HTML the server sent.

- NEVER read `window`, `document`, `localStorage`, `Date.now()` or `Math.random()` during render — there is no `window` on the server (crash), and values that differ between server and client cause hydration mismatches
- `'use client'` components still render on the server first — the directive means "may use hooks and state", not "browser only"
- Client-only value: initialize state to a safe default (or `null` + a placeholder/spinner) on both sides, then set the real value in `useEffect` after mount. Prefer `null` + placeholder over flashing a wrong default
- Content that legitimately differs (a live clock): `suppressHydrationWarning` on that one element (works one level deep) — don't fight the mismatch
- Invalid HTML nesting breaks hydration too (`<div>` inside `<p>`, anything after `</body>`)
- Show/hide by screen size with CSS (media queries, Tailwind breakpoints), not JS hooks like `useMediaQuery` — no server/client disagreement, and the component can stay a Server Component
- Persisted theme: `localStorage` is client-only, so server HTML always flashes the default theme. Store the choice in a cookie (set an explicit expiry, or it dies with the tab), read it on the server (`await cookies()` in Next 15+), and apply it to `<html>` in the first render. On toggle, update `document.documentElement` directly for instant feedback and persist the cookie — a server round-trip (`router.refresh()`) is too slow to feel instant
- Use `hydrateRoot`, not `createRoot`, when the server already produced HTML (frameworks do this for you)

## Rendering Strategy (HIGH)

- Choose per route: on-demand SSR (fresh per request, slower), static generation/SSG (built once, fastest, but no per-user content, stale until rebuild, build time grows with page count), ISR (static, revalidated later)
- Don't statically generate personalized, auth-gated, or fast-changing pages
- `next dev` always renders on demand; production picks static vs dynamic per route. "Works in dev, frozen in prod" (e.g. a server timestamp) is expected — verify with `next build && next start`
- Forcing a strategy *(version-sensitive)*: `export const dynamic = 'force-dynamic' | 'force-static' | ...` works up to Next 15 (and 16 without Cache Components). With Cache Components enabled in Next 16, `dynamic`, `revalidate` and `fetchCache` are removed — use the caching APIs in that version's docs
- Small, known set of dynamic params → `generateStaticParams()` to pre-render them at build time
- Don't use the local filesystem for persistence in production (read-only on most hosts) — use a database. Files read at runtime on serverless hosts may need `outputFileTracingIncludes` in `next.config` (otherwise `ENOENT` in prod only)

## Server & Client Components (HIGH)

- App Router components are Server Components by default: no state, effects, context, or browser APIs. Add `'use client'` only at the interactive leaf
- Keep client boundaries small: extract the interactive part; keep heavy libraries (syntax highlighting, markdown) in Server Components so they never reach the client bundle
- Measure before refactoring (`next build` output, `@next/bundle-analyzer` behind `ANALYZE=true`) — the biggest ~20% of components give most of the win; leaving low-impact ones as Client Components is fine
- Providers that need client APIs (Context, Motion's `MotionConfig`): put them in a tiny `'use client'` wrapper that renders `{children}`, and import it into the Server `layout`. Don't mark the whole layout `'use client'` — it pulls the tree into the client bundle and blocks `metadata` exports from that file
- Share state between a server-rendered tree and client leaves with Context defined in a Client Component; Server Components passed as `children` stay on the server
- Props crossing Server → Client must be serializable — no functions (except Server Actions)
- Don't dot into a client module from a Server Component (`Menu.Item` where `Menu` is a Client Component) — it throws. Export parts as named exports or use a namespace import (`import * as Menu`)
- In new App Router code, fetch in Server Components rather than prop-drilling page-level data. If a tutorial uses `getServerSideProps` or `next/router`, it targets the Pages Router and is likely outdated

## Suspense & Streaming (HIGH)

- Group related async sections under ONE `<Suspense>` boundary so they appear together instead of popping in one by one ("spinner hell"); a boundary catches suspension from anywhere below it
- Wrap just the slow section, not the whole page — fast content streams immediately, the slow part follows
- `loading.js` beside `page.js` is an automatic Suspense boundary for the route; share skeleton markup with the page through a common component
- When several components under one boundary need the same slow data, wrap the fetch in `React.cache` so it runs once per request
- Streaming helps when the SERVER is slow to produce HTML; lazy loading helps when the CLIENT is slow to download JS

## Routing, Data & Metadata (MEDIUM)

- Use `<Link>` (`next/link`) for internal navigation — a drop-in for `<a>`; never nest an `<a>` inside it. It prefetches, keeps real crawlable anchors, and announces route changes to screen readers
- Navigate from code with `useRouter` from `next/navigation` (not `next/router`): `router.push('/path')`
- Next 15+: `params`, `searchParams`, `cookies()` and `headers()` are async — `await` them *(version-sensitive)*
- Dynamic titles: export async `generateMetadata({ params })`; layouts set defaults, pages override
- Page and `generateMetadata` calling the same fetch doubles the work — wrap the shared function in `React.cache` (dedupes within one request only; resets between requests, so no cross-user leakage)
- Route Groups `(name)` organize folders without changing the URL
- Mutations: Route Handlers (fetch-based) or Server Actions

## Lazy Loading (MEDIUM)

- `React.lazy(() => import('./X'))` inside a `<Suspense>`; the path must be a static string — bundlers analyze `import()` at build time, so variables and template strings break
- Lazy loading doesn't change SSR output — server HTML still contains the component. The win is client JS priority: the rest of the page becomes interactive sooner
- `next/dynamic` is `React.lazy` + `Suspense` with a `loading` option. `ssr: false` (for browser-only libraries) is NOT allowed in Server Components and throws — call `dynamic()` from a Client Component
- Dynamically importing a Client Component from a Server Component isn't code-split
- Bake lazy loading into a thin wrapper you own so every consumer gets it for free; the same facade makes swapping the library cheap

## Tooling (MEDIUM)

- Check a dependency's bundle cost (Bundle Phobia) before adding it; inspect real bundles with `@next/bundle-analyzer`
- Use the `@/` import alias over deep `../../../` chains
- Reproduce a failed deploy locally with `next build && next start` before debugging the host
