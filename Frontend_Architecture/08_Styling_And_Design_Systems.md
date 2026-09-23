# 08 — Styling & Design Systems

> **The question this answers:** how do I change a colour in one place and have it change everywhere,
> without breaking something I've never heard of?

CSS has one dangerous property: it is **global by default**. Every styling approach below is a
strategy for containing that. Judge each by two things — how it scopes styles, and how much work it
does at runtime.

> This lesson is about *organising* CSS. For the layout mechanics themselves — flexbox, grid, and
> the patterns built from them — see [13](13_Flexbox.md), [14](14_CSS_Grid.md), and
> [15](15_Layout_Recipes.md).

---

## Styling approaches

### Global CSS + a naming convention (BEM)

```css
.card { }
.card__title { }
.card--featured { }
```

| Advantages | Trade-offs |
| :--- | :--- |
| Zero tooling, zero runtime, works anywhere | Scoping depends entirely on human discipline |
| Anyone can read it; browser devtools map 1:1 | Dead CSS accumulates — nobody dares delete |
| Smallest possible payload | Specificity wars, then `!important` |

**Use for:** small sites, email templates, or the global reset layer of a larger system.

### CSS Modules

```jsx
import s from './Card.module.css';
<div className={s.card} />       // → .Card_card__x7f2a
```

| Advantages | Trade-offs |
| :--- | :--- |
| Real scoping, enforced by the build, not by humans | Dynamic values need CSS variables or inline style |
| Zero runtime cost; plain CSS files | Composing across files is clumsy |
| Unused classes are visible to tooling | Two files open for every component |

**Use for:** a safe, boring, fast default. Ages extremely well.

### Runtime CSS-in-JS (styled-components, Emotion)

```jsx
const Button = styled.button`
  background: ${(p) => (p.tone === 'danger' ? 'red' : 'blue')};
`;
```

| Advantages | Trade-offs |
| :--- | :--- |
| Styles co-located with the component; deletes together | **Runtime cost** — style computation on every render |
| Props drive styles naturally; full JS expressiveness | Poor fit for Server Components (needs client JS) |
| Theming via context is straightforward | Larger bundle; serialisation work during hydration |

**Use for:** existing codebases that already use it. **Don't start new projects here** — the
zero-runtime options below give you the same ergonomics without the cost.

### Zero-runtime CSS-in-JS (vanilla-extract, Linaria, Panda)

Written in TypeScript, extracted to static CSS at **build time**.

```ts
export const button = recipe({
  base: { padding: vars.space.md },
  variants: { tone: { danger: { background: vars.color.danger } } },
});
```

| Advantages | Trade-offs |
| :--- | :--- |
| Type-safe styles *and* no runtime — best of both | Build-time setup; less mature tooling |
| Tokens become TypeScript constants you can't typo | Truly dynamic values still need CSS variables |
| Works with Server Components | Smaller community; fewer examples |

**Use for:** design systems where token correctness matters and you can afford setup time.

### Utility-first (Tailwind)

```jsx
<div className="flex items-center gap-3 rounded-lg bg-surface p-4 shadow-sm">
```

| Advantages | Trade-offs |
| :--- | :--- |
| No naming, no unused CSS, no file switching | Markup is noisy; long class strings |
| The config *is* the design system — constraints by default | Learning curve; needs `cn()`/`cva` discipline |
| CSS size plateaus as the app grows | Conditional variants get ugly without helpers |
| Trivially safe to delete markup | Non-frontend contributors find it opaque |

**Use for:** product teams shipping fast, especially with headless component libraries. Pair with
`cva`/`tailwind-variants` so variants live in one typed place instead of ternaries in JSX.

---

## Comparison

| | Global CSS | CSS Modules | Runtime CSS-in-JS | Zero-runtime | Tailwind |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Scoping | manual | automatic | automatic | automatic | n/a (atomic) |
| Runtime cost | none | none | **yes** | none | none |
| Type safety | none | weak | partial | strong | via plugin |
| Server Components | fine | fine | awkward | fine | fine |
| CSS size growth | linear | linear | linear | linear | plateaus |
| Dynamic per-prop styles | hard | medium | easiest | medium | medium |

---

## Design tokens: the actual foundation

Tokens are the contract between design and code. Get this layer right and the styling technology
above it barely matters.

```css
/* 1. primitives — raw values, never used directly in components */
--blue-600: #2563eb;
--gray-50:  #f9fafb;

/* 2. semantic tokens — what it MEANS, used by components */
--color-surface:       var(--gray-50);
--color-action:        var(--blue-600);
--color-text-muted:    var(--gray-500);
--space-md:            0.75rem;
--radius-card:         0.5rem;
```

**Components reference only semantic tokens.** This is the rule that makes dark mode a 20-line
change instead of a three-week audit:

