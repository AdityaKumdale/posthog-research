# `WorkspaceContext.tsx` — A Deep Dive

> Notes from a junior-dev's POV. Read this top-to-bottom once. Then come
> back and use the section headings as a map.

---

## 0. What this file is for, in one paragraph

We're building a multi-tab workspace inside a Next.js App Router app —
think of Chrome tabs *inside* one Chrome tab. Each tab has its own URL,
title, and history. The user expects:

- **Open** a tab → it appears.
- **Close** it → it disappears.
- **Switch** to it → the URL bar and content reflect it.
- **Browser back/forward** → restore the exact set of tabs that existed
  at that point in time.
- **Page reload** → tabs stay the same.

The hard part is the third bullet: making the *browser's* back/forward
buttons treat our tab list as part of "the page," not just the URL.
That's what this entire file is solving.

---

## 1. The mental model — three things that must stay in sync

```
┌────────────────────┐       ┌──────────────────────┐
│ React state        │       │ window.history.state │
│ (tabs, activeTabId)│ ◄───► │ .workspace           │
└────────────────────┘       │ ─────────────────────│
        ▲                    │ (one snapshot per    │
        │                    │  history entry, kept │
        ▼                    │  by the browser)     │
┌────────────────────┐       └──────────────────────┘
│ sessionStorage     │              ▲
│ "sxp_workspace_    │              │
│  backup"           │              │
└────────────────────┘              ▼
                              ┌──────────────────────┐
                              │ The URL bar          │
                              │ /web?foo=bar         │
                              └──────────────────────┘
```

Three storage layers:
1. **React state** — `useState` for `tabs`, `activeTabId`. What renders.
2. **Browser history state** — invisible blob attached to each entry in
   the browser's session history. **The browser snapshots/restores this
   for free on back/forward.** This is our authority for navigation.
3. **sessionStorage** — backup that survives a full page reload (history
   state survives reloads too in modern browsers, but belt-and-braces).

The whole file is essentially: keep these three in sync, and let the
browser drive when the user presses back/forward.

---

## 2. Why the browser's history.state is the secret sauce

The Web has a built-in primitive:
```js
history.pushState(state, "", url)        // add a new entry
history.replaceState(state, "", url)     // overwrite current entry
window.addEventListener("popstate", e => e.state)  // fires on back/fwd
```

`state` is **any serializable object** the browser stores per history
entry. When the user presses back/forward, the browser hands you back
the exact same `state` object on the `popstate` event.

So if I do:
```js
history.pushState({ workspace: { tabs: [a, b] } }, "", "/x")
history.pushState({ workspace: { tabs: [a, b, c] } }, "", "/y")
// user presses back
// → popstate fires with e.state = { workspace: { tabs: [a, b] } }
// user presses back again
// → popstate fires with e.state.workspace = { tabs: [a] } (or whatever was there)
```

PostHog's frontend tab system uses this exact trick. **We're stealing
their idea.** No `?hs=tab_id` URL params, no state library subscribing
to URL changes — just `history.state.workspace` per entry.

The catch: Next.js App Router calls `history.pushState(null, "", url)`
internally (no state). If we let Next.js do navigation directly, every
new history entry would have `state === null` — i.e., no workspace.
**We have to inject our workspace into every entry.** That's what the
patching does.

---

## 3. File walkthrough, section by section

### 3.1 Types

```ts
type AppId = "home" | "search" | "web" | …
interface WorkspaceTab { id, appId, title, savedPath, savedQueryString }
interface WorkspaceState { tabs, activeTabId }
type HistoryPayload = Record<string, unknown> & { workspace?: WorkspaceState }
```

`HistoryPayload` is the shape of `window.history.state` *as we see it*.
The `Record<string, unknown>` part is intentional: Next.js stores its
own keys (`__NA`, `tree`, etc.) on history.state. We must **never
overwrite the whole object** — only add/replace `workspace`. Hence
`{ ...existing, workspace: ... }` everywhere.

> **Lesson:** when you're injecting into a shared bag, always spread
> the existing keys first.

### 3.2 Constants

`APP_ROUTES` and `APP_TITLES` are static maps from app id to URL/label.
`STORAGE_KEY` is the sessionStorage key. `PENDING_TTL_MS = 1500` is the
maximum time a queued workspace can sit before we drop it (more on this
in §3.6).

