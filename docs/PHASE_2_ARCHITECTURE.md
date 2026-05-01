# Phase 2 — Workspace state architecture

> Companion to [`WORKSPACE_CONTEXT.md`](./WORKSPACE_CONTEXT.md).
> That doc covers the multi-tab controller (history.state + popstate +
> per-tab URL snapshots). This one covers everything **above** it:
> *where does scene state live, how does it survive tab switches, and
> what's the rule for picking storage*.

---

## 0. The one rule

> **Pick storage by lifetime, not by who needs it.**

Phase 1 asked the wrong question ("does my page need this value?") and
got the wrong answer ("put it in `useState`"). Whenever the page
remounted — tab switch, back/forward, reload — the value died.

Phase 2 reframes around lifetime:

| Lifetime | Storage | Examples |
|---|---|---|
| Lives until you close the tab | per-tab in-memory (`useTabInstance`) | scroll position, draft text, hovered row |
| Lives until reload, but per-tab | URL (nuqs / our store-sync) | filters, date ranges, sub-tab selectors, pagination |
| Lives across reloads, all tabs | localStorage (Zustand `persist`) | user prefs, last-used filter as seed for new tabs |
| Lives forever, instantly shared | global singleton (Zustand, no persist) | currently selected metric card, dev flags |

Once you pick a row, the implementation drops out almost mechanically.

---

## 1. The three storage modules we built

### 1a. `workspaceStore` — the tab registry

[`src/stores/workspaceStore.ts`]

```ts
interface WorkspaceState {
  tabs: WorkspaceTab[];           // ordered tab list
  activeTabId: string | null;
  isReady: boolean;
  tabData: Record<string, Record<string, unknown>>;
                                  // ← per-tab in-memory bag
  // setters + tab actions...
}
```

Two responsibilities, one store:

- **Tab list** — exactly what `WorkspaceContext` used to hold in
  `useState`. Migrated wholesale.
- **`tabData[tabId]`** — generic key/value store, scoped by tab id.
  Used by `useTabInstance`. Cleared in `closeTab` so closed tabs don't
  leak memory.

Why one store and not two: the tab list and `tabData` need to be
garbage-collected together. Having them in the same module means
`closeTab` can clear the right slot without crossing a module boundary.

### 1b. `useTabInstance` — per-tab in-memory state

[`src/hooks/useTabInstance.ts`]

```ts
const [scrollY, setScrollY] = useTabInstance("scroll", 0);
const [draft,   setDraft]   = useTabInstance("draft", "");
```

Drop-in for `useState`, but the value is keyed by the active tab id.
When you switch tabs, the page component unmounts and remounts (because
`<TabContentWrapper key={activeTabId}>`); on remount, this hook reads
the same slot from `tabData[tabId]` and you get your value back.

The scope is **(tab id, key)**. Two `/web` tabs have different ids, so
they have independent values for the same key. That's the per-tab
isolation we couldn't get from React state alone.

### 1c. `webAnalyticsStore` — scene state (URL + localStorage + memory)

[`src/stores/webAnalyticsStore.ts`]

One store covers the **whole `/web` scene** — both `/web` and
`/web/web-vitals`. Three storage tiers from one file:

```ts
const useWebAnalyticsStore = create<WebAnalyticsState>()(
  persist(
    (set) => ({
      dateFrom: "-7d",
      graphsTab: "PAGE_VIEWS",
      percentile: "p90",       // ← URL-bound + localStorage seed
      tablesOrderBy: "",       // ← localStorage only
      selectedMetric: "INP",   // ← in-memory only
      // setters...
    }),
    {
      name: "sxp_web_analytics",
      partialize: (s) => ({
        dateFrom: s.dateFrom,
        graphsTab: s.graphsTab,
        percentile: s.percentile,
        tablesOrderBy: s.tablesOrderBy,
        // selectedMetric intentionally omitted → never persisted
      }),
    },
  ),
);
```

The `partialize` is the whole trick. Anything excluded from it is
in-memory only. Anything included is restored from localStorage on
load. Whether something is also URL-bound is a separate decision,
made by the URL-sync hook.

---

## 2. The mental model: store as scratch register

This is the part of Phase 2 that took the longest to get right, and
the part the original plan got wrong.

### What we initially claimed

> "Two `/web` tabs share the same `dateFrom` because the store is
> global."

### What's actually true

Two `/web` tabs **have independent `dateFrom` values**. But not because
each tab has its own store instance — they share one singleton store.
Per-tab independence comes from somewhere else entirely: each tab's
`savedQueryString` in `workspaceStore.tabs[i]`.

