# PostHog frontend — exploration notes

A growing set of topic-scoped notes about the PostHog/posthog frontend,
compiled from the repo wiki and targeted reads of the real source.

Each topic is a standalone markdown file here **and** a separate Devin
knowledge note pinned to `PostHog/posthog`, so the right note surfaces
when the conversation is about that topic.

## Topics

| # | File | Knowledge-note trigger (summary) |
| --- | --- | --- |
| 01 | [`01-overview.md`](./01-overview.md) | High-level architecture, folder layout, boot path, tooling, running locally, testing, learning path |
| 02 | [`02-kea-endpointslogic.md`](./02-kea-endpointslogic.md) | Line-by-line deep-dive of `products/endpoints/frontend/endpointsLogic.tsx` — imports, builders, loaders, reducers, selectors, url sync, component consumption, exercises |
| 03 | [`03-tab-system.md`](./03-tab-system.md) | How PostHog's multi-tab workspace works without `?hs=`/query-string IDs — `sceneLogic.tabs`, `getRouterState` → `history.state`, `tabAwareUrlToAction` / `tabAwareActionToUrl`, sessionStorage/localStorage layers, recommendations for adapting to Next.js App Router |

## Conventions for future topics

- One `.md` file per topic, numbered `NN-slug.md` in the order explored.
- One Devin knowledge note per topic, pinned to `PostHog/posthog`, with a
  specific `trigger` so only the relevant note activates.
- Cross-link between files instead of duplicating content.
- Keep examples grounded in real source; link to the file on GitHub `main`.