### 3.3 Helpers

- `log(...)` — wrapped console.log so every line is prefixed
  `[Workspace]`. Easy to filter in DevTools.
- `quickid()` — lightweight random-string generator. Tab IDs don't need
  to be UUIDs.
- `pathnameToAppId(pathname)` — given `/web/web-vitals`, return
  `"web-vitals"`. Uses **longest-prefix match** so `/web/web-vitals`
  beats `/web`. Important for nested routes.
- `buildActiveUrl(tabs, activeId)` — reconstruct a URL from a tab's
  `savedPath` + `savedQueryString`.

### 3.4 The provider — state and refs

```ts
const [tabs, setTabs] = useState<WorkspaceTab[]>([]);
const [activeTabId, setActiveTabId] = useState<string | null>(null);
const [isReady, setIsReady] = useState(false);

const tabsRef = useRef<WorkspaceTab[]>([]);
const activeTabIdRef = useRef<string | null>(null);
```

We have two parallel storage shapes:

| | `useState` | `useRef` |
|---|---|---|
| triggers re-render? | ✅ | ❌ |
| current synchronously? | ❌ (closures lock in old value) | ✅ |

Inside event handlers and effects, **state from closures is one render
behind**. Refs give us the right-now value. So we keep a ref alongside
each piece of state and update both together. This is a common pattern
for code that mixes React state with imperative APIs (browser history
in our case).

> **Lesson:** when you need synchronous access from inside imperative
> callbacks (event listeners, history-API patches, effects that fire
> before React re-renders), pair `useState` with `useRef`.

### 3.5 `pendingByUrlRef` — the message queue

```ts
const pendingByUrlRef = useRef<
  Map<string, { ws: WorkspaceState; expiresAt: number }>
>(new Map());
```

This is a queue of "workspace payloads waiting to be attached to the
next history entry for a given URL." Fundamental problem:

```
WE want to: history.pushState({ workspace: {...} }, "", "/web")
But Next.js does: router.push("/web")
   which inside calls: history.pushState(null, "", "/web")
                                          ^^^^ no state!
```

So before we call `router.push("/web")`, we put the workspace into the
queue keyed by `"/web"`. When Next.js's call hits our **patched
pushState**, the patch finds the queued entry and merges it in.

Why a TTL? If `router.push("/web")` gets *deduped* (Next.js short-
circuits same-URL pushes), our patched pushState never fires, and the
payload would sit forever, attaching itself to the wrong navigation
later. The TTL says "if nobody picks me up in 1.5s, throw me out."

> **Lesson:** any time you stash data for a downstream call you don't
> control, give it an expiry. Otherwise you'll have a memory leak with
> a side-effect that bites you in 6 months.

### 3.6 `persistToSession` — the reload backup

Trivial: dump `tabs` + `activeTabId` to `sessionStorage`. Used only in
the init path when `history.state` is empty (full reload edge cases).

### 3.7 The patches — `pushState` / `replaceState`

This is the heart of the file. We *override* the global
`history.pushState` and `history.replaceState` for the lifetime of
the provider so every history entry gets our `workspace` blob.

```ts
useEffect(() => {
  const origPush = window.history.pushState.bind(window.history);
  const origReplace = window.history.replaceState.bind(window.history);

  const merge = (incomingState, url) => {
    // 1. caller already supplied workspace? respect it
    // 2. is there a queued workspace for this URL? consume it
    // 3. otherwise, keep the existing entry's workspace, or fall back
    //    to in-memory state
  };

  window.history.pushState = function patchedPush(state, title, url) {
    return origPush(merge(state, url), title, url);
  };
  window.history.replaceState = function patchedReplace(state, title, url) {
    return origReplace(merge(state, url), title, url);
  };

  return () => {
    window.history.pushState = origPush;
    window.history.replaceState = origReplace;
  };
}, []);
```

Visualization of the merge decision:

