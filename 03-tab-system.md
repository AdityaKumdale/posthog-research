# PostHog frontend · How the multi-tab workspace works (no `?hs=` needed)

> Companion to `01-overview.md` and `02-kea-endpointslogic.md`.
> This note answers: **how does PostHog let you have multiple tabs of the
> same scene with different state, and keep the browser URL clean?**
>
> Short answer: it stuffs the entire tab list into `window.history.state`
> using the History API, not into a query parameter. The visible URL only
> shows the *active* tab. Inactive-tab URLs are kept inside `sceneLogic.tabs`.

---

## TL;DR — the four moving parts

| Concern | PostHog uses | Where |
| --- | --- | --- |
| Per-tab state in memory | `sceneLogic.tabs: SceneTab[]` (Kea reducer) | `frontend/src/scenes/sceneLogic.tsx` |
| Visible URL | `kea-router` only ever reflects the **active** tab | active-tab pathname/search/hash |
| Back / Forward across tab arrangements | `window.history.state.tabs` (History API) | `getRouterState` in `frontend/src/initKea.ts` |
| Reload / new browser tab survival | `sessionStorage["scene-tab-state-<teamId>"]` | `persistSessionTabs` in `sceneLogic.tsx` |
| Pinned tabs across sessions | `localStorage["pinned-tab-state-<teamId>"]` | `persistPinnedTabs` in `sceneLogic.tsx` |
| Per-tab URL bindings inside Kea logics | `tabAwareUrlToAction` / `tabAwareActionToUrl` | `frontend/src/lib/logic/scenes/` |

There is no `?hs=` parameter, no shadow ID in the URL, no
`?tab_id=` — the URL is just `/web/web-vitals` (or whatever the active
tab is currently on). If you open Web Vitals in two tabs at once with
different filters, both tabs share the same URL pathname; their state
diverges *inside* `sceneLogic.tabs` and inside per-tab keyed Kea logics.

---

## 1. The data structure: `SceneTab`

