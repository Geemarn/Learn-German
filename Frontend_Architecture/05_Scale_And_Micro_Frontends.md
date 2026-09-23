# 05 — Scale & Micro-Frontends

> **The question this answers:** how do many teams ship to one product without waiting for each
> other?

This lesson is about **organisational** architecture. Micro-frontends solve a people problem, not a
technical one. If you have one team, everything here is pure cost — read it, then don't do it.

---

## The actual problem

You are ready for this discussion only if you can point at one of these:

- A release train where 6 teams must all be green before anyone deploys.
- A 25-minute CI run on every PR, so nobody ships twice a day.
- Teams blocked on a shared dependency upgrade for weeks.
- An acquisition that brought a Vue app into a React company.

If you cannot name one, your problem is a modular monolith away from solved.

---

## 1. Modular monolith (start here, stay here longer than you think)

One deployable app, strict internal boundaries, enforced by tooling.

```
src/features/{billing,catalog,search,account}/   ← one owner each, index.ts only
src/shared/{ui,lib}/                             ← no feature imports allowed
```

Add: CODEOWNERS per feature folder, lint rules blocking cross-feature imports, and route-level lazy
loading so each feature is its own chunk.

| Advantages | Trade-offs |
| :--- | :--- |
| One build, one version of React, no duplication | One CI pipeline — a broken test blocks everyone |
| Refactors across boundaries are a single PR | Deploys are coupled: you ship together |
| Cheapest possible ops | Boundaries need active enforcement |
| Can still lazy-load per route for performance | Very large repos get slow builds eventually |

**This handles surprisingly large organisations.** Reach for it first, always.

---

## 2. Module Federation (runtime-composed micro-frontends)

Independently built and deployed bundles that load each other **at runtime**.

```js
// host
new ModuleFederationPlugin({
  remotes: { checkout: 'checkout@https://cdn.example.com/checkout/remoteEntry.js' },
  shared: { react: { singleton: true, requiredVersion: '^18.0.0' } },
});
```

```jsx
const Checkout = lazy(() => import('checkout/CheckoutApp'));
```

| Advantages | Trade-offs |
| :--- | :--- |
| Teams deploy independently, on their own schedule | Version skew: host and remote can disagree at runtime |
| Fix production in one app without rebuilding the rest | `shared: singleton` is fragile — duplicate React = broken hooks |
| Different release cadences per business area | Integration bugs appear only in production |
| Enables gradual migration of a legacy app | Needs real platform investment: contracts, versioning, observability |

**Non-negotiables if you do this:** a typed contract per remote (published types package), a
fallback UI when a remote fails to load, strict shared-dependency policy, and end-to-end tests that
run against the *composed* app, not just each part.

---

## 3. Build-time integration (packages)

Each team publishes a versioned npm package; the shell depends on specific versions.

| Advantages | Trade-offs |
| :--- | :--- |
| Type-safe, tree-shakeable, one optimised bundle | Independent *deployment* is gone — shell must rebuild |
| Normal dependency management and semver | Upgrade coordination across teams |
| Simple to debug; no runtime surprises | Slow path for urgent fixes |

**Use it when** teams want ownership and isolated testing, but you don't actually need independent
deployment. Often the honest middle ground — and much safer than federation.

---

## 4. Route-level / edge composition

Each team owns whole URLs; a proxy, CDN, or edge worker routes to separate applications.

```
/                  → marketing app   (Astro, SSG)
/shop/*            → storefront      (Next.js)
/account/*         → account app     (React SPA)
```

| Advantages | Trade-offs |
| :--- | :--- |
| Total isolation; each team picks its own stack | Full page reload when crossing a boundary |
| Zero runtime coupling — nothing can break a sibling | Shared auth, session, and nav must be solved centrally |
| Simple to reason about and to operate | Duplicated framework downloads per app |

**Use it when** the sections are genuinely different products with rare cross-navigation. This is
the lowest-risk form of micro-frontends and frequently the right one.

---

## 5. iframes

Still the strongest isolation available in a browser.

| Advantages | Trade-offs |
| :--- | :--- |
| Hard CSS/JS sandbox; nothing leaks either way | Awkward sizing, scroll, and focus behaviour |
| Can host third-party or untrusted code safely | Communication only via `postMessage` |
| Any framework, any version, no conflicts | Poor accessibility and SEO; modals can't escape the frame |