```
            ┌────────────────────────┐
            │ patched pushState call │
            └──────────┬─────────────┘
                       │
        ┌──────────────▼──────────────┐
        │ Did the caller pass         │
        │ state.workspace explicitly? │
        └──────┬───────────────┬──────┘
               │ yes           │ no
               ▼               ▼
       use that workspace   ┌──────────────────────────┐
                            │ Is there a pending entry │
                            │ in pendingByUrlRef       │
                            │ for this URL?            │
                            └──────┬───────────────┬───┘
                                   │ yes           │ no
                                   ▼               ▼
                         consume + use it    ┌──────────────────────────┐
                                             │ Use existing entry's     │
                                             │ workspace, or in-memory  │
                                             │ tabs as last resort.     │
                                             └──────────────────────────┘
```

### 3.8 Init + popstate listener

```ts
useEffect(() => {
  const handlePopState = (e) => { ... };
  window.addEventListener("popstate", handlePopState);

  // INIT: read history.state.workspace → setTabs etc.
  // OR sessionStorage → setTabs etc.
  // OR fresh seed via commitState

  return () => window.removeEventListener("popstate", handlePopState);
}, []);
```

Three init branches:

```
            ┌─────────────────────────────────┐
            │ window.history.state.workspace? │
            └──────────────┬──────────────────┘
                           │ yes
                           ▼
                hydrate from history.state
                           │
            ───────────────┼─────────────────────────
                           │ no
                           ▼
            ┌─────────────────────────────────┐
            │ sessionStorage.getItem(BACKUP)? │
            └──────────────┬──────────────────┘
                           │ yes
                           ▼
                hydrate from sessionStorage
                + replaceState to stamp current entry
                           │
            ───────────────┼─────────────────────────
                           │ no
                           ▼
                fresh seed from current URL
                + commitState (replace=true)
```

`handlePopState` is the listener for back/forward. When `popstate`
fires, the browser is telling us "the user navigated; here's the state
that belongs to where they are now." We trust that state and call
`setTabs` / `setActiveTabId` to mirror it.

> **Why no `isMountedRef` guard?** Earlier versions had `if
> (isMountedRef.current) return` to prevent double-init. But Fast
> Refresh re-runs effects with refs preserved → guard short-circuits
> → listener never reattaches → app silently breaks until you reload.
> The init logic is now idempotent so it's safe to re-run any number
> of times.

### 3.9 `commitState` — the navigation orchestrator

Called when the user does an *intentional* action (open tab, close,
switch, sidebar). Steps:

```
commitState(nextTabs, nextActive, replace?)
       │
       ▼
1. setTabs / setActiveTabId / refs / sessionStorage — sync our state
       │
       ▼
2. compute URL from nextTabs[active].savedPath + savedQueryString
       │
       ▼
3. enqueue { ws: nextWs, expiresAt: now+1.5s } in pendingByUrlRef
       │
       ▼
4. Are we navigating to the *same* URL we're already on?
       │ yes                      │ no
       ▼                          ▼
   history.pushState(...)    router.push(url) (or replace)
   directly (BYPASS)             │
                                 ▼
                       Next.js eventually calls
                       history.pushState(null, ...)
                       which our patch intercepts +
                       merges in the pending workspace.
```

#### Why the same-URL bypass?

Next.js's router has an optimization: if you call `router.push(x)`
when the URL is already `x`, instead of pushing a real new entry, it
does a `replaceState` (no new entry). That collapses two history
entries into one — bad for us if the user opened two new tabs at
`/search` (we want each to be its own entry).

The bypass: detect that case and call native `history.pushState`
ourselves. Our patch still adds the workspace, and we get a real new
entry.

### 3.10 `urlSync` — the careful one

```ts
useEffect(() => {
  // ... two guards ...
  // ... write the active tab's appId/savedPath/savedQuery ...
  // ... replaceState to stamp it ...
}, [pathname, searchParamsStr, isReady]);
```

`urlSync` exists for one specific case: the user clicks a `<Link>` to
a sub-route inside the same app, e.g. `/web` → `/web/web-vitals`.
Next.js handles the navigation; the URL changes; but the active tab's
`savedPath` stays at `/web`. urlSync notices and updates it.

The guards exist because `urlSync` ALSO runs on back/forward — and
on back/forward the history entry is already correct (the browser
restored it). If we let urlSync write a `replaceState` here it would
**corrupt the just-restored entry**, which causes the "two tabs popping
at once on forward" bug we hunted for hours.

