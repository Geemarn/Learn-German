# 02 — Code Organisation

> **The question this answers:** when a new requirement arrives, how many folders do I have to open?

A good folder structure minimises the distance between "a change was requested" and "all the files
that change are next to each other". That is the entire goal. Everything below is judged by it.

---

## 1. Layered / "group by technical type"

The default most projects start with.

```
src/
  components/    Button.tsx  ProductCard.tsx  InvoiceRow.tsx
  hooks/         useCart.ts  useAuth.ts  useInvoices.ts
  services/      cart.ts  auth.ts  invoices.ts
  utils/         format.ts
  pages/         Cart.tsx  Login.tsx  Invoices.tsx
```

| Advantages | Trade-offs |
| :--- | :--- |
| Zero learning curve; everyone knows where a hook goes | A single feature is smeared across 5 folders |
| Works fine while the app is small | `components/` becomes a 200-file dumping ground |
| No decisions to make up front | Deleting a feature means hunting for its leftovers |

**Use it when:** fewer than ~30 components, one or two developers, or a prototype.

**The signal to move on:** you change one feature and your PR touches six unrelated folders.

---

## 2. Feature-Sliced / Modular

Group by **what it does for the user**, not what it is technically.

```
src/
  features/
    cart/
      components/  CartDrawer.tsx  CartRow.tsx
      hooks/       useCart.ts
      api/         cartApi.ts
      model/       cart.types.ts
      index.ts     <- the ONLY public entry point
    checkout/
    invoices/
  shared/
    ui/            Button.tsx  Modal.tsx      (no business knowledge)
    lib/           formatMoney.ts  http.ts
  app/             routing, providers, layout
```

Two rules make this work, and without them it degrades into layered-with-extra-steps:

1. **Import only through `index.ts`.** Never `features/cart/hooks/useCart` from outside.
2. **Features may not import each other.** If `checkout` needs cart data, either lift the shared
   part into `shared/`, or have the `app/` layer compose them.

| Advantages | Trade-offs |
| :--- | :--- |
| A feature is one folder: readable, deletable, ownable | Requires discipline; boundaries erode silently |
| Clear code ownership → clean PRs, easy review | "Is this shared or feature-specific?" debates |
| Natural seam for later code-splitting or extraction | Some duplication across features (usually healthy) |
| Onboarding is fast: "you own `features/billing`" | Over-engineered for a 10-file app |

**Use it when:** more than one developer, or more than roughly 5 distinct product areas. This is the
best default for serious applications.

**Enforce it with tooling**, not goodwill:

```json
// .eslintrc — import/no-restricted-paths
{
  "zones": [
    { "target": "./src/features/cart", "from": "./src/features/checkout" },
    { "target": "./src/shared", "from": "./src/features" }
  ]
}
```

That last zone is the important one: **shared code must never import feature code.** Dependencies
point inward and downward only.

---

## 3. Atomic Design

A vocabulary for the *UI layer* specifically: atoms → molecules → organisms → templates → pages.

```
ui/
  atoms/      Button  Input  Icon
  molecules/  SearchField (Input + Button)
  organisms/  SiteHeader (Logo + Nav + SearchField)
  templates/  DashboardLayout
```

| Advantages | Trade-offs |
| :--- | :--- |
| Shared language with designers (mirrors Figma) | Endless "is this a molecule or an organism?" arguments |
| Encourages genuinely reusable primitives | Says nothing about business logic or data |
| Pairs well with Storybook and design systems | Categories add ceremony without adding capability |

**Practical advice:** use atomic design *inside* a design system package, and use feature slices for
the application. Trying to organise a whole app atomically fails because business features do not
decompose into atoms.

---

## 4. Hexagonal / Clean Architecture on the frontend

Keep the domain in the middle; push frameworks and transports to the edges.

```
domain/          Order, Money, canRefund()        — pure TS, no React, no fetch
application/     placeOrder(), refundOrder()      — use cases, orchestration
infrastructure/  httpOrderRepository.ts           — REST/GraphQL details
ui/              OrderPage.tsx                    — React, calls use cases
```

The rule: **`domain/` imports nothing.** UI and infrastructure depend on the domain, never the
reverse.