**Use it when** embedding third-party or untrusted content, or wrapping a legacy app during
migration. Not a general composition strategy.

---

## Comparison

| | Modular monolith | Build-time packages | Module federation | Route/edge split | iframes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Independent deploy | no | no | yes | yes | yes |
| Shared design system | trivial | easy | needs discipline | duplicated | hard |
| Bundle efficiency | best | good | medium | poor | worst |
| Failure isolation | none | none | partial | strong | strongest |
| Ops complexity | lowest | low | highest | medium | low |
| Mixed frameworks | no | no | possible | yes | yes |

---

## What you must solve centrally, whatever you choose

Micro-frontends do not remove these; they make each one a cross-team negotiation.

| Concern | Approach |
| :--- | :--- |
| **Design consistency** | One design-system package + design tokens as CSS variables. Non-optional. |
| **Auth** | httpOnly cookies on a shared domain; never pass tokens between frontends |
| **Navigation & layout** | The shell owns the chrome; remotes render into a slot |
| **Cross-app communication** | Custom events or a tiny pub/sub with a *documented* schema. Never a shared mutable store |
| **Observability** | One tracing/error tool with a `team` tag, so you can tell who broke what |
| **Performance budget** | Per-remote byte budget enforced in CI, or the page grows without limit |

---

## BFF — Backend for Frontend

Independent of the composition choice: a thin server owned by the frontend team that shapes data for
the UI.

```
React app  →  BFF (/api/product-page)  →  catalog svc + pricing svc + reviews svc
```

| Advantages | Trade-offs |
| :--- | :--- |
| One request per screen instead of six waterfalls | Another deployable to own and monitor |
| Frontend evolves without backend team sign-off | Can become a second business-logic layer if undisciplined |
| Secrets, keys, and aggregation stay server-side | Duplicated effort if several BFFs need the same shaping |
| Adapts ugly internal APIs to clean UI shapes | |

**Rule:** a BFF **shapes and aggregates**; it does not own business rules. The moment it starts
making pricing decisions, it has become an untested microservice with no owner.

---

## Exercises

**1.** A 40-engineer company across 5 teams wants micro-frontends because "CI takes 22 minutes and
deploys are coupled". What do you propose?

<details><summary>Answer</summary>

Not micro-frontends yet. Both stated problems have far cheaper fixes:

- 22-minute CI → affected-only builds and tests (Turborepo/Nx), remote caching, sharded test runs.
  Usually drops to 3–5 minutes.
- Coupled deploys → feature flags and trunk-based development, so merging is decoupled from
  releasing. Teams ship behind flags on their own schedule with one pipeline.

Do modular monolith + CODEOWNERS + import boundaries + flags first. Revisit federation only if
teams still block each other after that — and then start with route-level splitting, which is far
cheaper to run than runtime federation.
</details>

**2.** After adopting Module Federation, the host is React 18 and one remote was built against
React 17. Users see "Invalid hook call" on that page. What went wrong and how do you prevent it?

<details><summary>Answer</summary>

Two React copies ended up on the page, so the remote's hooks called into a different React instance
than the one rendering. `shared: { react: { singleton: true } }` was either missing or its
`requiredVersion` allowed a mismatch, so the remote loaded its own React.

Prevention: mark `react`/`react-dom` as `singleton` with a strict `requiredVersion`, fail the remote
build if its React major differs from the host contract, and add a composed-app smoke test in CI
that renders every remote inside the real host. This class of bug exists only at runtime, which is
precisely why federation needs platform investment.
</details>

**3.** Your product page needs data from 4 services and currently makes 4 sequential browser
requests (the 4th needs an ID from the 2nd). Compare a BFF against GraphQL here.

<details><summary>Answer</summary>

Both collapse the waterfall into one round trip; choose on ownership.

A **BFF** is right if the shaping is specific to this screen and you want one team to own the
endpoint. Fastest to build, fully typed, easy to cache per route.

**GraphQL** (with a gateway over the 4 services) is right if *many* clients need different slices of
the same graph — web, mobile, partners. It costs more up front (schema ownership, N+1 protection,
query cost limits) but stops you from writing a new BFF endpoint per screen.

One product, one web client → BFF. Several clients with diverging needs → GraphQL.
</details>

---

**Next:** [06 — Data Layer](06_Data_Layer.md)