```css
[data-theme='dark'] {
  --color-surface: var(--gray-900);
  --color-text:    var(--gray-50);
}
```

| Advantages | Trade-offs |
| :--- | :--- |
| Theming, white-labelling, and dark mode become trivial | Requires naming discipline up front |
| One change propagates everywhere, predictably | Two layers of indirection to trace a colour |
| Designers and engineers share vocabulary | Semantic naming is genuinely hard to get right |

**Why CSS variables specifically** rather than a JS theme object: they cross framework boundaries,
work in Server Components, need no re-render to change, and are visible in devtools. A JS theme
object forces every consumer through React context — and re-renders the tree on theme change.

---

## Design system layers

Build in this order. Skipping to layer 3 is why design systems fail.

| Layer | Contents | Changes |
| :--- | :--- | :--- |
| 1. **Tokens** | colour, space, type, radius, shadow, motion | rarely |
| 2. **Primitives** | Button, Input, Text, Stack, Icon | occasionally |
| 3. **Components** | Card, Table, Modal, Combobox | regularly |
| 4. **Patterns** | LoginForm, DataTableToolbar, EmptyState | often |

Layer 4 usually belongs in the **app**, not the shared package. Patterns encode product decisions,
and product decisions are exactly what different apps need to differ on.

### Distribution

| Concern | Approach |
| :--- | :--- |
| Packaging | `packages/ui` in a monorepo; publish to a private registry only if consumers are external |
| Versioning | semver, and treat a changed token value as a **breaking** change |
| Docs | Storybook with a story per variant; the docs are the acceptance criteria |
| Adoption | ship a codemod with breaking changes, or teams will simply not upgrade |
| Escape hatch | accept `className` on every component — without it, teams fork your library |

That last row matters more than it looks. A design system with no escape hatch gets copy-pasted, and
then you have two design systems.

---

## Anti-patterns

| Anti-pattern | Why it hurts | Fix |
| :--- | :--- | :--- |
| Hard-coded `#3b82f6` in a component | Invisible to theming; dark mode breaks | Lint rule banning raw hex outside token files |
| Styling by descendant selector (`.card .btn`) | Breaks when markup moves; specificity creep | Style the component, not the context |
| Primitives used directly (`--blue-600`) | Loses all semantic meaning | Only semantic tokens in components |
| `!important` | A specificity fight you will lose later | Fix the source specificity |
| Component both styled and configurable via 15 style props | Design system with no opinion | Fixed variants + one `className` escape hatch |
| Two styling systems in one app | Double payload, unclear precedence | Pick one; migrate with a deadline |
| Media queries scattered per component | Breakpoints drift | Breakpoints as tokens, used everywhere |

---

## Exercises

**1.** Dark mode was added by wrapping the app in a React `ThemeProvider` holding a JS theme object.
Toggling it is janky and re-renders everything. Better approach?

<details><summary>Answer</summary>

The theme object lives in React state, so switching it changes context and re-renders every consumer
— the jank is React work, not CSS work.

Move themes to **CSS variables** switched by a `data-theme` attribute on `<html>`. Toggling writes
one attribute; the browser recalculates styles and **zero React components re-render**. Persist the
choice in `localStorage` and set it in an inline script before first paint to avoid a flash of the
wrong theme. Keep a context only for exposing the current value to code that must branch on it.
</details>

**2.** A 6-app company has a `packages/ui` library. Teams keep copying components out of it and
editing them. What's the likely cause?

<details><summary>Answer</summary>

Almost always a missing escape hatch or too slow a change process. Teams fork when they hit a real
requirement the component can't express and the alternative is waiting two sprints for a PR review
from the platform team.

Fixes: accept `className`/`style` and spread `...rest` onto the root element of every component;
expose compound sub-components so layout can vary ([lesson 04](04_Component_Patterns.md)); publish a
contribution path with a fast review SLA. Then measure forks — each one is a bug report about your
API, not about the team.
</details>

**3.** Your team wants to migrate 400 components from styled-components to Tailwind for performance.
How do you scope it?

<details><summary>Answer</summary>

First verify the premise: profile to confirm style serialisation is actually material. Often the
bigger win is just removing the runtime from the *hot* components (long lists, tables) rather than
all 400.

If the migration is justified: (a) go **token-first** — express the design tokens in the Tailwind
config so both systems render identically during the transition; (b) migrate leaf primitives first,
since everything depends on them and they are the smallest; (c) migrate the rest per feature slice
so each PR is reviewable and shippable; (d) add a lint rule banning new `styled` calls, so the
direction is enforced without a freeze; (e) set a deletion deadline for the styled-components
dependency, because running both systems indefinitely costs more than either alone.
</details>

---

**Next:** [09 — Performance Architecture](09_Performance_Architecture.md)
