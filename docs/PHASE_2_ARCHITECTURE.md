# Phase 2 — Workspace state architecture

> Companion to [`WORKSPACE_CONTEXT.md`](./WORKSPACE_CONTEXT.md).
> That doc covers the multi-tab controller (history.state + popstate +
> per-tab URL snapshots). This one covers everything **above** it:
> *where does scene state live, how does it survive tab switches, and
> what's the rule for picking storage*.

> **Reading order.** §0–§11 capture Phase 2 as originally merged.
> §12–§17 ("Phase 2.D — corrections and additions") capture four
> architectural changes that came out of real testing after that
> merge: a getState race fix, a new per-tab storage tier (`tabData`
> scene scratch), a sidebar-clear-without-clobber dep change, and a
> null-sentinel pattern for "explicit vs default". The earlier
> sections are kept verbatim for the historical record; inline
> pointers ("→ updated in §X") flag where they've been superseded.

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

> → **Updated in §12, §14, §15.** This effect now uses a live
> `getState()` read instead of closure deps (race fix), drops
> `searchParamsStr` from its dependency array (sidebar fix), and
> mount-skips its first run via a ref (no-clobber-on-rehydrate).
> The actual hook also has a hydration effect and a snapshot effect
> on top of these two — see §13 for the four-effect pattern.

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

> → **Superseded by §13 and §15.** Real testing showed this
> URL-only model can't preserve a tab's *default* value across
> switches (the URL has nothing to restore from when the value
> equals the default). The fix is two-part: a new per-tab
> `tabData[id].web` scratch tier (§13) plus a null-sentinel for
> `graphsTab` so explicit clicks always reach the URL (§15).

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

> → **Superseded by §16** (now 11 scenarios covering the new race
> fix, scene scratch, sidebar clear, and null-sentinel). The
> original 8-row matrix below is kept for the historical record.

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

---

# Phase 2.D — Corrections and additions

> Four architectural changes that came out of testing after the
> original §0–§11 was merged. Each subsection cross-references the
> earlier section it corrects. If you've read the original doc, this
> is the catch-up.

---

## 12. The cross-route tab-switch race (corrects §3)

### Symptom

1. Tab 1 on `/web` with `?date_from=Asq`.
2. Tab 2 on `/web` with `?date_from=Asqq`. Switch into it.
3. From Tab 2 navigate to `/web/web-vitals` (still `Asqq`).
4. Switch back to Tab 1.
5. Tab 1 now shows **`Asqq`**. Tab 1's saved URL has been silently
   mutated.

### Root cause

The store → URL effect in §3 used closure snapshots of `dateFrom`,
`graphsTab`, etc. captured during the render they were scheduled in.
On a cross-route switch, both effects queue for the same commit phase.
The render that schedules them has the **stale** store value (the
target tab hasn't received URL→store's setter yet):

```
switchTab(Tab 1)
  → router.push("/web?date_from=Asq")
  → React re-renders, snapshot dateFrom = "Asqq"   ← stale
  → commit phase:
      1. URL → store runs → setDateFrom("Asq")     (store now Asq)
      2. store → URL runs with closure dateFrom = "Asqq"  ← stale
         → router.replace("/web?date_from=Asqq")    ← clobbers Tab 1
```

There's no infinite ping-pong because Zustand's set is synchronous and
the deps eventually stabilize, but Tab 1's saved URL ends up wrong —
permanently, since `WorkspaceContext`'s `urlSync` records the clobbered
URL into `savedQueryString`.

### Fix

Read **live** store state via `useWebAnalyticsStore.getState()`
inside the effect, instead of relying on the closure-captured deps.
The deps are still triggers; they just don't carry the payload:

```ts
useEffect(() => {
  if (!storeToUrlMountRef.current) {
    storeToUrlMountRef.current = true;
    return;                    // see §15 — also skip first run on mount
  }
  const s = useWebAnalyticsStore.getState();        // ← live, not stale
  const expected = buildWebAnalyticsParams(s, { isVitalsPage, ... });
  // ... compare and replace
}, [
  dateFrom, dateTo, interval, graphsTab, percentile,
  pathname, isVitalsPage, router,
  // searchParamsStr deliberately NOT a dep — see §14
]);
```

Effects fire in declaration order, so by the time store → URL runs,
URL → store has already called `setDateFrom`. Zustand's `set` is
synchronous; `getState()` returns the fresh value. No race, no
clobber.

---

## 13. Per-tab scene scratch — a new storage tier (corrects §4 Cat. A)

### Symptom

