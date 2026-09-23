# 04 — Component Patterns

> **The question this answers:** how do I make a component reusable without turning it into a
> 30-prop monster?

Every pattern here exists to solve the same failure: a component that was reused three times and now
has `variant`, `showHeader`, `hideFooter`, `isCompact`, `altLayout`, and `specialCaseForBilling`.

---

## The core distinction: logic vs presentation

Reusable **behaviour** and reusable **appearance** are different problems and need different tools.
Mixing them is why components ossify.

| You want to reuse | Use |
| :--- | :--- |
| Behaviour (fetching, keyboard nav, validation) | A custom hook |
| Appearance (spacing, colour, typography) | A presentational component |
| Behaviour with *caller-supplied* appearance | A headless component |
| Structure with flexible slots | Compound components |

---

## 1. Presentational / Container split

The container knows *where data comes from*; the presentational component knows *how it looks*.

```jsx
// container: data, no markup decisions
function UserProfilePage({ id }) {
  const { data, isPending } = useQuery({ queryKey: ['user', id], queryFn: () => api.user(id) });
  if (isPending) return <ProfileSkeleton />;
  return <UserProfile user={data} onSave={...} />;
}

// presentational: props in, JSX out. No fetching, no store access.
function UserProfile({ user, onSave }) { /* ... */ }
```

| Advantages | Trade-offs |
| :--- | :--- |
| Presentational parts are trivial to test and to Storybook | Doubles the file count |
| Designers can iterate on UI without touching data code | Can feel like ceremony for one-off screens |
| Swapping REST → GraphQL touches only containers | Hooks already give you most of this benefit |

**Modern form:** the split survives, but as **"components that call hooks" vs "components that only
take props"** rather than two literal classes. Keep the discipline, skip the folder names.

---

## 2. Custom hooks — the default composition tool

```js
function useDebouncedValue(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(t);
  }, [value, delay]);
  return debounced;
}
```

| Advantages | Trade-offs |
| :--- | :--- |
| Reuse logic with zero impact on markup | Only callable from components/other hooks |
| Composable: hooks call hooks | Hidden state — two callers get separate instances |
| Testable in isolation | Easy to create a 200-line "god hook" |

**Use for:** basically all logic reuse. This replaced higher-order components and render props for
most purposes.

**When a hook grows past ~60 lines**, split it by concern rather than adding parameters: `useTable`
becomes `useSorting` + `usePagination` + `useSelection`, composed by `useTable`.

---

## 3. Compound components

Related components that share implicit state through context, assembled by the caller.

```jsx
<Tabs defaultValue="billing">
  <Tabs.List>
    <Tabs.Trigger value="billing">Billing</Tabs.Trigger>
    <Tabs.Trigger value="team">Team</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Panel value="billing"><BillingSettings /></Tabs.Panel>
</Tabs>
```

Compare against the alternative API: `<Tabs items={[...]} renderPanel={...} activeClass="..." />`.

| Advantages | Trade-offs |
| :--- | :--- |
| Caller controls structure and order — no prop explosion | More implementation complexity (context plumbing) |
| Reads like HTML; self-documenting | Callers can nest it wrongly; needs runtime guards |
| New variations need no changes to the component | Slightly harder to type strictly |

**Use for:** anything with variable internal layout — tabs, accordions, menus, selects, modals,
data tables, form fields.

---

## 4. Headless / behaviour-only components

Ship the logic, accessibility, and keyboard handling; ship **no** styles. This is how Radix UI,
Headless UI, TanStack Table, and React Hook Form work.

```jsx
// all ARIA, focus management and keyboard nav — your classes
<Popover.Root>
  <Popover.Trigger className="btn-primary">Filters</Popover.Trigger>
  <Popover.Content className="card p-4">…</Popover.Content>
</Popover.Root>
```

| Advantages | Trade-offs |
| :--- | :--- |
| Accessibility solved by specialists, once | You must supply every style |
| Total visual freedom — fits any brand | More initial wiring than a styled kit |
| No fighting a library's CSS specificity | Another dependency to track |

**Use for:** building a design system on top of correct, accessible primitives. Getting combobox or
dialog focus-trapping right yourself is weeks of work you will get wrong.

---

## 5. Composition over configuration

The single highest-leverage habit in this lesson. When a component needs flexibility, accept
**elements**, not more booleans.

