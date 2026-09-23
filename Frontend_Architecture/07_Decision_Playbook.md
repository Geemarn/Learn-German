# 07 — Decision Playbook

> **The question this answers:** it's Monday, I'm starting something. What do I actually pick?

Use this as a checklist. The lessons explain *why*; this file is *what*.

---

## Step 1 — Answer six questions before writing code

| # | Question | Why it decides things |
| :--- | :--- | :--- |
| 1 | Does Google need to index it? | Rules out CSR |
| 2 | Is it behind a login? | Rules out static; kills CDN caching of the data |
| 3 | How stale may data be — seconds, minutes, or days? | Picks SSR vs ISR vs SSG |
| 4 | How many engineers, in how many teams? | Picks monolith vs micro-frontends |
| 5 | Are there other clients (mobile, partners)? | Picks REST/GraphQL vs tRPC/RSC |
| 6 | How long must this live — 3 months or 8 years? | Decides how much structure to buy |

Question 6 is the one people skip. A campaign microsite and a core banking console deserve opposite
answers to everything else.

---

## Step 2 — Scenario → architecture

### Marketing site / docs / blog
```
Rendering   SSG (+ islands for the few interactive bits)
Structure   content collections; pages are thin
State       URL + local state only
Data        markdown/CMS at build time
Deploy      CDN, no server
```
**Why:** fastest possible, essentially free, cannot go down. Adding a framework with a server here
buys nothing and costs money forever.

### Internal admin / dashboard (behind login)
```
Rendering   CSR
Structure   feature slices, route-level lazy loading
State       server-state library + one small store (Zustand) + URL for filters
Data        REST via a generated typed client, or tRPC if same monorepo
Components  headless primitives + your own thin design system
```
**Why:** no SEO value, known audience, long sessions. Optimise for development speed and in-app
navigation, not first paint.

### E-commerce storefront
```
Rendering   ISR for product/category pages · SSR for cart & checkout · client-side for price/stock
Structure   feature slices; catalog / cart / checkout / account
State       server-state cache for catalogue · store for cart · state machine for checkout
Data        BFF aggregating catalog + pricing + inventory
Extra       per-route performance budget in CI; Core Web Vitals are revenue
```
**Why:** product pages must be indexable and instant but change rarely — ISR. Checkout is personal
and must be current — SSR. The two live in one app with different per-route strategies.

### SaaS product (marketing + app in one place)
```
Rendering   SSG for /(marketing) · RSC+streaming or CSR for /(app)
Structure   monorepo: apps/web + packages/ui + packages/api-client
State       server-state library; atomic store for cross-feature client state
Data        tRPC or REST+codegen; one BFF layer if services are many
Auth        httpOnly cookies, middleware-protected route groups
```
**Why:** the two halves have opposite requirements. Route groups let one codebase serve both.

### Content platform with user-generated content
```
Rendering   ISR with on-demand revalidation on publish
Structure   feature slices
State       server-state cache with optimistic updates for votes/likes
Data        REST + CDN caching for public content; personalised bits fetched client-side
Real-time   SSE for notifications; polling for counters
```
**Why:** public content is shared across users, so cache it at the edge; only the personal
overlay (your votes, your feed position) needs per-user fetching.

### Collaborative editor
```
Rendering   CSR (app shell only)
Structure   hexagonal — the document model is real domain logic, framework-free
State       CRDT/sync engine as source of truth; UI derives from it
Data        WebSocket; offline-first persistence in IndexedDB
Components  headless + heavy virtualisation
```
**Why:** the document model is the product. Keep it in pure TypeScript so it is testable and
survives every UI rewrite.

---

## Step 3 — Sequence the work

Adopt in this order. Each step earns the right to the next.

