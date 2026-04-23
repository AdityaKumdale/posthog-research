# PostHog frontend · Kea deep-dive — `endpointsLogic` line-by-line

> Companion to `01-overview.md`. Read this after the overview, as a
> concrete walkthrough of one real Kea logic.

Source (PostHog `main` as of 2026-04-23):
[`products/endpoints/frontend/endpointsLogic.tsx`](https://github.com/PostHog/posthog/blob/main/products/endpoints/frontend/endpointsLogic.tsx)

Full file (90 lines) — we'll annotate each chunk.

```ts
import { actions, afterMount, kea, key, path, props, reducers, selectors } from 'kea'
import { loaders } from 'kea-loaders'

import api from 'lib/api'
import { tabAwareUrlToAction } from 'lib/logic/scenes/tabAwareUrlToAction'
import { createFuse } from 'lib/utils/fuseSearch'
import { urls } from 'scenes/urls'

import { EndpointType } from '~/types'

import type { endpointsLogicType } from './endpointsLogicType'
```

## Imports (lines 1–11)

| Import | What it is | Why it's here |
| --- | --- | --- |
| `kea` | Factory that turns a list of *builders* into a logic object | The core of every logic file |
| `actions, afterMount, key, path, props, reducers, selectors` | Builders from core Kea | Compose the logic's behaviour |
| `loaders` (from `kea-loaders`) | Builder for async data + auto loading/error state | Fetches endpoint list from API |
| `api` (`lib/api`) | Typed HTTP client auto-scoped to current project | Issues the `GET /api/projects/:id/endpoint` call |
| `tabAwareUrlToAction` | PostHog's wrapper around `kea-router`'s `urlToAction` | Same logic instance can live in multiple browser-tab contexts; this version keys URL→action bindings by the `tabId` prop |
| `createFuse` | Thin wrapper around Fuse.js | Fuzzy search across the endpoints list |
| `urls` | Centralized URL builder (`urls.endpoints()` → `/project/:id/endpoints`) | Single source of truth for route strings |
| `EndpointType` (from `~/types`) | Shared TS type | Strongly types loader return & selector output |
| `endpointsLogicType` (from `./endpointsLogicType`) | **Generated** by `kea-typegen` | Provides typing for actions/values/selectors — never edit by hand |

> `~/` resolves to `frontend/src/` via a tsconfig path alias.

---

```ts
export type EndpointsTab = 'endpoints' | 'usage'

export interface EndpointsFilters {
    search: string
}

export const DEFAULT_FILTERS: EndpointsFilters = {
    search: '',
}

export interface EndpointsLogicProps {
    tabId: string
}
```

## Types & constants (lines 13–25)

- `EndpointsTab` — discriminated union used by the `activeTab` reducer.
- `EndpointsFilters` — shape of the filter reducer. Today it's just `search`, but defining an interface makes it painless to grow (`{ search, owner, status, … }`).
- `DEFAULT_FILTERS` — explicit default so the reducer's initial value is trivially correct.
- `EndpointsLogicProps` — **props the logic accepts**. Here, a `tabId` string. Kea props are *not* React props — they're passed at mount time via `BindLogic logic={...} props={...}` or `logic({ tabId }).mount()`.

---

```ts
export const endpointsLogic = kea<endpointsLogicType>([
    path(['products', 'endpoints', 'frontend', 'endpointsLogic']),
    props({} as EndpointsLogicProps),
    key((props) => props.tabId),
```

## `kea<…>([...])` + first three builders (lines 27–30)

- **`kea<endpointsLogicType>(...)`** — the generic parameter locks in the generated type. If you add an action and forget to rerun typegen, TypeScript complains immediately.
- **`path([...])`** — unique identifier in the Redux store. Shows up in Redux DevTools as `products.endpoints.frontend.endpointsLogic` and is used for hot-reloading and plugin bookkeeping. Convention: mirror the file path.
- **`props({} as EndpointsLogicProps)`** — declares the prop *shape*. The `{}` is a dummy runtime value; only the type matters. This is how the generated type file learns the prop type.
- **`key((props) => props.tabId)`** — makes the logic *keyed*. Each distinct `tabId` gets its own independent state slice. When you mount `endpointsLogic({ tabId: 'a' })` and `endpointsLogic({ tabId: 'b' })`, they live side-by-side — you can have two endpoints tabs open, each with its own filters and loading state.

---

```ts
    actions({
        setFilters: (filters: Partial<EndpointsFilters>) => ({ filters }),
        setActiveTab: (activeTab: EndpointsTab) => ({ activeTab }),
    }),
```

## `actions` (lines 31–34)

Actions are **named events** other parts of the logic (or outside consumers) can fire.

- The function on the right is an **action creator / payload builder**. Whatever it returns becomes the action payload.
- `setFilters: (filters) => ({ filters })` — takes a `Partial<EndpointsFilters>` and wraps it in an object. Partial means callers can send `{ search: 'foo' }` without specifying every field.
- Consumers call them via `useActions(endpointsLogic)`: `setFilters({ search: 'foo' })`.
- Internally Kea dispatches `{ type: 'set filters (products.endpoints…)', payload: { filters: { search: 'foo' } } }`.

Pattern rule of thumb: **actions describe intent, not state transitions.** "User typed in search box" → `setFilters`, not `setSearchStringTo`.

---

```ts
    loaders(() => ({
        allEndpoints: [
            [] as EndpointType[],
            {
                loadEndpoints: async () => {
                    const response = await api.endpoint.list()
                    return response.results || []
                },
            },
        ],
    })),
```

## `loaders` (lines 35–45) — the most powerful Kea plugin

One block gives you **six things for free**:

1. A value `allEndpoints: EndpointType[]` with default `[]`.
2. An action `loadEndpoints()` that, when dispatched, runs the async function.
3. A boolean value `allEndpointsLoading` — auto-toggled `true` when the async fn starts, `false` when it resolves/rejects.
4. A success action `loadEndpointsSuccess({ allEndpoints })` fired when the promise resolves.
5. A failure action `loadEndpointsFailure({ error })` fired on rejection.
6. **Global error handling** — `loadersPlugin` in `frontend/src/initKea.ts` wires failures to `lemonToast.error(...)` so every API error becomes a toast unless you override.

The arrow `loaders(() => ({…}))` is a factory that receives `({ actions, values, props })` so loaders can reference other parts of the logic. Here no destructuring is needed.

You'll see `useValues(endpointsLogic).allEndpointsLoading` consumed in the scene to drive the Reload button's spinner (see `Endpoints.tsx`, the `loading={allEndpointsLoading}` prop on `LemonTable`).

---

```ts
    reducers({
        filters: [
            DEFAULT_FILTERS as EndpointsFilters,
            {
                setFilters: (state, { filters }) => ({ ...state, ...filters }),
            },
        ],
        activeTab: [
            'endpoints' as EndpointsTab,
            {
                setActiveTab: (_, { activeTab }) => activeTab,
            },
        ],
    }),
```

## `reducers` (lines 46–59)

Each entry is a tuple `[initialValue, handlerMap]`.

- **`filters`** — initial = `DEFAULT_FILTERS`; when `setFilters` is dispatched, return `{ ...state, ...filters }`, i.e. shallow-merge the partial update. This is why callers can send just `{ search }` without clobbering future fields.
- **`activeTab`** — initial = `'endpoints'`; when `setActiveTab` fires, replace with the new tab.
- Handlers are **pure functions** of `(state, payload)`. They must not call actions, hit the network, or mutate `state`. Under the hood this is a Redux reducer.

Notice: `allEndpoints` is *not* in `reducers`. The loader manages it.

---

```ts
    selectors({
        endpoints: [
            (s) => [s.allEndpoints, s.filters],
            (allEndpoints, filters) => {
                if (!filters.search) {
                    return allEndpoints
                }

                const fuse = createFuse<EndpointType>(allEndpoints, {
                    keys: ['name', 'description', 'query.query'],
                    threshold: 0.3,
                })
                return fuse.search(filters.search).map((result) => result.item)
            },
        ],
    }),
```

## `selectors` (lines 60–75) — derived, memoized state

Each selector is `[dependencyFn, computeFn]`:

- **Dependency function** `(s) => [s.allEndpoints, s.filters]` — `s` is a typed proxy of every value available in this logic (loader values, reducer values, other selectors). Return the list of inputs.
- **Compute function** — runs only when a dependency *reference* changes. Memoized with `reselect` under the hood.
- When `filters.search` is empty, short-circuit and return the full list unchanged — **returning the same reference lets downstream components skip re-renders**.
- Otherwise build a Fuse index with weighted keys (`name`, `description`, nested `query.query`) and return the filtered items.

> Rule of thumb: **never do filtering/sorting in a component** — do it in a selector. You get memoization + testability.

Downstream, `Endpoints.tsx` does:

```ts
const { endpoints, allEndpointsLoading, filters } = useValues(endpointsLogic({ tabId }))
```

Note: `endpoints` is a *different* value from `allEndpoints`. The first is filtered; the second is the raw loader state.

---

```ts
    afterMount(({ actions }) => {
        actions.loadEndpoints()
    }),
```

## `afterMount` lifecycle (lines 76–78)

Runs exactly once when the logic first mounts into the store (first consumer calls `useValues`/`useActions`/`BindLogic`). Here it kicks off the initial data load. Equivalent shorthand for:

```ts
events(() => ({ afterMount: () => { actions.loadEndpoints() } }))
```

The counterpart `beforeUnmount` runs when the last consumer unmounts — good for cleanup (timers, subscriptions). Not needed here.

---

```ts
    tabAwareUrlToAction(({ actions }) => ({
        [urls.endpoints()]: (_, searchParams) => {
            if (searchParams.tab === 'usage') {
                actions.setActiveTab('usage')
            } else {
                actions.setActiveTab('endpoints')
                actions.loadEndpoints()
            }
        },
    })),
])
```

## `tabAwareUrlToAction` (lines 80–90) — URL → state sync

- `tabAwareUrlToAction` is PostHog's wrapper around `kea-router`'s `urlToAction`. It ensures each URL handler is scoped to the current browser tab's `tabId` prop, so opening the same scene in two tabs doesn't cross-fire.
- Shape: `{ [pattern]: (pathParams, searchParams, hashParams) => void }`.
- `[urls.endpoints()]` resolves to something like `/project/:id/endpoints`. The handler fires whenever the URL matches.
- Behaviour:
  - `?tab=usage` → switch to the `usage` tab (no reload; `endpointsUsageLogic` handles its own data).
  - anything else → switch to `endpoints` tab **and** re-fire `loadEndpoints()` so returning to the tab shows fresh data.
- There's no `actionToUrl` here — URL updates flow in the other direction via `<LemonTabs>` links (`link: urls.endpoints()`), so the router updates naturally when you click a tab.

---

## How a React component consumes `endpointsLogic`

From [`Endpoints.tsx`](https://github.com/PostHog/posthog/blob/main/products/endpoints/frontend/Endpoints.tsx):

```tsx
export const EndpointsTable = ({ tabId }: { tabId: string }) => {
    const { setFilters, loadEndpoints }           = useActions(endpointsLogic({ tabId }))
    const { endpoints, allEndpointsLoading, filters } = useValues(endpointsLogic({ tabId }))
    // ...
    return (
        <LemonInput value={filters.search}
                    onChange={(x) => setFilters({ search: x })} />
        <LemonButton onClick={() => loadEndpoints()} loading={allEndpointsLoading}>Reload</LemonButton>
        <LemonTable dataSource={endpoints} loading={allEndpointsLoading} columns={columns} />
    )
}
```

And at the scene level ([`EndpointsScene.tsx`](https://github.com/PostHog/posthog/blob/main/products/endpoints/frontend/EndpointsScene.tsx)):

```tsx
<BindLogic logic={endpointsLogic} props={{ key: 'endpointsLogic', tabId: tabId || '' }}>
  {/* children use endpointsLogic via useValues/useActions without passing tabId again */}
</BindLogic>
```

`BindLogic` supplies the props once; keyed logics require the same key/prop every time they're called, so `BindLogic` centralizes that.

---

## Data-flow summary (one-screen mental model)

```
     user types in search box
             │
             ▼
     setFilters({search: 'x'})     ◄── action
             │
             ▼
     reducers.filters ── merges ── { search: 'x' }
             │
             ▼
     selectors.endpoints ── recomputes ── fuzzy-filtered list
             │
             ▼
     useValues(endpointsLogic) re-renders <LemonTable>
```

```
     scene mounts                  URL change (?tab=usage)
         │                                │
         ▼                                ▼
     afterMount ──► loadEndpoints ◄── tabAwareUrlToAction
                     │
                     ▼ (kea-loaders)
          allEndpointsLoading=true
                     │
                     ▼
               api.endpoint.list()
                     │
                ┌────┴─────┐
                ▼          ▼
        Success          Failure
        allEndpoints      lemonToast.error (via loadersPlugin)
        loadEndpointsSuccess
        allEndpointsLoading=false
```

---

## Exercises to cement it

1. **Add a loading-aware empty state.** `allEndpointsLoading` is already exposed; check where it's used and make the empty-state copy distinguish "loading vs genuinely empty".
2. **Add a second filter.** Extend `EndpointsFilters` with `owner: string | null`, add a handler on `setFilters` (already generic enough — nothing to change!), update the `endpoints` selector to filter by owner, and add a dropdown in `Endpoints.tsx`. Notice how **no action plumbing changes** because `setFilters` takes a `Partial`.
3. **Add `actionToUrl`.** Make `setFilters({ search })` push `?search=` into the URL so searches are shareable. Pair with the existing `tabAwareUrlToAction` to read it back on mount.
4. **Mount two instances.** In a Storybook story, mount `<BindLogic logic={endpointsLogic} props={{ tabId: 'A' }}>` and `<BindLogic logic={endpointsLogic} props={{ tabId: 'B' }}>` side-by-side and verify their filters are independent — this is `key()` at work.
5. **Trace it in Redux DevTools.** Open the app, find `products.endpoints.frontend.endpointsLogic`, type into the search box, and watch each action fire.
