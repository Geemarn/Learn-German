# 03 — State Management

> **The question this answers:** who owns this value, and who is allowed to change it?

Most "our frontend is a mess" complaints are state complaints. The single most valuable idea in this
whole course is the one in the next section.

---

## First: split state into two kinds

| | **Client state** | **Server state** |
| :--- | :--- | :--- |
| Examples | Is the modal open, form draft, selected tab, theme | Users list, order details, product catalogue |
| Owner | The browser | The database |
| Truth | Whatever the user just did | Somewhere else, and it changes without you |
| Needs | Predictable updates | Caching, refetching, staleness, retries, dedup |

Treating server state as client state is the original sin of frontend architecture. It leads to
hand-written `loading` / `error` / `data` triples in every component, stale screens, duplicate
requests, and a Redux store that is really a bad cache.

**Rule:** use a **server-state library** (TanStack Query, SWR, RTK Query, Apollo) for anything that
came from an API, and keep your state manager for genuine client state. This one split removes
maybe 70 % of typical state code.

---

## The escalation ladder

Start at step 1. Only climb when you feel actual pain. Never start at step 5.

### 1. Local component state

```jsx
const [isOpen, setIsOpen] = useState(false);
```

**Advantages:** no ceremony, perfectly scoped, deletes itself with the component.
**Trade-offs:** invisible to siblings.
**Use for:** open/closed, hover, input values, anything one component cares about.

### 2. Lifted state + props

Move the value to the closest common parent and pass it down.

**Advantages:** explicit data flow you can read off the JSX; trivially testable.
**Trade-offs:** **prop drilling** once the distance exceeds ~3 levels.
**Use for:** state shared between a handful of nearby components.

> Before reaching for a global store, try **composition** instead: pass the rendered element down
> rather than the data it needs. Half of all prop drilling disappears with `children`.

### 3. Context (+ reducer)

```jsx
const ThemeContext = createContext(null);
// consumers: const theme = useContext(ThemeContext)
```

**Advantages:** removes drilling; built in; ideal for rarely-changing, app-wide values.
**Trade-offs:** **every consumer re-renders on any change** — Context has no selectors. Putting
fast-changing state in one big context is a classic performance bug.
**Use for:** theme, locale, current user, feature flags.
**Mitigation:** split into several small contexts (`UserContext`, `ThemeContext`), and separate
*state* from *dispatch* so action-only consumers never re-render.

### 4. External store — Flux / Redux style

A single store, changed only by dispatched actions, read via selectors.

```js
dispatch(cartSlice.actions.addItem({ id, qty }));   // intent
const total = useSelector(selectCartTotal);         // subscription
```

| Advantages | Trade-offs |
| :--- | :--- |
| One inspectable source of truth; time-travel devtools | Boilerplate, even with Redux Toolkit |
| Every change is a named, loggable event — great for audit | Tempts you to cache server data badly |
| Middleware hooks for logging, analytics, persistence | Indirection: "where does this change?" is 3 files away |
| Selectors give fine-grained re-render control | Overkill for small apps |

**Use for:** complex, interdependent client state with real invariants — a form builder, a
spreadsheet, a checkout flow with 8 coupled steps, an app needing undo/redo.

### 5. Atomic stores — Zustand / Jotai / Recoil-style

Many small independent pieces of state, each subscribed to directly.

```js
const useCart = create((set) => ({
  items: [],
  add: (item) => set((s) => ({ items: [...s.items, item] })),
}));

// component re-renders only when the count changes
const count = useCart((s) => s.items.length);
```

| Advantages | Trade-offs |
| :--- | :--- |
| Minimal boilerplate; no providers needed | Easy to scatter state into 40 uncoordinated atoms |
| Precise subscriptions → fewer re-renders by default | Weaker "every change is an event" story |
| Usable outside React (in plain functions, tests) | Cross-atom invariants are on you to maintain |

**Use for:** most modern apps that need *some* global client state but don't need Redux's audit
trail. This is the pragmatic default in 2026.

### 6. State machines — XState

Model states and legal transitions explicitly; illegal transitions become impossible.

```
idle → submitting → success
              ↓
            error → submitting
```

| Advantages | Trade-offs |
| :--- | :--- |
| Eliminates impossible states (`loading && error`) | Real learning curve; verbose for simple cases |
| The diagram *is* the spec — reviewable by non-devs | Overkill for a login form |
| Handles cancellation, retries, timeouts rigorously | Fewer engineers know it |