1. **TypeScript strict + a typed API client.** Removes a whole category of bug. Do this first.
2. **Server-state library.** Deletes the most code for the least effort.
3. **Feature slices + enforced import boundaries.** Makes growth survivable.
4. **Per-route rendering strategy.** Performance where it pays.
5. **Design system package.** Only when a second app or a second team exists.
6. **Monorepo.** Only when there are two deployables.
7. **Micro-frontends.** Only when teams demonstrably block each other after 1–6.

Most teams get 80 % of the benefit from steps 1–3.

---

## Migration paths that work

| From → To | Approach |
| :--- | :--- |
| Layered → feature-sliced | Don't big-bang it. Create `features/`, move **one** feature completely, add the lint rule for that folder, repeat. Leave `components/` shrinking in place. |
| CSR → SSR/RSC | Route by route behind a proxy. Start with the highest-traffic public page; it has the most to gain and the clearest metric. |
| Redux holding server data → query library | Migrate one slice at a time: replace the thunk with `useQuery`, delete the slice, keep the selector name so components don't change. |
| Monolith → micro-frontends | Extract the *most independent* feature first, not the most painful one. You are testing the platform, not solving the hardest case. |
| jQuery/legacy → modern | Strangler pattern: mount the new app into a container div, migrate page by page, share auth via cookies. |

**Universal rule:** never run two architectures in parallel without a written deadline for deleting
the old one. Permanent dual-stack is worse than either stack.

---

## Anti-patterns, ranked by damage

| # | Anti-pattern | Why it's expensive |
| :--- | :--- | :--- |
| 1 | Resume-driven architecture | Complexity with no constraint behind it; the team pays rent forever |
| 2 | Micro-frontends with one team | All coordination costs, zero coordination benefit |
| 3 | Server data in a global store | A cache with no invalidation — guaranteed stale UI |
| 4 | SSR everything "for performance" | Slower TTFB and a server bill, for pages nobody indexes |
| 5 | One shared `utils/` god module | Every PR touches it; no ownership, no boundaries |
| 6 | Abstracting on the first duplication | You abstract the wrong axis and every future change fights it |
| 7 | Filters and tabs in state, not the URL | Refresh loses work; links can't be shared; back button lies |
| 8 | No performance budget | Bundles only grow; nobody is ever responsible |
| 9 | Component library with 40-prop components | Configuration where composition was needed |
| 10 | Rewrite instead of strangle | 18 months of no features, then the same problems in new syntax |

---

## Signals that you chose wrong

Architecture problems show up as *process* symptoms before they show up in code.

| Symptom | Likely cause | Lesson |
| :--- | :--- | :--- |
| "Where do I put this file?" asked weekly | No clear structure convention | [02](02_Code_Organization.md) |
| Every feature touches 8 files across 5 folders | Grouped by type, not by feature | [02](02_Code_Organization.md) |
| Stale data until refresh | Manual server-state handling | [03](03_State_Management.md) |
| Typing in one input re-renders the whole page | Fast-changing value in a broad context | [03](03_State_Management.md) |
| Nobody dares touch `<DataTable>` | Boolean-prop god component | [04](04_Component_Patterns.md) |
| Network panel shows a staircase | Request waterfall | [06](06_Data_Layer.md) |
| Merge conflicts in the same files every week | Missing feature boundaries | [02](02_Code_Organization.md) |
| Releases wait for all teams | Coupled deploys | [05](05_Scale_And_Micro_Frontends.md) |
| Dark mode would take three weeks | Hard-coded colours, no semantic tokens | [08](08_Styling_And_Design_Systems.md) |
| Bundle grew 300 KB and nobody noticed | No performance budget in CI | [09](09_Performance_Architecture.md) |
| High coverage, still shipping broken flows | Wrong test shape — unit-heavy, no integration | [10](10_Testing_Architecture.md) |
| Can't log a compromised user out immediately | Long-lived JWTs, no revocation | [11](11_Auth_And_Security.md) |
| Bugs are found by users, not by you | No error tracking or release tagging | [12](12_Observability_And_Delivery.md) |
| Deploys feel dangerous | Deploy coupled to release; no flags | [12](12_Observability_And_Delivery.md) |