Guard 1 — pathname stale check:
```ts
if (window.location.pathname !== pathname) return;
```
Right after popstate, Next.js's `usePathname()` lags behind
`window.location.pathname` for one tick. Skip until they agree.

Guard 2 — history.state authoritative match:
```ts
const histWs = window.history.state?.workspace;
if (histWs?.tabs[active].savedPath === pathname) {
  // history.state is the truth; just sync in-memory and bail.
  return;
}
```
If the entry's stored workspace already says "the active tab is on
this URL," we are *inside* a popstate restore. Don't overwrite it.

Decision tree:

```
                  pathname or search changed
                            │
                            ▼
                  ┌───────────────────────┐
                  │ Is React's pathname   │
                  │ stale relative to     │
                  │ window.location?      │
                  └─────┬───────────────┬─┘
                        │ yes           │ no
                        ▼               ▼
                     skip       ┌─────────────────────────┐
                                │ Does history.state's    │
                                │ active tab already      │
                                │ match the current URL?  │
                                └─────┬───────────────┬───┘
                                      │ yes           │ no
                                      ▼               ▼
                          catch in-memory       update active tab
                          state up if needed,   + replaceState
                          do NOT write.         (sub-route nav case)
```

### 3.11 Public actions

`openTab`, `closeTab`, `switchTab`, `sidebarNavigate` — all small
functions that:

1. Compute the next `tabs` array.
2. Pick the next `activeTabId`.
3. Call `commitState(...)`.

They never touch `history.state` directly — `commitState` is the only
place that does.

---

## 4. Five complete user flows traced through the code

### 4.1 Cold load (first ever visit to /home)

```
Browser navigates to /home
  ↓
Next.js renders ShellLayout → WorkspaceProvider mounts
  ↓
useEffect (patches): patches pushState/replaceState. Logs "history APIs patched"
  ↓
useEffect (init+popstate):
  • adds popstate listener
  • reads history.state → no workspace
  • reads sessionStorage → empty (first ever visit)
  • takes "fresh seed" branch
  • creates initialTab (appId="home")
  • commitState([initialTab], initialTab.id, replace=true)
       ↓
       • setTabs / setActiveTabId / refs / sessionStorage
       • enqueue { /home → ws } in pendingByUrlRef
       • router.replace("/home")     ← (replace=true)
       ↓
       (Next.js internally calls patched replaceState)
       ↓
  patched replaceState:
       • merge() finds pending /home → consumes it
       • origReplace({ ...existing, workspace: {tabs:[home], activeTabId:home.id} })
  • setIsReady(true)
  ↓
URL bar: /home   |   history.state.workspace = { 1 tab, active=home }
```

### 4.2 Open a new search tab from /home

```
User clicks "+" button → TabBar.openTab()
  ↓
openTab(): newTab = {appId:"search"}; commitState([home, newTab], newTab.id)
  ↓
commitState:
  • state/refs updated
  • url = "/search"
  • enqueue { /search → ws[home,search] }
  • currentUrl = "/home" ≠ "/search" → router.push("/search")
  ↓
Next.js eventually: history.pushState(null, "", "/search")
  ↓
patched pushState:
  • finds queued /search → consumes
  • origPush({...existing, workspace: ws[home,search]}, "", "/search")
  ↓
Browser: new history entry. URL=/search. state.workspace=ws[home,search]
```

### 4.3 Press back from /search

```
User clicks browser back button
  ↓
Browser pops the /search entry. window.location.pathname = "/home"
window.history.state = the /home entry's saved state (workspace=[home only])
  ↓
Browser fires "popstate" event with e.state = the /home state
  ↓
Two listeners (in order):
  1. Next.js's listener: re-renders pathname="/home"
  2. Our handlePopState:
       • setTabs(ws.tabs) [home only]
       • setActiveTabId(home.id)
       • refs updated
  ↓
React re-renders. urlSync runs.
  ↓
urlSync:
  • pathname="/home" matches window.location ✓
  • histWs.activeTabId=home.id, histActive.savedPath="/home", pathname="/home" ✓ MATCH
  • If in-mem differs from histWs → catch up. Otherwise return.
  • DOES NOT WRITE replaceState. (This is the bug fix.)
```