The store is a **scratch register that always reflects the active
tab's URL.** Switching tabs replays the new active tab's URL through
the URL→store sync, and the store updates to match.

```
                        ┌─────────────────────────────────┐
                        │  workspaceStore.tabs[]          │
                        │  ┌──────────┐  ┌──────────┐    │
                        │  │ Tab A    │  │ Tab B    │    │
                        │  │ saved=-30│  │ saved=-7d│    │
                        │  └──────────┘  └──────────┘    │
                        └────────┬─────────────┬──────────┘
                                 │             │
              switchTab(A)       │             │  switchTab(B)
                                 ▼             ▼
                        ┌─────────────────────────────────┐
                        │  router.push(active.savedURL)   │
                        └────────────────┬────────────────┘
                                         │
                                         ▼
                        ┌─────────────────────────────────┐
                        │  URL = /web?date_from=...       │
                        └────────────────┬────────────────┘
                                         │
                              URL → store sync (effect)
                                         │
                                         ▼
                        ┌─────────────────────────────────┐
                        │  webAnalyticsStore.dateFrom = ? │
                        │  ← always mirrors active tab    │
                        └─────────────────────────────────┘
```

The flow on a write is the mirror image:

```
user types in <input>
   │
   ▼
store.setDateFrom("-15d")
   │
   ▼  store → URL effect
router.replace("/web?date_from=-15d")
   │
   ▼  workspace.urlSync (existing)
active tab's savedQueryString = "date_from=-15d"
   │
   ▼  (other tabs' savedQueryString unchanged)
Tab A frozen at -30d, Tab B updated to -15d, clean.
```

This is exactly PostHog's model: `webAnalyticsLogic` is a kea
singleton, and per-tab state lives in `sceneLogic.values.tabs[i].url`.
The singleton just reflects whatever tab is currently mounted.

---

## 3. Bidirectional URL sync without infinite loops

[`src/hooks/useWebAnalyticsUrlSync.ts`]

Two effects that look symmetric:

**URL → store** (read URL, write store if different):
```ts
useEffect(() => {
  const sp = new URLSearchParams(searchParamsStr);
  const s = useWebAnalyticsStore.getState();
  const u = sp.get("date_from");
  if (u !== null && u !== s.dateFrom) s.setDateFrom(u);
  // ... other params
}, [searchParamsStr]);
```

**store → URL** (read store, write URL if different):
```ts
useEffect(() => {
  const expected = buildWebAnalyticsParams(state, { isVitalsPage, ... });
  const expectedUrl = pathname + (expected ? "?" + expected : "");
  if (expectedUrl !== window.location.pathname + window.location.search) {
    router.replace(expectedUrl, { scroll: false });
  }
}, [dateFrom, dateTo, interval, graphsTab, percentile, pathname, isVitalsPage]);
```

No loops because both effects short-circuit when values already match.
A pure URL change updates the store; the store→URL effect then sees
the URL is already correct and does nothing. A pure store change
updates the URL; the URL→store effect then sees the store is already
correct and does nothing.

Two important details:

1. **`URLSearchParams.get()` returns `null`, not `""`, when a key is
   absent.** The `null` check matters — it's the difference between
   "URL says reset to empty" and "URL doesn't mention this key, leave
   it alone." This is how the **bleed-through** behavior works (next
   section).

2. **`buildWebAnalyticsParams` starts from existing params and only
   sets/deletes the keys it owns.** No allowlist. Unknown query params
   pass through untouched. Same as PostHog's `stateToUrl`.

---

## 4. The four state categories, with an example each

PostHog's web analytics has four distinct categories. Phase 2.C
demonstrates all four in one scene store.

### Category A — Per-tab URL only (`graphsTab`)

- Each tab has its own value via `savedQueryString`.
- Switching tabs replays the URL → store updates.
- New tab opened from sidebar: tab has no `?graphs_tab=` in URL → URL
  doesn't fire setter → store keeps whatever value the previously
  active tab had → store→URL writes that value to the new tab.
- Result: "recently visited tab leaks into fresh tab" — a *feature*,
  not a bug, because users expect the new tab to inherit context.

### Category B — Per-tab URL + localStorage seed (`percentile`)

Same as A, plus `persist` middleware writes the value to localStorage
on every change. On a fresh browser session, the store rehydrates from
localStorage before any URL is read, so even brand-new tabs start with
the user's last-used percentile.

### Category C — localStorage only, never URL (`tablesOrderBy`)

In the store. In `partialize`. **Not** in `buildWebAnalyticsParams`.
Survives reloads forever, never appears in the URL, never copied when
the user shares a link. Right place for "user pref" values that
shouldn't bleed into shared URLs.

