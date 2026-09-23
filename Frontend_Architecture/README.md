# Frontend Architecture — A Practical Course

Frontend architecture is not a framework choice. It is a set of **decisions about where things
live**: where HTML is produced, where state is kept, where data is fetched, and where the
boundaries between teams are drawn.

Every architecture on this list is *correct somewhere*. The skill is matching the architecture to
the constraints you actually have — team size, SEO needs, data freshness, and how long the product
must live.

---

## The four questions every frontend architecture answers

| Question | Decision area | Lesson |
| :--- | :--- | :--- |
| Where does the HTML come from? | Rendering strategy | [01](01_Rendering_Architectures.md) |
| How is the code arranged in folders? | Code organisation | [02](02_Code_Organization.md) |
| Where does the data live while the app runs? | State management | [03](03_State_Management.md) |
| How do teams ship without blocking each other? | Scale & boundaries | [05](05_Scale_And_Micro_Frontends.md) |

---

## Part I — Structural decisions

These four choices define the shape of the application.

| # | File | What it covers |
| :--- | :--- | :--- |
| 01 | [Rendering Architectures](01_Rendering_Architectures.md) | CSR, SSR, SSG, ISR, streaming, React Server Components, islands |
| 02 | [Code Organisation](02_Code_Organization.md) | Layered, feature-sliced, atomic design, hexagonal, monorepos |
| 03 | [State Management](03_State_Management.md) | Local, lifted, context, Flux/Redux, atomic, server state, state machines |
| 04 | [Component Patterns](04_Component_Patterns.md) | Container/presentational, compound, headless, hooks, composition |
| 05 | [Scale & Micro-Frontends](05_Scale_And_Micro_Frontends.md) | Modular monolith, module federation, iframes, BFF, design systems |
| 06 | [Data Layer](06_Data_Layer.md) | REST, GraphQL, tRPC, BFF, caching, optimistic updates, real-time |
| 07 | [Decision Playbook](07_Decision_Playbook.md) | Scenario → recommended architecture, migration paths, anti-patterns |

## Part II — Cross-cutting concerns

Every architecture in Part I still has to solve all five of these. They are where products succeed
or quietly rot.

| # | File | What it covers |
| :--- | :--- | :--- |
| 08 | [Styling & Design Systems](08_Styling_And_Design_Systems.md) | CSS Modules, CSS-in-JS, Tailwind, design tokens, system layers, distribution |
| 09 | [Performance Architecture](09_Performance_Architecture.md) | LCP/INP/CLS, diagnosis order, bundle splitting, budgets, images, render cost |
| 10 | [Testing Architecture](10_Testing_Architecture.md) | Testing trophy, component/integration/E2E, MSW, flake control, CI shape |
| 11 | [Auth & Security](11_Auth_And_Security.md) | Token storage, sessions vs JWT, OIDC/PKCE, capabilities, XSS, CSP, supply chain |
| 12 | [Observability & Delivery](12_Observability_And_Delivery.md) | Errors, spans, structured logs, feature flags, canary rollouts, deploy hazards |

## Part III — CSS layout

The hands-on half: illustrated tutorials on the two layout systems everything else is built from.

| # | File | What it covers |
| :--- | :--- | :--- |
| 13 | [Flexbox](13_Flexbox.md) | Main/cross axes, `justify-content`, `align-items`, the `flex` shorthand, wrapping, shrink traps |
| 14 | [CSS Grid](14_CSS_Grid.md) | Lines and tracks, `fr`, placement, named areas, `auto-fit` vs `auto-fill`, alignment |
| 15 | [Layout Recipes](15_Layout_Recipes.md) | Ten copy-paste patterns: app shells, card grids, centring, full-bleed, debugging checklist |

Both tutorials are illustrated with diagrams in [`illustrations/`](illustrations/) and come with a
live playground you can open directly in a browser — no build step:

**▶ [`playground/layout-playground.html`](playground/layout-playground.html)** — change
`justify-content`, `flex-grow`, `grid-template-columns`, and `auto-fit`/`auto-fill` with sliders
and dropdowns, and read the generated CSS underneath.

---

## The one table to remember

A rough map of where each architecture wins. Read across the row, not down the column.

| Architecture | Wins at | Loses at | Typical product |
| :--- | :--- | :--- | :--- |
| Client-rendered SPA (CSR) | Rich interaction, cheap hosting | First paint, SEO | Dashboard, internal tool |
| Static generation (SSG) | Speed, cost, resilience | Fresh or personalised data | Docs, marketing, blog |
| Server rendering (SSR) | SEO + fresh data | Server cost, complexity | E-commerce, news |
| Incremental static (ISR) | Static speed with updates | Stale windows, cache reasoning | Catalogue, large content site |
| Server Components + streaming | Small bundles, fast data | New mental model, framework lock-in | Modern full-stack app |
| Islands / partial hydration | Almost no JS shipped | Awkward for app-like UI | Content site with widgets |
| Micro-frontends | Independent team delivery | Bundle duplication, ops overhead | Large org, many teams |

---

## How to work through this material

1. Read [01 Rendering](01_Rendering_Architectures.md) first — it constrains everything else.
2. Read [03 State](03_State_Management.md) next. Most "bad architecture" pain is really state pain.
3. Skim [02](02_Code_Organization.md) and [04](04_Component_Patterns.md) and apply one idea to a
   project you already have.
4. Only read [05 Scale](05_Scale_And_Micro_Frontends.md) when you have more than one team. Before
   that it is a solution looking for a problem.
5. Use [07 Decision Playbook](07_Decision_Playbook.md) as a checklist when starting something new.
6. Then work through Part II. [09 Performance](09_Performance_Architecture.md) and
   [11 Auth & Security](11_Auth_And_Security.md) are the two where mistakes are most expensive to
   discover late.

Each lesson ends with **Exercises** — small design tasks with worked answers. Do them; architecture
is learned by making the trade-off, not by reading about it.

---

## Three rules that survive every trend

1. **Push work as far away from the user's device as you can.** Build time is cheaper than server
   time, which is cheaper than browser time.
2. **Architecture is about reversibility.** Prefer the design that is cheapest to undo when you are
   wrong, because you will be wrong.
3. **Complexity must be paid for by a real constraint.** If you cannot name the constraint that
   forces the complexity, delete the complexity.