| Advantages | Trade-offs |
| :--- | :--- |
| Business rules testable with no DOM and no mocks | Lots of indirection and mapping code |
| Swap REST → GraphQL by replacing one folder | Overkill for CRUD screens |
| Survives framework migrations | Team must actually understand the discipline |

**Use it when:** the frontend holds real business rules — pricing engines, insurance quoting,
trading, medical scheduling, tax calculation. **Skip it when** your app is mostly forms over an API;
there you'd be wrapping `fetch` in three layers to no benefit.

---

## 5. Monorepo

One repository, multiple packages, shared tooling.

```
apps/
  web/           customer site
  admin/         internal console
packages/
  ui/            design system
  api-client/    generated, typed client
  config/        eslint / tsconfig / tailwind presets
```

| Advantages | Trade-offs |
| :--- | :--- |
| Atomic cross-package changes in one PR | Needs real tooling (Turborepo, Nx, pnpm workspaces) |
| One version of React, one lint config, no drift | CI must be smart or every push rebuilds everything |
| Shared types from DB to UI — refactors are safe | Coarse repo permissions; large clones |
| Easy internal reuse without publishing to npm | Tempts teams into accidental tight coupling |

**Use it when:** two or more deployable apps share meaningful code.

**Avoid it when:** you have exactly one app. A monorepo with one app is just a repo with extra
config files.

---

## Choosing: a quick decision table

| Situation | Structure |
| :--- | :--- |
| Prototype, solo, < 2 weeks | Layered |
| Single product, 2–8 devs | Feature-sliced + `shared/ui` |
| Design system used by several apps | Atomic design inside a package, in a monorepo |
| Heavy domain logic in the browser | Feature-sliced + hexagonal in the complex features only |
| Several apps, one company | Monorepo + feature slices per app |

You are allowed to apply hexagonal layering to *two* complex features and leave the other twenty as
plain feature folders. Architecture is applied unevenly on purpose — spend complexity where the
difficulty actually is.

---

## Smells and their fixes

| Smell | What it really means | Fix |
| :--- | :--- | :--- |
| `utils/index.ts` with 40 unrelated exports | No home for domain concepts | Move each into the feature that owns it |
| Circular imports between features | Boundaries are fictional | Lift the shared concept up, or invert with an event/callback |
| A component with 600 lines | Rendering and logic are fused | Extract a hook for logic, keep JSX dumb |
| `components/common/common/` | Nobody knows what is shared | One `shared/ui` level, no nesting by vagueness |
| Every PR edits the same three files | Hidden god-module | Split by feature; those files are a bottleneck |

---

## Exercises

**1.** A 4-person team keeps hitting merge conflicts in `src/components/` and `src/hooks/`. Which
change helps most, and why does it help *conflicts* specifically?

<details><summary>Answer</summary>

Move to **feature slices**. Conflicts happen because unrelated work lands in the same directories
and barrel files. When each developer owns `features/x`, their edits are physically disjoint, so
git has nothing to merge. Add `import/no-restricted-paths` so the separation cannot quietly rot.
</details>

**2.** Where does `formatCurrency` belong — `shared/lib` or `features/billing`?

<details><summary>Answer</summary>

`shared/lib` if it is pure formatting used by several features. But if it encodes billing rules
(rounding for VAT, currency per contract type) it belongs in `features/billing` — that is domain
knowledge, and putting it in `shared/` invites every other feature to depend on billing rules.
Test: could you explain it without mentioning the product? If no, it isn't shared.
</details>

**3.** Your team proposes hexagonal architecture for a 12-screen CRUD admin panel. Argue the other
side.

<details><summary>Answer</summary>

There is no domain to protect: the screens are forms over an API, so `domain/` would contain
anonymous data shapes and `application/` would contain one-line pass-throughs to `infrastructure/`.
You'd triple the file count and add mapping bugs for zero testability gain. Feature slices with a
typed API client give you the same safety at a fraction of the cost. Revisit if genuine rules
(approval chains, entitlement calculations) ever move into the client.
</details>

---

**Next:** [03 — State Management](03_State_Management.md)