### 4.4 Click `<Link href="/web/web-vitals">` while on /web

```
User clicks Web vitals link inside the /web layout
  ↓
Next.js calls router.push("/web/web-vitals")
  ↓
Eventually: history.pushState(null, "", "/web/web-vitals")
  ↓
patched pushState:
  • no caller workspace
  • no pending /web/web-vitals (we didn't enqueue!)
  • fall back to existing entry's workspace (savedPath="/web")
  • origPush({...existing}, "", "/web/web-vitals")
  ↓
Now the new entry has URL=/web/web-vitals but workspace says
the active tab's savedPath=/web. Inconsistent — needs urlSync.
  ↓
React re-renders with new pathname.
  ↓
urlSync:
  • pathname="/web/web-vitals" matches ✓
  • histWs.activeTab.savedPath="/web" ≠ pathname → does NOT match
  • Falls through to write branch
  • Updates active tab: appId="web-vitals", savedPath="/web/web-vitals"
  • replaceState({...existing, workspace: updatedWs})
  ↓
Now URL and history.state are consistent.
```

### 4.5 Press forward back into /web/web-vitals

```
User pressed back earlier; now presses forward
  ↓
Browser navigates to /web/web-vitals entry
  ↓
popstate fires; e.state.workspace.activeTab.savedPath="/web/web-vitals"
(saved by urlSync in step 4.4 — that's why it had to write!)
  ↓
handlePopState restores tabs/active.
  ↓
urlSync runs, finds histWs already matches → catch up if needed, return.
  ↓
No corruption, no double-popping.
```

---

## 5. The Next.js tweaks — what we did and is it safe?

Three things we patch:

| Thing | Why | Risk |
|---|---|---|
| `window.history.pushState` | Inject `state.workspace` into entries Next.js's router creates with `state=null`. | Low. We `bind()` the original first and call it through. We restore on cleanup. We only spread/add a `workspace` key — never remove Next.js's own (`__NA`, `tree`). |
| `window.history.replaceState` | Same reason. | Same risk profile. |
| Same-URL bypass: native `history.pushState` instead of `router.push` | Next.js's same-URL deduplication collapses our entries; we want real new entries. | Low. We still hit our own `pushState` patch which Next.js's router doesn't know we patched, but Next.js's *internal* listener for `popstate` correctly handles entries created via native pushState. |

**Are we fighting Next.js?** A little, but only at one specific layer.
We're not replacing the router or hijacking navigation — we're just
adding metadata to the history entries Next.js already creates. The
router still owns:
- Route matching
- RSC payloads
- Layout caching
- Prefetching
- Loading UI

We own:
- The `state` slot of each history entry
- Our own state mirroring it

The only Next.js behavior we work around is the same-URL dedup, and
even then only for the explicit "open another search tab" case. For
everything else, `router.push`/`router.replace` runs unmodified.

**What could break in a future Next.js update?**
- If Next.js stops calling `history.pushState` and switches to a
  different mechanism (unlikely; pushState is the web standard).
- If Next.js internally relies on `state === null` semantics. Currently
  it accepts arbitrary state objects (their own use it for `__NA`,
  `tree`, etc.).

In practice, this pattern is exactly what PostHog uses (a
non-Next-but-React app) and what app-routers around the web have
converged on. The library `next-usequerystate` works similarly. It's
not exotic.

---

## 6. Bugs we hit and what we learned

### Bug 1 — TypeScript "never" inside setState callback

```ts
setTabs(prev => {
  prev.findIndex(...)   // ← inferred as never
})
```

Fix: read `tabsRef.current` directly. Functional setState narrows
overzealously when there's union state.

> **Lesson:** if TypeScript narrows weirdly, swap to a ref read.

### Bug 2 — "Last 2 tabs close at once on forward"

Cause: Next.js dedup collapsed multiple `router.push("/search")` into
a single `replaceState`, destroying intermediate entries.

Fix: same-URL bypass with native `history.pushState`.

> **Lesson:** frameworks optimize for common cases. When your case is
> not common, drop down a level. Don't be afraid of the platform API.

### Bug 3 — "Two web analytics popping in at once on forward"

Cause: after popstate, urlSync ran with stale-but-not-stale-enough
state and called `replaceState`, overwriting the just-restored entry's
workspace with wrong data. Forward then walked through entries that
all looked identical.