---

## The one-page cheat sheet

```
SEO needed?            no  → CSR
                       yes → content stable?   yes → SSG
                                               no  → per-user?  yes → SSR
                                                               no  → ISR

Data from an API?      → server-state library. Always. Not your store.
Shareable view state?  → URL.
Global client state?   → one small atomic store. Not Redux, unless you need the audit trail.
Reusable behaviour?    → hook.  Reusable looks? → component.  Both? → headless.
Component got 4+ booleans? → you needed composition.
More than one team blocked? → fix CI and feature flags first. Then talk micro-frontends.
Can't name the constraint forcing the complexity? → delete the complexity.
```

---

## Exercises

**1.** A startup with 3 engineers is building a B2B SaaS with a marketing site, an app, and a
customer-facing API. They propose: micro-frontends, GraphQL, Redux, and a design system package.
Rewrite the plan.

<details><summary>Answer</summary>

Keep one thing, defer three.

- **Micro-frontends → no.** Three engineers cannot be blocked by each other. One Next.js app with
  route groups: `(marketing)` static, `(app)` behind auth.
- **GraphQL → not yet.** They *do* have a public API requirement, which is the real argument for it
  — but ship REST with OpenAPI first (one service, one client) and introduce GraphQL when partner
  demand is concrete. A schema nobody queries is pure overhead.
- **Redux → no.** TanStack Query for server data, `useState` plus one Zustand store for the rest.
- **Design system package → premature but cheap to prepare.** Keep `shared/ui` in the app now,
  structured so it can be extracted to `packages/ui` the day a second app exists.

Plan: one repo, feature slices, SSG + protected app routes, typed REST client, query library.
Revisit at ~12 engineers.
</details>

**2.** An 8-year-old Angular 1 app, 200k lines, still generating revenue. Leadership wants a React
rewrite in 6 months. Your counter-proposal?

<details><summary>Answer</summary>

A 200k-line rewrite in 6 months will miss, and during the attempt you ship no features while
competitors do. Counter-propose the **strangler pattern**:

1. Put the new React app behind the same domain, routed per URL prefix (route-level composition from
   [lesson 05](05_Scale_And_Micro_Frontends.md)). Share auth via httpOnly cookies.
2. Migrate the highest-value, most-isolated route first — prove the pipeline end to end with real
   users in weeks, not months.
3. Migrate by user journey, never by technical layer. Each migration ships value on its own.
4. Freeze feature work in the legacy app; all new work lands in React, which makes the new stack the
   default without a flag day.
5. Publish a burn-down of remaining legacy routes so leadership sees continuous progress.

You get revenue continuity, reversibility at every step, and learning applied early rather than
discovered at month five.
</details>

**3.** Your product page scores 42 on Lighthouse. The team wants to switch frameworks. What do you
investigate first?

<details><summary>Answer</summary>

Framework choice is almost never the cause. Investigate, in order:

1. **Images** — unoptimised hero images and missing dimensions usually dominate LCP and CLS.
2. **Third-party scripts** — tag managers, chat widgets, and analytics often outweigh the app bundle.
3. **Bundle composition** — run the analyser. Typically one accidental import (a full icon set, a
   moment.js locale bundle, a charting library on a page with no chart).
4. **Waterfalls** — staircase in the network panel ([lesson 06](06_Data_Layer.md)).
5. **Hydration cost** — only now does rendering architecture enter, and the fix is usually islands
   or Server Components rather than a different framework.

Fixing 1–3 typically moves the score 30+ points in days. A framework migration costs months and
would carry all four problems along with it.
</details>

---

**Next:** Part II begins with [08 — Styling & Design Systems](08_Styling_And_Design_Systems.md)
· **Index:** [Course index](README.md)