From [`frontend/src/scenes/sceneTypes.ts`](https://github.com/PostHog/posthog/blob/main/frontend/src/scenes/sceneTypes.ts):

```ts
export interface SceneTab {
    id: string
    pathname: string         // e.g. "/project/1/web/web-vitals"
    search: string           // e.g. "?date_from=-7d&filter_test_accounts=true"
    hash: string             // e.g. "#panel=foo"
    title: string
    active: boolean          // exactly one tab is active
    customTitle?: string
    iconType: FileSystemIconType | 'loading' | 'blank'
    pinned?: boolean
    badge?: boolean
    sceneId?: string
    sceneKey?: string
    sceneParams?: SceneParams
}
```

Every tab carries its **own** `pathname/search/hash`. So two tabs both on
"web vitals" are simply two `SceneTab` objects whose `pathname` happens to
match — but their `search`/`hash` differ, and the keyed Kea logics they
mount have different `tabId`s, so their reducers/loaders are independent.

---

## 2. The single source of truth: `sceneLogic.tabs`

`sceneLogic` is a Kea logic with a `tabs` reducer. It contains
`newTab`, `removeTab`, `activateTab`, `clickOnTab`, `duplicateTab`,
`setTabs`, etc. The active tab is selected via:

```ts
activeTab: [(s) => [s.tabs], (tabs) => tabs.find((t) => t.active) || tabs[0] || null],
activeTabId: [(s) => [s.activeTab], (t) => t?.id ?? null],
```

The browser URL bar follows the **active** tab. Inactive tabs' URLs live
only in `sceneLogic.tabs[i].{pathname,search,hash}`.

### Two places where the URL → tab sync happens

[`frontend/src/scenes/sceneLogic.tsx`](https://github.com/PostHog/posthog/blob/main/frontend/src/scenes/sceneLogic.tsx)
listens to `kea-router`'s `locationChanged` action (lines 1052-1093):

```ts
locationChanged: ({ pathname, search, hash, routerState, method }) => {
    pathname = addProjectIdIfMissing(pathname)

    // (A) Browser Back/Forward: routerState carries the saved tab list.
    if (routerState?.tabs && method === 'POP') {
        actions.setTabs(routerState.tabs)
        return
    }

    // (B) Programmatic push or first load: just sync the active tab's URL.
    const activeTabIndex = values.tabs.findIndex((tab) => tab.active)
    if (activeTabIndex !== -1) {
        actions.setTabs(
            values.tabs.map((tab, i) =>
                i === activeTabIndex ? { ...tab, pathname, search, hash } : tab
            )
        )
    } else {
        // No tabs yet → seed one from the URL
        actions.setTabs([{ id: generateTabId(), active: true, pathname, search, hash, ... }])
    }
    persistTabs(values.tabs, values.homepage)
}
```

Notice path (A): **on a `POP` (back/forward) we trust `routerState.tabs`
and replace the entire tab array.** This is what makes the back button
restore the workspace exactly. It works *because of* part 3 ↓.

---

## 3. The trick that hides the multi-tab state from the URL: `getRouterState`

`kea-router` accepts a `getRouterState()` callback. Whatever it returns
is passed as the `state` argument to `history.pushState(state, '', url)`
on every navigation. From
[`frontend/src/initKea.ts`](https://github.com/PostHog/posthog/blob/main/frontend/src/initKea.ts):

```ts
routerPlugin({
    /* … */
    getRouterState: () => {
        const logic = sceneLogic.findMounted()
        if (logic) {
            const tabs = getTabsSnapshotForHistory(logic.values.tabs)
            return { tabs: structuredClone(tabs) }   // → goes into history.state
        }
        return undefined
    },
}),
```

So every time the URL changes, `kea-router` calls
`window.history.pushState({ tabs: [...] }, '', '/web/web-vitals?...')`.

The browser's history stack now contains, per entry, the full tab array
that was open at that moment. Click Back → `popstate` fires →
`kea-router` reads `event.state` and dispatches `locationChanged({ method: 'POP', routerState: { tabs } })` →
`sceneLogic` swaps its tab list to that snapshot.

**The URL stays clean** (`/web/web-vitals`) because the workspace is in
`history.state`, not in the search string.

> History.state is **per browser tab** and gets lost on full page reload.
> That's exactly why PostHog *also* writes to sessionStorage, see §4.

---

## 4. Surviving reload: sessionStorage (a mirror, not a key)

`sceneLogic.tsx` defines:

```ts
const persistSessionTabs = (tabs: SceneTab[]): void => {
    sessionStorage.setItem(
        getStorageKey(TAB_STATE_KEY),                     // single fixed key, e.g. "scene-tab-state-1"
        JSON.stringify(tabs.map((t) => tabToPersistableSnapshot(t)))
    )
}
```

It is called every time the tab list changes (`persistTabs(...)` is
sprinkled throughout the listeners). On boot, the app reads this key
and seeds `sceneLogic.tabs`. The key is **fixed per browser tab**
(sessionStorage isolates per-tab automatically) and team-scoped — there
is no per-snapshot ID in the URL pointing into it.

So:
- **history.state.tabs** is the master record for back/forward inside
  the same browser tab.
- **sessionStorage[scene-tab-state-…]** is the master record for
  reloading inside the same browser tab.
- **localStorage[pinned-tab-state-…]** is the master record for *pinned*
  tabs across sessions and across browser tabs.

None of these are referenced by an ID in the URL.

---

## 5. Per-tab URL sync inside Kea logics

OK, but how does `endpointsLogic`'s URL handler still get URL params for
**its** tab when the browser URL only shows the active tab?

The two helpers in `frontend/src/lib/logic/scenes/`:

### `tabAwareUrlToAction.ts` — only fire when this tab is active

```ts
export const tabAwareUrlToAction = <L>(input) => (logic) => {
    const finalInput = typeof input === 'function' ? input(logic) : input
    const newPayload = Object.fromEntries(
        Object.entries(finalInput).map(([k, v]) => [
            k,
            (params, searchParams, hashParams, payload, previousLocation) => {
                if (!sceneLogic.isMounted()) return v(params, searchParams, ...)
                if (sceneLogic.values.activeTabId === logic.props.tabId) {
                    return v(params, searchParams, hashParams, payload, previousLocation)
                }
                // else: the URL change is for some OTHER tab → ignore.
            },
        ])
    )
    urlToAction(newPayload)(logic)
}
```

So a logic instance with `tabId='A'` will only react to URL changes when
tab `A` is the active tab. The same logic with `tabId='B'` ignores the
same URL change. That's how two web-vitals tabs can have independent
filters reading from `?date_from=…`.

### `tabAwareActionToUrl.ts` — write to URL only if active, else stash

```ts
if (sceneLogic.values.activeTabId === logic.props.tabId) {
    const response = v(payload)            // normal: push to browser URL
    trackUrlChange(response, logic.pathString, k)
    return response
}
// We're inactive: don't touch the URL, just update OUR tab's url field.
sceneLogic.actions.setTabs(
    sceneLogic.values.tabs.map((tab) =>
        tab.id === logic.props.tabId
            ? { ...tab, ...combineUrl(pathname, search, hash) }
            : tab
    )
)
return undefined
```

The inactive tab updates its own `pathname/search/hash` inside
`sceneLogic.tabs`. When the user later clicks that tab, `clickOnTab`
dispatches `router.actions.push(tab.pathname, tab.search, tab.hash)`,
the URL bar updates to that tab's URL, that tab becomes active, and its
`tabAwareUrlToAction` handlers fire to restore filters/state.

---

## 6. Putting it all together — life of one tab switch

```
You have 3 tabs:
  - tab A: /home               (active)
  - tab B: /web/web-vitals?date_from=-7d
  - tab C: /web?date_from=-30d

You click tab B.

1. clickOnTab(B) listener
     ↓
2. setTabs([... A.active=false, B.active=true ...])   ← in-memory
     ↓
3. router.actions.push(B.pathname, B.search, B.hash)
     ↓
4. kea-router calls window.history.pushState(
       getRouterState(),                    ← { tabs: [A,B,C with current state] }
       '',
       '/web/web-vitals?date_from=-7d'
   )
     ↓
5. locationChanged dispatches; method !== 'POP', so we just sync
     the active tab's URL fields (already correct).
     ↓
6. webVitalsLogic({ tabId: B.id }) was already mounted; its
     tabAwareUrlToAction now runs because activeTabId === B.id.
     It reads `?date_from=-7d` and refreshes filters.
     ↓
7. persistTabs(...) writes the latest tab list to sessionStorage.

Browser URL: /web/web-vitals?date_from=-7d   ← clean, no hs
History.state: { tabs: [A,B,C] }             ← invisible, but there
```

If you now hit **Back**:

```
1. Browser pops → popstate event
2. kea-router reads event.state = { tabs: [A,B,C with previous URLs] }
   and dispatches locationChanged({ method: 'POP', routerState: { tabs } })
3. sceneLogic short-circuits to: setTabs(routerState.tabs)
   → A is active again, B/C stay in place.
4. URL bar: /home (whatever it was on the popped entry).
```

---

## 7. Answering your specific questions

### Q1. *"I didn't want the URL to track tabs. PostHog uses `?hs=`, right?"*

**No, it does not.** PostHog does **not** put any tab-list pointer in the
URL. The URL only ever reflects the active tab's path. The multi-tab
arrangement lives in:

- `window.history.state.tabs` (for back/forward inside the browser tab)
- `sessionStorage["scene-tab-state-<teamId>"]` (for reload inside the browser tab)
- `localStorage["pinned-tab-state-<teamId>"]` (for pinned tabs across sessions)

The "AI bot" you talked to was wrong about that.

### Q2. *"If we have `?hs=` why also use sessionStorage? Isn't one of them redundant?"*

In your current design they really *are* somewhat redundant — `?hs` is a
key, sessionStorage is the value. You could collapse them into a single
mechanism. Two cleaner options:

- **Like PostHog**: drop `?hs` from the URL. Use `history.state` for
  back/forward, sessionStorage as reload backup.
- **Embed everything in the URL**: the value (compact JSON in the query)
  rather than a key into storage. This makes URLs shareable but ugly.

Don't keep both an opaque key in the URL **and** a separate storage —
that's the worst of both: ugly URL, and any URL whose key isn't in the
current sessionStorage will silently dead-end (which is exactly the bug
you're hitting).

### Q3. *"My web-vitals tab gets clobbered into a plain web tab when I navigate."*

Your `useEffect` (the `if (!hs)` branch) treats *any* missing-`hs` URL as
"first load" → it nukes the entire tab array and seeds a single tab from
`pathname`. So every navigation that doesn't carry `hs` (e.g. `<Link>`
components, browser back, manual address bar edit) destroys your
workspace.

The other AI's diagnosis is correct on this point. Its fix — "redirect
back to the latest known `hs` URL when `hs` is missing" — works, but
fights the framework. You'd be intercepting every navigation just to
re-add a parameter that isn't load-bearing. Better: stop using `hs`
altogether and follow PostHog's design.

### Q4. *Verify the random AI's answer*

Specific claims and my verdict:

| Claim | Verdict |
| --- | --- |
| "`?hs=` is the only way to keep multi-tab layout separate from the URL path" | **False.** PostHog uses `history.state` for the same purpose with no URL pollution. |
| "Without sessionStorage you'd lose tabs on reload" | **True.** sessionStorage handles reload. (history.state is wiped on full reload.) |
| "PostHog consciously shows `?hs=` in the URL" | **False.** Look at `getRouterState` in `initKea.ts` and `locationChanged` in `sceneLogic.tsx`. There is no `?hs=` anywhere in PostHog's URLs. |
| The web-vitals "tab clobbering" diagnosis | **True.** That fallback path is the bug. |
| The "redirect on missing hs" patch | **Works**, but it's a band-aid. The right fix removes `hs` entirely. |
| "Use the History API state instead — but you'll lose it on reload, can't bookmark, no shared links" | **Partially true.** You lose history.state on reload (use sessionStorage as PostHog does). Bookmarks/shared links restoring the *exact* tab list is a feature you almost certainly don't want anyway — bookmarks should restore one *page*, not someone else's whole workspace. |

---

## 8. Concrete recommendation for your Next.js App Router app

**Drop `?hs=` and the sessionStorage-keyed-by-hs scheme.** Replace with:

1. **Keep your in-memory tab list in a context (or Zustand/Jotai/Kea/whatever).**
2. **On every tab switch / open / close / URL change, write the tab list
   to `window.history.state`** alongside the navigation:
   ```ts
   const routerPush = (url: string, tabs: WorkspaceTab[]) => {
       // Next.js's router.push doesn't accept a state arg, so do this in two steps:
       router.push(url)                                   // updates URL
       // After the push, replace the new entry's state with our tabs:
       window.history.replaceState(
           { ...window.history.state, tabs },
           '',
           window.location.href
       )
   }
   ```
   In Next.js App Router, `router.push` calls `history.pushState` under
   the hood; you can immediately follow with `replaceState` to attach
   your `tabs` payload to that history entry. (Recent Next versions also
   accept a `state` option on `router.push`, check the version you're on.)
3. **Listen for `popstate` to handle back/forward**:
   ```ts
   useEffect(() => {
       const onPop = (e: PopStateEvent) => {
           if (e.state?.tabs) setTabs(e.state.tabs)
       }
       window.addEventListener('popstate', onPop)
       return () => window.removeEventListener('popstate', onPop)
   }, [])
   ```
4. **Mirror tabs to sessionStorage** under a single fixed key (no
   per-snapshot id) on every change. On boot, seed from sessionStorage
   if `history.state.tabs` is empty.
5. **Per-tab state for same-route tabs**: keep two parallel `web-vitals`
   tabs by storing each tab's saved query string inside the tab object
   (you're already doing this with `savedQueryString`). When activating
   a tab, push `${tab.appId-route}?${tab.savedQueryString}`. Any
   feature-level state inside the page should be keyed by `tab.id` (the
   PostHog equivalent of your `props.tabId` + `key()` pattern).
6. **Delete the `if (!hs)` fallback that nukes the workspace.** Without
   `hs`, your app will:
   - try `history.state.tabs` first;
   - then `sessionStorage["sxp_tabs"]`;
   - then, only if both are empty, seed a single tab from `pathname`.

That kills the web-vitals bug as a side effect — a manually typed
`/web` URL just becomes the active tab's new path; the other tabs are
untouched because nothing in the new flow ever rebuilds the tab array
from a single pathname.

### About `"use client"` in `WorkspaceContext.tsx`

You're right to mark it `"use client"`. It uses `useRouter`,
`usePathname`, `useSearchParams`, `useState`, `useEffect`,
`sessionStorage`, `window.history` — all client-only APIs. Server
components can't run any of this. The fact that you needed
`"use client"` is correct, not a sign of doing something wrong.

If you want server-rendered shells *with* client tab logic, the standard
App Router pattern is:
- Server `(shell)/layout.tsx` for fonts, providers that don't need
  client APIs.
- Inside it, mount a `<WorkspaceContextProvider>` that is `"use client"`.
- Server-rendered `page.tsx` files for each route's content.
That matches your current architecture.

---

## 9. Files to read in PostHog for reference

- [`frontend/src/scenes/sceneTypes.ts`](https://github.com/PostHog/posthog/blob/main/frontend/src/scenes/sceneTypes.ts) — `SceneTab`, `SceneParams`, scene config types.
- [`frontend/src/scenes/sceneLogic.tsx`](https://github.com/PostHog/posthog/blob/main/frontend/src/scenes/sceneLogic.tsx) — the brain. Tab reducer, `locationChanged`, `clickOnTab`, persistence, pinned-tab logic. ~1700 lines.
- [`frontend/src/initKea.ts`](https://github.com/PostHog/posthog/blob/main/frontend/src/initKea.ts) — `getRouterState`. The 5 lines that hide tabs in `history.state`.
- [`frontend/src/lib/logic/scenes/tabAwareUrlToAction.ts`](https://github.com/PostHog/posthog/blob/main/frontend/src/lib/logic/scenes/tabAwareUrlToAction.ts) — gates URL→action by active tab.
- [`frontend/src/lib/logic/scenes/tabAwareActionToUrl.ts`](https://github.com/PostHog/posthog/blob/main/frontend/src/lib/logic/scenes/tabAwareActionToUrl.ts) — routes inactive-tab URL writes into `setTabs` instead of the browser URL.
- [`frontend/src/lib/logic/scenes/tabAwareScene.ts`](https://github.com/PostHog/posthog/blob/main/frontend/src/lib/logic/scenes/tabAwareScene.ts) — 13-line builder that auto-keys logics by `props.tabId`.
- [`frontend/src/lib/logic/scenes/tabSceneUtils.ts`](https://github.com/PostHog/posthog/blob/main/frontend/src/lib/logic/scenes/tabSceneUtils.ts) — `getTabSceneParams`, `updateTabUrl` for inactive-tab URL writes.

The interesting comparison: PostHog's tab system is built on top of Kea
+ kea-router (which is what enables the `getRouterState` hook). In a
Next.js App Router app you don't have that hook — but you have direct
access to `window.history.pushState/replaceState` and `popstate`, which
is what kea-router uses under the hood. The pattern translates
1-to-1.