Fix: `urlSync` checks `history.state.workspace` first. If the entry's
workspace already matches the URL, it's authoritative — only sync
in-memory state, do not write.

> **Lesson:** when two systems can both write to the same store,
> figure out which one is authoritative *for this moment* and have the
> other one defer. The browser's history.state is the authority during
> popstate.

### Bug 4 — "After Fast Refresh, popstate stops working"

Cause: an `isMountedRef.current` guard skipped re-init on Fast Refresh
runs (refs survive Fast Refresh). The popstate listener was removed by
the cleanup but never reattached.

Fix: removed the guard; made init idempotent.

> **Lesson:** dev-mode tooling has different lifetimes than production.
> Make effects re-runnable. Don't gate on "have I run before?" unless
> you really mean it across renders.

### Bug 5 — "Forward back into the app shows blank page"

Cause: not bfcache (ruled out by the absence of `pageshow` triggering).
A Turbopack dev-mode quirk in cross-document forward navigation.

Fix: not needed. Doesn't reproduce in `bun start build` / production.

> **Lesson:** before contorting your code, test in production. Some
> dev-only artifacts shouldn't drive architecture.

---

## 7. Junior-dev lessons, summarized

1. **Pair `useState` with `useRef` whenever you mix React with
   imperative APIs.** Refs give you the right-now value; state gives
   you the renders.
2. **The browser's `history.state` is free per-entry storage.** Use it
   instead of inventing URL-based state mechanisms.
3. **Cleanups must restore everything you touched.** Patches to
   globals, listeners, timers — all must be undoable.
4. **Effects should be safe to re-run.** Idempotency is cheaper than
   guards.
5. **When two systems can write to the same place, decide who's
   authoritative *and when*.** Document it. Code defensively.
6. **Pending queues need TTLs.** Anything that waits on a downstream
   call you don't control must self-clean.
7. **Test in production builds.** Dev tooling lies.
8. **Frameworks have escape hatches.** `router.push` is convenient;
   `history.pushState` is the foundation. Use the lower level when the
   higher level optimizes against you.
9. **Logs are docs.** A consistent `[Workspace]` prefix turned hours
   of guessing into seconds of grep.
10. **PostHog's frontend is open source.** Reading other people's
    solved-it-already code is faster than reinventing.

---

## 8. Quick reference — log lines

| Log | Means |
|---|---|
| `history APIs patched` | pushState/replaceState wrapped |
| `init: hydrate from history.state` | Came from a back/forward or page reload with state preserved |
| `init: hydrate from sessionStorage` | history.state empty, used reload backup |
| `init: fresh — seed from URL` | First visit ever |
| `commitState` | User action triggered a new/replaced entry |
| `enqueuePending` | Workspace queued for the URL Next.js's router is about to push |
| `pending consumed` | Patched pushState pulled the queued workspace |
| `pending expired (TTL)` | Router deduped or didn't push — payload dropped |
| `commitState: same-URL → native pushState` | Bypassed Next's same-URL dedup |
| `pushState` / `replaceState` | History updated; `hasWs` confirms our payload attached |
| `popstate` | Browser back/forward fired; tabs restored from the entry |
| `popstate fallback: re-attached in-memory workspace` | Defensive fallback |
| `urlSync: skipped (router state stale)` | Pathname lag after popstate; wait one tick |
| `urlSync: catching up to history.state` | history.state authoritative; in-memory just synced to it |
| `urlSync: updated active tab` | Sub-route nav; active tab's savedPath updated |
| `openTab` / `closeTab` / `switchTab` / `sidebarNavigate` | User actions |

---

## 9. Where to go from here

- Read the actual `WorkspaceContext.tsx` alongside this doc.
- Open DevTools, navigate around, watch the `[Workspace]` logs flow.
- Compare to PostHog's own implementation:
  https://github.com/PostHog/posthog/tree/master/frontend/src/scenes
- Try breaking it. Add a log that does `replaceState` after every
  popstate. Watch the corruption. Now you understand guard 2 in
  urlSync.

The best way to internalize all this: delete one of the guards, see
which scenario breaks, then put it back. The bugs you reproduce
yourself are the ones you'll never write.