**Use for:** multi-step wizards, video/media players, payment and KYC flows, anything where a wrong
transition means money or safety.

### 7. URL as state

The most underrated store in the browser.

```
/search?q=shoes&size=42&page=3&sort=price
```

**Advantages:** shareable, bookmarkable, survives reload, back button works, zero library.
**Trade-offs:** strings only, length limits, and it's public — never put secrets there.
**Use for:** filters, pagination, sort order, selected tab, open detail panel. If a user could
reasonably want to send someone "this exact view", it belongs in the URL.

---

## Where each kind of state actually belongs

| State | Correct home | Why |
| :--- | :--- | :--- |
| Search filters, page number | URL | Shareable, back button |
| Fetched list of orders | Server-state library cache | Needs staleness + refetch |
| Auth token | httpOnly cookie (not JS state) | XSS safety |
| Current theme | Context + `localStorage` | Rare change, app-wide |
| Is the dropdown open | `useState` | Nobody else cares |
| Multi-step form draft | Zustand or reducer (+ storage) | Coupled fields, must survive navigation |
| Checkout step machine | State machine | Illegal transitions cost money |
| Optimistic cart | Server-state cache + optimistic update | Server owns it; UI must feel instant |

---

## Server state, concretely

```jsx
const { data, isPending, error } = useQuery({
  queryKey: ['orders', { status }],
  queryFn: () => api.getOrders({ status }),
  staleTime: 30_000,          // don't refetch for 30s
});
```

What you get for free, and would otherwise hand-roll in every component: request deduplication,
caching by key, background revalidation, retries with backoff, `isPending`/`error` states, cache
invalidation after mutations, and optimistic updates.

**The mental model shift:** you are not *storing* server data, you are *caching* it. A cache needs
a key and a staleness policy — and those two decisions are the whole design.

---

## Anti-patterns

| Anti-pattern | Why it hurts | Fix |
| :--- | :--- | :--- |
| API responses in Redux | You wrote a cache without invalidation | Server-state library |
| One giant context for everything | Whole app re-renders on any change | Split contexts, or use an atomic store |
| Deriving state into more state | Two sources of truth that drift | Compute during render / with a selector |
| `useEffect` to sync A into B | Extra render, race conditions, loops | Derive it, or lift the source |
| Global store for one modal | Global blast radius for local concern | `useState` in the owner component |
| Filters in state, not the URL | Refresh loses the view; can't share links | URL params |

The `useEffect`-to-sync one is worth internalising: **if a value can be computed from other values,
it is not state.** Compute it.

---

## Exercises

**1.** A dashboard has 6 charts, each with its own `useState` + `useEffect` + `fetch`. Switching a
global date range refetches all 6, twice, and two charts briefly show old data. Diagnose and fix.

<details><summary>Answer</summary>

Server state is being managed manually. Double fetches come from effects firing on mount *and* on
the date change; the stale flash comes from each chart keeping its own copy with no invalidation.

Fix: date range → **URL** (shareable, survives reload). Chart data → `useQuery` with
`queryKey: ['chart', id, range]`. Changing the range changes the keys, so all six refetch once,
deduplicated, with cached previous data available. Delete every `useEffect`.
</details>

**2.** Theme changes cause every component to re-render. Why, and what are two fixes?

<details><summary>Answer</summary>

Context has no selector mechanism — all consumers re-render whenever the provider value changes, and
if the value is an object literal it changes identity every render.

Fixes: (a) memoise the provider value and split state from dispatch into two contexts so
setter-only consumers don't subscribe to the value; (b) skip re-rendering entirely — set a
`data-theme` attribute on `<html>` and let CSS variables do the work, so zero React components
re-render on theme change. (b) is usually the right answer for theming.
</details>

**3.** Which of these belong in a global store? (a) sidebar collapsed (b) selected rows in a table
(c) unread notification count (d) current search query

<details><summary>Answer</summary>

(a) Global-ish — persist in `localStorage` + a tiny atom; many components read it.
(b) Local to the table component. Lift only if a toolbar outside the table acts on the selection.
(c) Server state — it comes from the backend and changes without you. Cache it, poll or subscribe.
(d) URL.

Note that only one of the four wants a traditional global store, and even that one is a single atom.
</details>

---

**Next:** [04 — Component Patterns](04_Component_Patterns.md)