### Category D — Truly global singleton (`selectedMetric`)

In the store. **Not** in `partialize`. **Not** in
`buildWebAnalyticsParams`. Lives in memory, instantly shared across
every component that reads it. Switching the metric on one tab
*instantly* changes the metric all other tabs are looking at, because
they all read the same in-memory value.

This is what PostHog does for the web-vitals metric card (INP / LCP /
FCP / CLS): no `persistConfig`, no URL param, just one in-memory value
on the singleton logic. We do the same via `partialize` exclusion.

---

## 5. PostHog quirks we deliberately don't replicate

While reverse-engineering, we observed two PostHog behaviors that are
**side effects of their kea singleton-with-frozen-props architecture**,
not intentional design. We kept the cleaner Next.js semantics.

### Quirk: "Switching to an old tab silently overwrites it"

Reproduce in PostHog:
1. Open `/web/web-vitals`, set percentile to P75. Call this Tab 1.
2. Open another `/web/web-vitals`, set percentile to P90. Tab 2.
3. Open a third, P90 inherited (correct bleed-through). Tab 3.
4. Switch back to Tab 1. **Tab 1 now shows P90.** Its saved URL has
   been silently mutated.

The cause is `tabAwareActionToUrl`'s inactive branch:

```ts
sceneLogic.actions.setTabs(
  sceneLogic.values.tabs.map((tab) => {
    if (tab.id === logic.props.tabId) {
      return { ...tab, url: <current router URL> };
    }
    return tab;
  }),
);
```

`webAnalyticsLogic` has no `key()`, so it's a singleton. The
singleton's `props.tabId` is set on first mount and **never updates**.
So `logic.props.tabId` is always Tab 1's id, regardless of which tab
is actually inactive. Whenever any other tab fires a URL-affecting
action, this code "saves the inactive tab's URL" — and always picks
Tab 1, overwriting it with the current router URL.

**Why we don't replicate it.** Our `workspaceStore.tabs[i].savedQueryString` is only ever updated for the **currently active**
tab, by `WorkspaceContext`'s `urlSync`. There's no "save the inactive
tab" code path, so there's nothing to misfire. Each tab's saved URL is
exactly what it was when the tab was last active, full stop. Result:
Tab 1 always restores to P75 the way users would expect.

### Quirk: timing-dependent state in the same flow

In PostHog the second time you run the same experiment (after Tab 1
explicitly re-saves to P75), the overwrite doesn't happen — because
`tabAwareActionToUrl` last ran with a URL that already had `?percentile=p75`. State that depends on "which URL was the router at
when this last fired" is fragile. Our model has no equivalent timing
window: writes go through the active tab's `savedQueryString` only,
deterministically.

---

## 6. File map

```
src/
├── stores/
│   ├── workspaceStore.ts        ← tab registry + tabData (per-tab bag)
│   └── webAnalyticsStore.ts     ← scene state (4 categories above)
│
├── hooks/
│   ├── useTabInstance.ts        ← drop-in useState, scoped by tabId
│   └── useWebAnalyticsUrlSync.ts ← bidirectional URL ↔ store
│
├── context/
│   └── WorkspaceContext.tsx     ← effects host, popstate, history.state
│                                  (now reads/writes workspaceStore
│                                  instead of useState)
│
└── app/(shell)/
    ├── layout.tsx               ← <TabContentWrapper key={activeTabId}>
    ├── TabContentWrapper.tsx    ← forces remount on tab switch
    └── web/
        ├── layout.tsx           ← sub-tab nav reads STORE, not searchParams
        ├── page.tsx             ← demos the four storage categories
        └── web-vitals/page.tsx  ← shares the same store, owns ?percentile=
```

The two new modules (`webAnalyticsStore`, `useWebAnalyticsUrlSync`) are
the template for any future scene. To add `/replay` analytics state,
copy the pair, rename, change the param keys.

---

## 7. Why one store per *scene*, not per page

PostHog's web analytics has two routes (`/web` and `/web/web-vitals`)
but **one** `webAnalyticsLogic`. Both pages read and write the same
store. That's why `dateFrom` survives `/web → /web/web-vitals → /web`:
the store is mounted once at the layout level, both pages reuse it.

The naive "store per page" would create one store on `/web`, throw it
away on navigate, recreate on return — the value would die in the
round-trip. The fix isn't to add a per-page sync; it's to recognize
that `/web` and `/web/web-vitals` are **the same scene** with two
views.

In our app the URL-sync hook is mounted in `web/layout.tsx`, which
wraps both pages. The store is a module-level singleton (Zustand), so
remounting the layout doesn't recreate it. Sub-tab navigation builds
the URL from the store — never from `useSearchParams()` — so values
that aren't currently visible on the URL still come along for the
ride.

