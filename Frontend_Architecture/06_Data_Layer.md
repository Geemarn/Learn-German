# 06 — Data Layer

> **The question this answers:** how does data get from the database to the pixel, and how fast does
> it feel?

Two things decide perceived speed: the **number of sequential round trips** and whether the UI
**waits** for them. Almost everything below is an attack on one of those two.

---

## Transport options

### REST

| Advantages | Trade-offs |
| :--- | :--- |
| Universally understood; cacheable by HTTP semantics | Over-fetching (you get the whole object) |
| Trivial to debug with curl and browser devtools | Under-fetching → waterfalls across endpoints |
| Works with CDNs, proxies, and every tool | Endpoint sprawl as screens diverge |
| No extra runtime on the client | Types are a promise, not a guarantee |

**Use when:** simple resource-shaped data, public APIs, or you want HTTP caching to do the heavy
lifting. Pair with OpenAPI and a **generated client** so types are real, not hand-written.

### GraphQL

| Advantages | Trade-offs |
| :--- | :--- |
| One request for exactly the fields a screen needs | Server complexity: N+1 queries, cost limiting, depth limits |
| Schema is a real, introspectable contract | HTTP caching mostly lost (POST to one URL) |
| Many clients can diverge without new endpoints | Heavier client (normalised cache) |
| Fields deprecate gracefully | A malicious or careless query can melt the database |

**Use when:** several clients need different slices of the same data, or screens aggregate many
entities. **Don't use it** for one web client over one service — you'd take on schema ops for
nothing.

### tRPC (and RPC generally)

| Advantages | Trade-offs |
| :--- | :--- |
| End-to-end types with **no codegen step** — instant | TypeScript-only, monorepo-only, same-repo coupling |
| Rename a field on the server → client fails to compile | Not a public API; third parties can't consume it |
| Minimal boilerplate; fastest development loop | Ties frontend and backend release cycles |

**Use when:** a TypeScript monorepo owned by one team. The DX is unmatched; the cost is that your
API isn't a product.

### Server Components / server functions

Skip the API layer entirely: the component queries the database on the server.

| Advantages | Trade-offs |
| :--- | :--- |
| No client/server serialisation layer to maintain | Framework lock-in |
| No over/under-fetching — you write the exact query | Only works for that framework's clients |
| Zero client-side data-fetching JavaScript | Mobile apps still need a real API |

**Use when:** a full-stack app in one framework with no other consumers.

---

## Comparison

| | REST | GraphQL | tRPC | Server Components |
| :--- | :--- | :--- | :--- | :--- |
| Type safety | via codegen | via codegen | native | native |
| Over-fetching | likely | no | no | no |
| HTTP caching | excellent | poor | poor | route-level |
| Server complexity | low | high | low | medium |
| Public API | yes | yes | no | no |
| Setup cost | low | high | lowest | medium |

---

## Killing waterfalls — the real performance work

A waterfall is when request B cannot start until A finishes. Three requests at 200 ms each,
sequenced, is 600 ms of nothing.

```jsx
// BAD: each child fetches after its parent renders → 3 sequential trips
<Page>          {/* fetches user */}
  <Orders />    {/* fetches orders, only after user resolves */}
  <Invoices />  {/* fetches invoices, only after orders render */}
</Page>
```

Fixes, in order of preference:

1. **Aggregate on the server** — one BFF endpoint or one GraphQL query per screen.
2. **Start requests in parallel** as high up as possible:
   ```js
   const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);
   ```
3. **Prefetch on intent** — fetch on link hover or focus, before the click.
4. **Stream** — render the shell immediately, let slow sections arrive later (`Suspense`).

**Diagnostic:** open the network panel and look at the waterfall shape. Staircase = you have this
problem. Flat left edge = you don't.

---

## Caching: the four decisions

Whatever library you use, you are answering these four questions:

| Decision | Meaning | Typical answer |
| :--- | :--- | :--- |
| **Key** | What identifies this data? | `['orders', { status, page }]` — every input in the key |
| **Staleness** | How long is a cached copy acceptable? | seconds for live data, minutes for catalogues |
| **Invalidation** | What makes it wrong? | mutations on the same entity |
| **Scope** | Per user, or shared? | anything personal must never hit a shared cache |

```js
// after a mutation, tell the cache what it no longer knows
await api.updateOrder(id, patch);
queryClient.invalidateQueries({ queryKey: ['orders'] });
```

**The most common cache bug:** a key that omits an input. If `['orders']` ignores the `status`
filter, switching filters shows the previous list. If in doubt, put it in the key.

**Layers of cache**, from cheapest to most expensive to get wrong:

```
CDN (shared, public)  →  server/BFF cache  →  client query cache  →  component state
```