1. Open `/web` Tab 1. Leave on Visitors / -7d (defaults — never
   click any control).
2. Open `/web` Tab 2, click "Sessions". URL becomes
   `/web?graphs_tab=NUM_SESSION`.
3. Switch back to Tab 1.
4. Tab 1 shows **Sessions**. Its URL is `/web` (clean), but the
   singleton store now holds `graphsTab = "NUM_SESSION"` from Tab 2,
   and Tab 1 has nothing in its URL to override the singleton.

The original §4 Category A described this as a *feature* ("recently
visited tab leaks into fresh tab"). Real users found it confusing for
**existing** tabs — it's only acceptable for genuinely-new tabs.

### Cause

A fundamental constraint: **the URL cannot carry default values**
without cluttering every URL. So a tab whose state equals the default
has no per-tab memory and inherits whatever the singleton currently
holds. URL alone is not enough to give per-tab isolation across the
default surface.

### Fix: a second per-tab storage tier

Add a sidecar map in `workspaceStore`:

```ts
interface WorkspaceState {
  // ... existing ...
  tabData: Record<string, {
    [sceneKey: string]: unknown;   // generic — reused by useTabInstance
    web?: WebSceneSnapshot;        // structured — for /web scene
  }>;
  setTabSceneSnapshot: (
    tabId: string,
    sceneKey: "web",
    snapshot: WebSceneSnapshot,
  ) => void;
  clearTabSceneSnapshot: (tabId: string) => void;
}

interface WebSceneSnapshot {
  dateFrom: string;
  dateTo: string;
  interval: string;
  graphsTab: string | null;        // see §15
  percentile: string;
}
```

`tabData` is **in-memory only** — explicitly excluded from the
`partialize` allowlist. It's the structured cousin of
`useTabInstance`'s generic kv bag. Cleared in `closeTab` so closed
tabs don't leak.

### The four-effect pattern in `useWebAnalyticsUrlSync`

The hook now runs four effects in this declaration order:

```ts
// 1. HYDRATION — restores the tab's saved scene state on every
//    TabContentWrapper remount. Authoritative for per-tab memory.
const hasHydratedRef = useRef(false);
useEffect(() => {
  if (hasHydratedRef.current) return;
  hasHydratedRef.current = true;
  const scratch = useWorkspaceStore.getState().tabData[activeTabId]?.web;
  if (scratch) useWebAnalyticsStore.setState(scratch);
}, [activeTabId]);

// 2. URL → store (unchanged — null-skip when param absent).
useEffect(() => { /* ... */ }, [searchParamsStr, isVitalsPage]);

// 3. store → URL (mount-skip + getState — see §12, §14).
const storeToUrlMountRef = useRef(false);
useEffect(() => { /* ... */ }, [/* scene state, NOT searchParamsStr */]);

// 4. SNAPSHOT — persists current scene state into tabData on every
//    user change. The "memory" hydration reads back next time.
useEffect(() => {
  setTabSceneSnapshot(activeTabId, "web", {
    dateFrom, dateTo, interval, graphsTab, percentile,
  });
}, [activeTabId, dateFrom, dateTo, interval, graphsTab, percentile]);
```

### Tab-switch flow with scratch

```
Tab 1 → Tab 2 switch:
  TabContentWrapper key change → /web layout remounts
    1. hydration:  tabData[tab2].web → scene store    (authoritative)
    2. URL→store:  URL params override if present     (idempotent)
    3. store→URL:  mount-skip ref → return            (no clobber)
    4. snapshot:   scene store → tabData[tab2].web    (idempotent)
```

Tab 1's defaults survive on later switch-back because
`tabData[tab1].web` has the real values, even though Tab 1's URL was
clean. The URL is now a **shareable projection**, not the source of
truth for per-tab identity.

### Updated storage tier table (corrects §0)

| Lifetime | Storage | Examples |
|---|---|---|
| Lives until tab close, generic kv | `useTabInstance` → `tabData[id][key]` | scroll, draft text |
| Lives until tab close, scene-shaped | **`tabData[id].web` (NEW)** | per-tab `dateFrom`, `graphsTab`, etc. |
| Lives until reload, per-tab, shareable | URL | non-default scene state |
| Lives across reloads, all tabs | localStorage (`persist` allowlist) | last-used filter as seed |
| Lives forever, instantly shared | global singleton (no `persist`) | `selectedMetric` |

The new tier is the structural answer to "the URL can't carry
defaults". Anything that needs per-tab identity *and* might equal a
default goes here.

---

## 14. Sidebar click clears URL — without store→URL clobber

### Symptom

User on Tab 1 with `/web?date_from=Asq`. Clicks the "Web analytics"
sidebar link. Expected: URL clears to `/web`, choices stay (display
still shows the Asq range data; the scratch holds the value). Actual
(before fix): URL momentarily clears, then **store → URL re-dirties
it back to `/web?date_from=Asq`** within the same render cycle.

### Cause

The store → URL effect had `searchParamsStr` in its dependency array.
When `router.push("/web")` cleared the URL, `searchParamsStr` changed
from `"date_from=Asq"` to `""`, store → URL fired again:

```
sidebar click → router.push("/web")
  → searchParamsStr changes
  → store → URL re-runs:
      reads getState() → dateFrom still Asq
      expected = "/web?date_from=Asq"
      current  = "/web"
      → router.replace("/web?date_from=Asq")    ← clobber
```

### Fix

Remove `searchParamsStr` from store → URL deps. The effect should
fire on **scene store changes only**, not on URL changes. URL changes
that don't originate from a scene-store change leave the store
untouched, and so should leave the URL untouched too:

```ts
useEffect(() => {
  // ... mount-skip + getState (§12, §15) ...
}, [
  dateFrom, dateTo, interval, graphsTab, percentile,
  pathname, isVitalsPage, router,
  // searchParamsStr deliberately NOT a dep
]);
```

Plus: in `WorkspaceContext.sidebarNavigate`, when the target appId
matches the active tab's appId, explicitly route to the clean base
URL and clear the active tab's `savedQueryString`:

```ts
const sidebarNavigate = useCallback((appId: AppId) => {
  const activeTab = tabsRef.current.find(
    (t) => t.id === activeTabIdRef.current,
  );
  if (activeTab && activeTab.appId === appId) {
    // Same-scene sidebar click — clear URL, preserve display.
    const cleanRoute = APP_ROUTES[appId];
    const next = tabsRef.current.map((t) =>
      t.id === activeTab.id
        ? { ...t, savedPathname: cleanRoute, savedQueryString: "" }
        : t,
    );
    commitState(next, activeTab.id);
    return;
  }
  // ... existing different-scene branch
}, [commitState]);
```

### Resulting behavior matrix

| Action | Pathname | URL search | Display |
|---|---|---|---|
| User changes a value | unchanged | mirrors store (default-stripped) | follows store |
| Switch to different tab | mirrors target | mirrors target | mirrors target's scratch |
| Sidebar click on same scene | unchanged | cleared | unchanged (store + scratch stay) |
| Browser back/forward | what history says | what history says | follows URL→store |
| Hard reload | what URL says | what URL says | URL → store; persist seed on absent |

---

## 15. Null-sentinel for "explicit vs default" (corrects §4 Cat. A)

### Symptom

User clicks **Visitors** (the value that happens to equal the default).
Expected: URL becomes `/web?graphs_tab=PAGE_VIEWS` so the choice
survives refresh, share, and any singleton-bleed source. Actual
(before fix): URL stays `/web` because the default-strip rule treats
`"PAGE_VIEWS"` as the default and omits it.

### Cause

`graphsTab: string` with a literal default of `"PAGE_VIEWS"` conflates
two distinct states:

- "user has never touched this control" → URL should omit the param
- "user explicitly chose Visitors" → URL must include `graphs_tab=PAGE_VIEWS`

You can't tell them apart with a single non-nullable string.

### Fix

Make `graphsTab` nullable. `null` means "never touched"; any string
means "explicit". Display falls back via a selector. This is exactly
PostHog's `_graphsTab: null as string | null` pattern:

```ts
// Store
interface WebAnalyticsState {
  graphsTab: string | null;            // ← was `string`
  setGraphsTab: (v: string) => void;   // callers always pass non-null
}
graphsTab: null,                       // default

// Producer (buildWebAnalyticsParams)
if (state.graphsTab !== null) {
  params.set("graphs_tab", state.graphsTab);  // any non-null writes
} else {
  params.delete("graphs_tab");
}

// Consumers — always resolve via selector
const graphsTab = useWebAnalyticsStore((s) => s.graphsTab ?? "PAGE_VIEWS");
// or, factored:
export const useResolvedGraphsTab = () =>
  useWebAnalyticsStore((s) => s.graphsTab ?? "PAGE_VIEWS");
```

Apply the same pattern to any future state where "user choice equal
to the shipped default" must be distinguishable from "never set". For
`dateFrom`/`dateTo`/`interval`, the existing default-strip on the
group as a whole (`dateFilterDiverges`) is fine because users always
diverge from the default when they explicitly set a date — the
sentinel is implicit in the group.

### Mount-skip ref companion (§3 update)

The store → URL effect also gets a `useRef` mount-skip so it does
**not** fire on the very first render after a tab mount:

```ts
const storeToUrlMountRef = useRef(false);
useEffect(() => {
  if (!storeToUrlMountRef.current) {
    storeToUrlMountRef.current = true;
    return;     // mount = "rehydrate quietly", not "write URL"
  }
  // ... store → URL body ...
}, [/* scene state */]);
```

Without it, freshly hydrated state on a new tab would write to the
URL on mount and clutter it. PostHog avoids this implicitly because
their `stateToUrl` is action-mapped (only fires on explicit setters).
Our React-effect equivalent is the ref. The `hasHydratedRef` from §13
is the matching "skip first run" guard for the hydration effect.
Both are necessary; neither alone is sufficient.

---

## 16. Updated test matrix (replaces §10)

Run on a fresh browser profile. Each scenario probes a specific
invariant; if any regress, the section linked in *Why* is the place
to look.

| # | Action | Expected | Why |
|---|---|---|---|
| 1 | Open `/web` (fresh) | URL `/web`, display defaults | clean URL on open |
| 2 | Open second `/web` tab | URL `/web` | clean URL on new tab |
| 3 | Tab 1 = Visitors (default, never clicked), Tab 2 = Sessions, switch back to Tab 1 | Tab 1 shows Visitors, URL `/web` | per-tab scene scratch (§13) |
| 4 | Tab 1 sets `?date_from=Asq`, Tab 2 sets `?date_from=Asqq`, switch back | Tab 1 shows Asq | per-tab URL + scratch agree |
| 5 | Tab 1 (Asq) → Tab 2 (Asqq) → Tab 2 vitals → Tab 1 | Tab 1 shows Asq | cross-route race fix (§12) |
| 6 | On Tab 1 with `?date_from=Asq`, click sidebar "Web analytics" | URL `/web`, display Asq | sidebar clear (§14) |
| 7 | After 6, switch to Tab 2, switch back to Tab 1 | URL `/web`, display Asq | scratch survives URL clear |
| 8 | Open new vitals tab from sidebar after a P75 vitals tab | New tab shows P75 | percentile bleed (Cat. B) |
| 9 | Click FCP on one vitals tab | Other vitals tab shows FCP | singleton (Cat. D) |
| 10 | On a fresh tab click "Visitors" | URL `/web?graphs_tab=PAGE_VIEWS` | null-sentinel (§15) |
| 11 | Close a tab and inspect Zustand devtools | `tabData[id]` is gone | GC via `clearTabSceneSnapshot` |

Scenarios 1, 2, 3, 6, 7, 10 are the new ones; 4, 5, 8, 9, 11 carry
over from the original §10 (renumbered).

---

## 17. File map (updates §6)

```
src/
├── stores/
│   ├── workspaceStore.ts         ← + tabData[id].web (NEW tier)
│   │                               + setTabSceneSnapshot
│   │                               + clearTabSceneSnapshot
│   │                               + closeTab clears scratch
│   │                               + sidebarNavigate same-scene branch
│   │                                 (clears savedQueryString)
│   └── webAnalyticsStore.ts      ← graphsTab nullable (§15)
│                                   buildWebAnalyticsParams default-strip
│                                   keeps date group, null-test for graphs_tab
│
├── hooks/
│   ├── useTabInstance.ts         ← unchanged
│   └── useWebAnalyticsUrlSync.ts ← FOUR effects:
│                                   1. hydrate from tabData[id].web
│                                   2. URL → store (null-skip)
│                                   3. store → URL (mount-skip + getState,
│                                      no searchParamsStr dep)
│                                   4. snapshot store → tabData[id].web
│
├── context/
│   └── WorkspaceContext.tsx      ← sidebarNavigate same-scene branch
│                                   closeTab → clearTabSceneSnapshot
└── ...
```

Five storage tiers (one new — per-tab scene scratch). Four effects in
the URL-sync hook (two of the original two, plus hydrate and
snapshot). Everything else from §6–§9 is still accurate.

The scene-store template is now: nullable null-sentinel for any
"explicit vs default" field, default-stripped producer, four-effect
hook with hydrate/snapshot bookends, and a `WebSceneSnapshot` shape
in `workspaceStore.tabData[id]`. Anything in `/replay` or `/sql`
that wants the same per-tab semantics copies the pair.