---

## 8. The sub-tab navigation rule

Don't read query state from the URL of the page you're standing on.
Read it from the store.

```ts
// ❌ WRONG — searchParams on /web/web-vitals doesn't see /web's params
const navigate = (path: string) => {
  const qs = searchParams.toString();
  router.push(qs ? `${path}?${qs}` : path);
};

// ✅ RIGHT — store always has the current scene state
const navigate = (path: string) => {
  const isVitalsTarget = path.startsWith("/web/web-vitals");
  const params = buildWebAnalyticsParams(
    useWebAnalyticsStore.getState(),
    { isVitalsPage: isVitalsTarget, existing: new URLSearchParams(window.location.search) },
  );
  if (!isVitalsTarget) params.delete("percentile");
  const qs = params.toString();
  router.push(qs ? `${path}?${qs}` : path);
};
```

The `existing` argument carries through unknown query params (no
allowlist). The explicit `params.delete("percentile")` strips the
vitals-only param when leaving the vitals route. Everything else flows
from the store.

---

## 9. What didn't change

Worth explicitly listing, because every refactor has a "did we break
the foundation" anxiety:

- The history.state ↔ popstate restoration logic in `WorkspaceContext`
  is byte-for-byte the same as Phase 1. Pushing history entries,
  catching `popstate`, replaying the snapshot, sessionStorage backup —
  all preserved.
- The `urlSync` effect that records the active tab's URL into
  `savedQueryString` is unchanged.
- The `<TabContentWrapper key={activeTabId}>` remount-on-switch
  pattern is unchanged.
- The `RightPanelContext` is unchanged. Phase 2 never touched it.
- The native `pushState` / `replaceState` patches that bypass Next.js
  router for our internal nav are unchanged.

What we replaced inside `WorkspaceContext`:
- `useState<WorkspaceTab[]>([])` → `useTabs()` selector from store
- `useState<string|null>(null)` for `activeTabId` → `useActiveTabId()`
- `useState<boolean>(false)` for `isReady` → `useIsReady()`
- `setTabs(next)` calls → `useWorkspaceStore.getState().setTabs(next)`
  (same name, same semantics, just routed through the store)

The component still owns all the side effects; the store just owns the
state.

---

## 10. Test matrix that proves the architecture

These are the scenarios that, between them, exercise every code path
that matters. Run them after any future change to Phase 2 surfaces.

| # | Action | Expected | Why |
|---|---|---|---|
| 1 | Type "A" in `useTabInstance` input on Tab 1, switch to Tab 2, switch back | "A" still there | Per-tab state survives switch |
| 2 | Two `/web` tabs, set Tab A's `dateFrom` to `-30d`, switch to Tab B | Tab B shows `-7d` (its saved value) | Per-tab URL independence |
| 3 | Set `dateFrom` on Tab A, hard reload | After reload, store rehydrates from localStorage; URL still wins on collision | localStorage seed |
| 4 | Open new `/web/web-vitals` tab from sidebar | Inherits previously active tab's `percentile` | Bleed-through (Category A) |
| 5 | Click FCP card on Tab 1, switch to Tab 2 | Tab 2 already shows FCP | Singleton (Category D) |
| 6 | `/web?date_from=-30d` → click "Web vitals" → click "Web analytics" | URL is `/web?date_from=-30d` (param survived round-trip) | Sub-tab nav reads store |
| 7 | Press browser back from a deeply navigated state | Restores tabs and URL together | history.state replay |
| 8 | `closeTab(id)` then look at Zustand devtools | `tabData[id]` is gone | GC via `clearTabInstance` |

If any of these regress, look at the dependency arrays of the two
URL-sync effects first. They're the most fragile bit.

---

## 11. Where this is going (Phase 3 preview)

Phase 2 fixed the **state** dimension. Phase 3 fixes the **identity**
dimension of tabs:

- **Custom titles** — let the user rename a tab without changing its
  URL. Stored on the `WorkspaceTab` record.
- **Scroll restoration** — currently we have the storage layer
  (`useTabInstance`) but no auto-save/restore wrapper. Phase 3 adds
  one.
- **Pinned tabs** — survive workspace reset, sit at the front of the
  bar, can't be closed without unpin.
- **Duplicate** — clone an existing tab including its URL state.
  Trivial once we have the actions in place.
- **Reorder** — drag-to-rearrange. Pure store action, no URL impact.

None of these need new storage layers. They're all on top of the four
categories above. That's the win of Phase 2 — the state foundation is
done; everything else is shape-of-data.