```jsx
// configuration: every new need adds a prop
<Card title="Sales" showBadge badgeColor="green" footerButtonLabel="Export" hideDivider />

// composition: every new need is the caller's problem
<Card>
  <Card.Header>
    Sales <Badge tone="success">Live</Badge>
  </Card.Header>
  <Card.Body><Chart /></Card.Body>
  <Card.Footer><Button>Export</Button></Card.Footer>
</Card>
```

| Advantages | Trade-offs |
| :--- | :--- |
| Prop count stops growing with feature count | Less enforced consistency — callers can make ugly things |
| Card never changes again; callers change | Slightly more verbose at each call site |
| Fixes prop drilling: pass elements, not data | Needs a documented set of slots |

**Rule of thumb:** a boolean prop that only controls *whether some markup appears* should be a slot
instead. Three or more such booleans means you needed composition a while ago.

---

## 6. Patterns you will meet in older code

| Pattern | Looks like | Status |
| :--- | :--- | :--- |
| **HOC** | `withRouter(withAuth(Component))` | Superseded by hooks — wrapper hell, opaque prop origins |
| **Render props** | `<Fetch render={(d) => …} />` | Superseded by hooks, except for genuinely render-shaped APIs |
| **Mixins** | `createClass({ mixins: [...] })` | Dead |

Render props are still the right call in one situation: when the *caller must decide the markup for
each item*, e.g. virtualised lists (`renderRow`) or table cell renderers.

---

## Anti-patterns

| Anti-pattern | Symptom | Fix |
| :--- | :--- | :--- |
| Boolean explosion | 8 `is*`/`show*` props | Compound components / slots |
| God component | 600 lines, fetch + logic + markup | Extract hook, split by section |
| Prop drilling 5 levels | Middle components pass props they ignore | Pass elements via `children`, or context |
| Leaky presentational component | A "dumb" component calls `useQuery` | Move the call to the container |
| `variant` doing two jobs | `variant="danger-large-inline"` | Separate `tone`, `size`, `layout` axes |
| Premature abstraction | A shared component with three mutually exclusive modes | Duplicate it; extract only the third time |

On that last one: **two similar components are cheaper than one wrong abstraction.** Wait for the
third use before generalising — by then you can see what actually varies.

---

## Exercises

**1.** A `<Modal>` has grown these props: `title`, `subtitle`, `showClose`, `footerButtons`,
`hideOverlay`, `bodyPadding`, `headerIcon`. Redesign the API.

<details><summary>Answer</summary>

Compound components with slots:

```jsx
<Modal onClose={close}>
  <Modal.Overlay />                      {/* omit it instead of hideOverlay */}
  <Modal.Header icon={<AlertIcon />}>
    Delete project
    <Modal.Close />
  </Modal.Header>
  <Modal.Body>This cannot be undone.</Modal.Body>
  <Modal.Footer>
    <Button onClick={close}>Cancel</Button>
    <Button tone="danger" onClick={del}>Delete</Button>
  </Modal.Footer>
</Modal>
```

Every boolean that hid markup became "don't render that child". `footerButtons` — a data array that
would need its own mini-schema for labels, tones, and handlers — becomes plain JSX. `Modal` now
needs no further changes as new dialogs appear.
</details>

**2.** You need an accessible multi-select combobox with async search. Build, style a library, or
use headless?

<details><summary>Answer</summary>

**Headless** (Radix/Downshift/Ark) + your own styles. Combobox accessibility is among the hardest
widgets to get right: `aria-activedescendant`, type-ahead, focus return, virtual cursor behaviour
in screen readers, touch handling. Building it costs weeks and will still fail an audit. A fully
styled kit gets you running fastest but you will fight its CSS forever once brand requirements
arrive. Headless splits the difference correctly: borrowed correctness, owned appearance.
</details>

**3.** Two teams need a `UserCard`: one shows role + last login, the other shows company + plan.
Someone proposes `<UserCard mode="admin" | "billing">`. Better options?

<details><summary>Answer</summary>

`mode` is a switch over two unrelated layouts — it will grow to five and each branch will get its
own special-case props.

Best: extract shared presentation primitives (`<Avatar>`, `<UserName>`, `<CardShell>`) and let each
team compose `AdminUserCard` and `BillingUserCard`. Duplicated *arrangement* is cheap; a component
with mutually exclusive modes is expensive. Middle ground if you want one component: slots —
`<UserCard user={u}><UserCard.Meta>…</UserCard.Meta></UserCard>`.
</details>

---

**Next:** [05 — Scale & Micro-Frontends](05_Scale_And_Micro_Frontends.md)