Push caching outward (toward the CDN) for public data; keep personalised data in the client cache
only.

---

## Optimistic updates

Apply the change locally, send the request, roll back if it fails. This is what makes an app feel
instant.

```js
useMutation({
  mutationFn: toggleLike,
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: ['post', id] });
    const previous = queryClient.getQueryData(['post', id]);
    queryClient.setQueryData(['post', id], (p) => ({ ...p, liked: !p.liked }));
    return { previous };                              // rollback data
  },
  onError: (_e, id, ctx) => queryClient.setQueryData(['post', id], ctx.previous),
  onSettled: (_d, _e, id) => queryClient.invalidateQueries({ queryKey: ['post', id] }),
});
```

| Advantages | Trade-offs |
| :--- | :--- |
| UI responds in 0 ms — feels native | You must implement rollback, and it must be correct |
| Hides network latency on mobile | Briefly lies to the user |
| Fewer spinners | Server-computed values can't be predicted locally |

**Use for:** likes, toggles, reordering, checkbox lists, adding to a cart — reversible actions whose
result you can predict.

**Never for:** payments, irreversible deletes, or anything where the server decides the outcome
(inventory allocation, seat booking). Showing "purchased" and then retracting it is worse than a
spinner.

---

## Real-time

| Approach | Advantages | Trade-offs | Use for |
| :--- | :--- | :--- | :--- |
| **Polling** | Trivial; works everywhere; no state | Wasted requests; latency = interval | Dashboards, job status |
| **SSE** | One-way push over plain HTTP; auto-reconnect | Server→client only; connection limits on HTTP/1.1 | Notifications, feeds, AI token streams |
| **WebSocket** | Full duplex, lowest latency | Stateful servers, reconnection and auth complexity | Chat, collaboration, trading |
| **CRDT / sync engine** | Offline-first, automatic conflict resolution | Large conceptual and payload cost | Multiplayer editors, Figma-like apps |

**Default to polling** until latency requirements prove otherwise. A 5-second poll on a dashboard
costs one afternoon; WebSocket infrastructure costs a quarter and never stops needing attention.

---

## Error and loading design is architecture

Decide these once, globally, not per component:

| Situation | Pattern |
| :--- | :--- |
| First load, no data | Skeleton matching the final layout (prevents layout shift) |
| Refetch with data present | Keep old data, show a subtle indicator — never a full spinner |
| Request failed, retryable | Inline message + retry button, scoped to the failed section |
| One section failed | Error boundary around that section; the page survives |
| Session expired (401) | Central interceptor → redirect to login, preserve the return URL |
| Offline | Detect once, show a banner, queue mutations |

The rule that matters: **a failure in one widget must not blank the page.** Error boundaries per
section, not one at the root.

---

## Exercises

**1.** A product list is fetched with key `['products']`. Filters work, but when a user picks
"in stock" the old list flashes and sometimes stays. Why?

<details><summary>Answer</summary>

The filter is not in the cache key, so every filter combination reads and writes the same cache
entry. The library returns the cached (unfiltered) data immediately, and depending on timing the
newer response can be overwritten by an in-flight older one.

Fix: `queryKey: ['products', { inStock, category, page }]`. Each filter set becomes its own entry —
correct data, instant back-and-forth between filters, and no race. Also put the filters in the URL
so the view is shareable (see [lesson 03](03_State_Management.md)).
</details>

**2.** A "publish article" button takes 3 s because the server regenerates a static page. Should it
be an optimistic update?

<details><summary>Answer</summary>

Not fully. Publishing has server-side consequences you cannot predict locally (validation, slug
collisions, the regeneration itself), and wrongly showing "published" is misleading.

Better: optimistically move the item to a **"Publishing…"** state — honest, instant feedback that
doesn't claim success — then confirm or roll back on the response. Longer term, make the endpoint
return as soon as the record is saved and trigger regeneration asynchronously, so the user's action
is genuinely fast instead of merely appearing so.
</details>

**3.** A React Native app, a web app, and two partner integrations all need order data, with
different fields. REST, GraphQL, or tRPC?

<details><summary>Answer</summary>

**GraphQL.** This is exactly its case: multiple independent clients wanting different field sets
from the same graph, evolving on different schedules. REST would spawn per-client endpoints
(`/orders/mobile-summary`) or force over-fetching. tRPC is disqualified outright — partners are
external and not in your TypeScript monorepo.

Budget for the real costs from day one: DataLoader-style batching against N+1, query depth and cost
limits (partners will send expensive queries), persisted queries so you can still cache, and clear
schema ownership.
</details>

---

**Next:** [07 — Decision Playbook](07_Decision_Playbook.md)
