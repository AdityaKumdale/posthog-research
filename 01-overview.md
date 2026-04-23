# PostHog Frontend — Exploration Guide

A curated starting point for exploring the frontend of [PostHog/posthog](https://github.com/PostHog/posthog),
compiled from the repo wiki and targeted Q&A against the codebase.

Wiki root: https://deepwiki.com/PostHog/posthog — Frontend section: §9 Frontend Architecture.

---

## 1. Tech stack at a glance

| Concern | Choice |
| --- | --- |
| UI | React 18.3 + TypeScript 5.7 |
| State | **Kea 4** (`kea-router`, `kea-forms`, `kea-loaders`, `kea-subscriptions`, `kea-waitfor`, `kea-window-values`, `kea-localstorage`) |
| Build | Vite (dev) + Esbuild (prod), pnpm + Turborepo monorepo |
| Styling | Tailwind CSS via `@posthog/tailwind`, some legacy SCSS |
| UI primitives | `@radix-ui/*`, `@base-ui/react`, internal `lemon-ui` |
| SDK | `posthog-js` for analytics calls |

The unusual piece is **Kea** — it's the lens through which almost everything
flows. Learn it first; the rest is idiomatic React.

---

## 2. Monorepo layout

```
posthog/
├── frontend/               # main React app
│   ├── src/
│   │   ├── scenes/         # top-level pages + sceneLogic
│   │   │   ├── App.tsx        # root component
│   │   │   └── scenes.ts      # URL → Scene registry
│   │   ├── layout/         # nav, sidebar, shell
│   │   ├── lib/            # shared components, hooks, utils
│   │   ├── models/         # shared Kea logics (teamLogic, userLogic, …)
│   │   ├── initKea.ts      # Kea bootstrap + plugin config
│   │   ├── products.tsx    # GENERATED — do not edit
│   │   └── products.json   # GENERATED — do not edit
│   ├── build-products.mjs  # codegen that scans products/*/manifest.tsx
│   ├── package.json
│   └── jest.config.ts
├── products/               # 24+ self-contained product packages
│   ├── logs/
│   │   ├── manifest.tsx       # registers scenes, routes, URLs, nav items
│   │   ├── frontend/          # product-local React + Kea code
│   │   └── package.json
│   ├── session-recordings/
│   ├── experiments/
│   └── …
├── common/
│   ├── esbuilder/          # prod build pipeline
│   ├── hogvm/typescript/   # HogQL VM runtime (client-side eval)
│   ├── tailwind/           # shared Tailwind preset
│   └── storybook/          # shared Storybook config
└── playwright/             # E2E tests
```

Key idea: **features are products**. Each folder under `products/` is a workspace
package with its own `package.json`, and the frontend ingests them at build time
via `frontend/build-products.mjs`, which generates `frontend/src/products.tsx`.

---

## 3. How the app boots

1. Vite (dev) or esbuild (prod) builds the SPA entry.
2. `frontend/src/initKea.ts` calls `initKea(...)` with plugins:
   `routerPlugin`, `formsPlugin`, `loadersPlugin`, `subscriptionsPlugin`,
   `waitForPlugin`, `windowValuesPlugin`, `localStoragePlugin`.
3. `frontend/src/scenes/App.tsx` mounts — it renders the global layout
   (nav shell from `frontend/src/layout/navigation-3000/…`) and delegates
   the body to `sceneLogic`.
4. `sceneLogic` watches the URL (via `kea-router`), looks up `scenes.ts`,
   and lazy-loads the matching scene. Scene configs (`projectBased`,
   `organizationBased`, `onlyUnauthenticated`, …) live in
   `frontend/src/scenes/sceneTypes.ts`.
5. Product scenes register themselves through the generated `products.tsx`
   so they plug into the same scene machinery.

---

## 4. Mental model for Kea (most important skill)

> **"Data lives in logics, not in components."** Avoid `useState`/`useEffect`
> for anything an unrelated component might need. Derive with selectors, mutate
> via actions.

A logic is a small state machine composed of named builders. Minimal shape:

```ts
// products/endpoints/frontend/endpointsLogic.tsx (real example)
export const endpointsLogic = kea<endpointsLogicType>([
  path(['products', 'endpoints', 'frontend', 'endpointsLogic']),
  props({} as EndpointsLogicProps),
  key((props) => props.tabId),                 // one instance per tab

  actions({
    setFilters: (filters: Partial<EndpointsFilters>) => ({ filters }),
    setActiveTab: (activeTab: EndpointsTab) => ({ activeTab }),
  }),

  loaders(() => ({                             // kea-loaders → auto loading/error
    allEndpoints: [[] as EndpointType[], {
      loadEndpoints: async () => (await api.endpoint.list()).results || [],
    }],
  })),

  reducers({
    filters:   [DEFAULT_FILTERS, { setFilters: (s, { filters }) => ({ ...s, ...filters }) }],
    activeTab: ['endpoints',     { setActiveTab: (_, { activeTab }) => activeTab }],
  }),

  selectors({                                  // pure, memoized derivations
    endpoints: [(s) => [s.allEndpoints, s.filters], (all, f) =>
      f.search ? new Fuse(all, { keys: ['name', 'description'] })
                    .search(f.search).map(r => r.item)
               : all,
    ],
  }),

  afterMount(({ actions }) => actions.loadEndpoints()),

  tabAwareUrlToAction(({ actions }) => ({      // kea-router hook
    [urls.endpoints()]: (_, { tab }) => {
      actions.setActiveTab(tab === 'usage' ? 'usage' : 'endpoints')
      if (tab !== 'usage') actions.loadEndpoints()
    },
  })),
])
```

### Builder cheatsheet

| Builder | Purpose |
| --- | --- |
| `path([...])` | Unique dotted path; shows up in Redux devtools |
| `props<T>()` / `key(fn)` | Parameterize + allow multiple instances |
| `actions({...})` | Payload-returning action creators |
| `reducers({...})` | Pure state → state on action |
| `loaders({...})` | Auto actions/state for async calls (`load…`, `load…Success`, `load…Failure`, `…Loading`) |
| `selectors({...})` | Memoized derived values; first arg lists dependencies |
| `listeners({...})` | Side effects & action orchestration |
| `forms({...})` | `kea-forms` — form state, validation, submit |
| `subscriptions({...})` | React to value changes |
| `connect({...})` | Import actions/values from another logic |
| `urlToAction` / `actionToUrl` | Two-way bind URL ↔ state |
| `events({ afterMount, beforeUnmount })` / `afterMount` | Lifecycle |

### Consuming from React

```tsx
import { useValues, useActions, BindLogic } from 'kea'

<BindLogic logic={endpointsLogic} props={{ tabId }}>
  {() => {
    const { endpoints, filters, allEndpointsLoading } = useValues(endpointsLogic)
    const { setFilters } = useActions(endpointsLogic)
    return /* JSX */
  }}
</BindLogic>
```

`BindLogic` is only needed for keyed logics or when children need the same
mount context; otherwise just call `useValues(logic)` directly.

### `*Type.ts` generated files

For every `fooLogic.tsx` there is a sibling `fooLogicType.ts` produced by
`kea-typegen`. The `kea<fooLogicType>([...])` line wires them together, giving
fully-typed actions/values/selectors. **Never hand-edit `*Type.ts` files** —
run typegen instead.

---

## 5. Routing

- All routing goes through `kea-router`, configured in
  `frontend/src/initKea.ts`. It handles project-ID rewriting (URLs become
  `/project/:id/...`).
- `sceneLogic` matches URLs to scene keys defined in
  `frontend/src/scenes/scenes.ts`; scene config types are in
  `frontend/src/scenes/sceneTypes.ts`.
- Product-local routes come in via each product's `manifest.tsx` and are merged
  into the main app by `build-products.mjs`.
- Inside a logic, use `urlToAction` / `actionToUrl` to stay in sync with URL
  params (e.g. filters, active tab).

---

## 6. Running the frontend locally

**Recommended (full stack):**

```bash
git clone --filter=blob:none https://github.com/PostHog/posthog && cd posthog
brew install flox          # macOS; use equivalent on Linux
flox activate              # provisions the toolchain
hogli start                # boots Docker infra + backend + frontend
```

App: http://localhost:8010

**Frontend only (against a running backend):**

```bash
pnpm install
pnpm --filter=@posthog/frontend start          # or: cd frontend && pnpm start-vite
```

`start-vite` runs Vite + Tailwind watch + the toolbar bundle concurrently.

---

## 7. Testing

| Layer | Tool | Location | Command |
| --- | --- | --- | --- |
| Unit / component | Jest + React Testing Library | co-located `*.test.ts(x)` | `hogli test frontend/src/` |
| Component catalogue | Storybook | `*.stories.tsx` + `frontend/__snapshots__/` | `hogli storybook`, `hogli storybook:test` |
| Visual regression | Storybook + Playwright | snapshots under `frontend/__snapshots__/` | via `storybook:test` in CI |
| E2E | Playwright | `playwright/**.spec.ts` | `hogli test:e2e` |

Good first files to read:
- `frontend/src/lib/components/DateFilter/DateFilter.test.tsx` — simple Jest/RTL test
- `frontend/src/scenes/insights/stories/TrendsLine.stories.tsx` — multi-viewport story
- `playwright/README.md` — E2E conventions (prefer `getByRole`/`getByTestId`, assert on UI not network)

Cypress is **not** used — all E2E is Playwright.

---

## 8. Suggested learning path

Do these in order — each step builds on the previous:

1. **Read the shell** — skim, don't deep-dive.
   - `frontend/src/scenes/App.tsx`
   - `frontend/src/initKea.ts`
   - `frontend/src/scenes/scenes.ts` + `sceneTypes.ts`
   - `frontend/src/layout/navigation-3000/navigationLogic.tsx`

2. **Learn Kea by imitation.** Pick ONE small logic and read it line by line
   with the `*Type.ts` file open next to it. Good candidates:
   - `products/endpoints/frontend/endpointsLogic.tsx` (CRUD + loaders + url sync)
   - `frontend/src/scenes/product-tours/productToursLogic.ts` (simple)
   - `frontend/src/models/userLogic.tsx` (global/shared logic)

3. **Follow one product end-to-end.** Recommended: `products/logs/` — it's
   recent, self-contained, and hits every subsystem (scene, logic, API, URL
   state, table rendering).
   - Start at `products/logs/manifest.tsx`
   - Then its `frontend/` folder → scene component → logic → API calls

4. **Trace a URL.** Paste a URL like `/project/1/insights/new` into the
   browser and trace it:
   `initKea` → `sceneLogic` → `scenes.ts` entry → scene component →
   its logic(s) → API client (`frontend/src/lib/api.ts`).

5. **Run the test for that area** (`hogli test <path>`) and tweak a
   selector/reducer to see Kea's devtools light up. Redux DevTools work
   out of the box.

6. **Make one tiny change** to earn muscle memory:
   - Add a new filter option to `endpointsLogic` and render it
   - Migrate a `useState` inside a scene component into a new Kea reducer
   - Replace a bespoke SCSS block with Tailwind classes

---

## 9. Useful anchors when you get lost

- **API client**: `frontend/src/lib/api.ts`
- **Feature flags in frontend**: `frontend/src/lib/logic/featureFlagLogic.ts`
- **User/team/org state**: `frontend/src/models/{userLogic,teamLogic,organizationLogic}.tsx`
- **Global toast/notifications**: `lemonToast` (used inside `loadersPlugin` error handler)
- **Shared UI components (lemon-ui)**: `frontend/src/lib/lemon-ui/`
- **Product manifest contract**: look at any `products/*/manifest.tsx`

---

## 10. Wiki entry points worth bookmarking

- Frontend Architecture (§9): https://deepwiki.com/PostHog/posthog#9
- Kea State Management (§9.1): https://deepwiki.com/PostHog/posthog#9.1
- Product Module System (§9.2): https://deepwiki.com/PostHog/posthog#9.2
- Build System and Tooling (§9.3): https://deepwiki.com/PostHog/posthog#9.3
- Frontend Workspace and Product Packages (§2.1): https://deepwiki.com/PostHog/posthog#2.1
- Frontend and E2E Tests (§10.3): https://deepwiki.com/PostHog/posthog#10.3
