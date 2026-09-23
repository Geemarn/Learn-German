# 10 — Testing Architecture

> **The question this answers:** which tests give me the confidence to deploy on a Friday, at a
> price I can afford?

Every test costs you twice: once to write, then forever to maintain. A test suite is an architecture
decision, and the wrong shape produces the worst outcome available — slow, flaky, and still missing
real bugs.

---

## The shape: pyramid vs trophy

The classic pyramid (mostly unit tests) was designed for backends with deep logic. Frontends are
mostly **integration**: components, state, and network talking to each other. Bugs live in the
wiring, not in the functions.

```
      /\        E2E          few, critical journeys
     /--\       Integration  ← MOST of your value lives here
    /----\      Component
   /------\     Unit         pure logic only
  ----------    Static       TypeScript + ESLint (free, catches the most)
```

**The "testing trophy":** a fat integration middle, thin unit and E2E ends, on a base of static
analysis. Aim for that.

---

## The levels

### Static analysis — TypeScript strict + ESLint

| Advantages | Trade-offs |
| :--- | :--- |
| Catches whole bug classes at zero marginal cost | Can't catch logic errors |
| Runs in the editor — feedback in milliseconds | Types can lie (`any`, bad casts, unchecked API shapes) |

**The highest-value testing you can adopt**, and most teams under-use it. `strict: true`,
`noUncheckedIndexedAccess`, and validating API responses at the boundary with Zod removes more
production bugs than a hundred unit tests.

### Unit tests — pure functions

```js
expect(calculateVat(100, 'DE')).toBe(19);
```

| Advantages | Trade-offs |
| :--- | :--- |
| Fast, precise, excellent failure messages | Prove nothing about whether the app works |
| Ideal for real domain logic and edge cases | Coupled to structure — refactors break them |

**Use for:** pricing, permissions, date maths, parsers, reducers, state machines. **Don't** unit
test components; that's the next level.

### Component tests — Testing Library

```jsx
render(<LoginForm onSubmit={fn} />);
await user.type(screen.getByLabelText('Email'), 'a@b.com');
await user.click(screen.getByRole('button', { name: 'Sign in' }));
expect(fn).toHaveBeenCalled();
```

| Advantages | Trade-offs |
| :--- | :--- |
| Tests what the user does, so refactors don't break them | jsdom isn't a browser: no layout, no real events |
| Queries by role/label double as accessibility checks | Slower than unit tests |
| Survives internal rewrites of the component | Async behaviour needs care to avoid flake |

**The rule that makes these durable:** query by **role, label, or text** — never by test ID or
class. If you can't find an element the way a user would, that's an accessibility bug your test just
caught.

### Integration tests — features with mocked network (MSW)

Mock at the **network** layer, not the module layer.

```js
server.use(
  http.get('/api/orders', () => HttpResponse.json([{ id: 1, total: 42 }])),
);
// then render the whole feature and assert on what the user sees
```

| Advantages | Trade-offs |
| :--- | :--- |
| Highest confidence per second of runtime | Handlers drift from the real API unless generated |
| Exercises real routing, state, caching, and error paths | More setup than mocking a module |
| Same handlers reusable in dev and in Storybook | Still not the real backend |

**This is where most of your tests should be.** Mocking your own modules with `jest.mock` couples
tests to internal structure and passes happily while production is broken; mocking HTTP keeps every
layer under your control real.

### End-to-end tests — Playwright / Cypress

| Advantages | Trade-offs |
| :--- | :--- |
| The only tests that prove the system actually works | Slow: minutes, not milliseconds |
| Catches build, routing, auth, and CSP problems | Flaky without discipline |
| Real browser: real layout, real events, real cookies | Needs environments, seed data, and maintenance |

**Use for:** 5–15 revenue-critical journeys. Sign up, log in, checkout, the one report the CEO
opens. Not every form.

**Flake control** (without it, the suite gets ignored and then deleted):
- Never `waitForTimeout`. Wait for a visible state: `await expect(el).toBeVisible()`.
- Seed data through the API, not the UI; log in by injecting a session, not by typing.
- Each test independent and parallel-safe — no shared mutable fixtures.
- Quarantine a flaky test immediately, with an owner and a deadline. One flaky test teaches the team
  to ignore red builds.

### Visual regression

| Advantages | Trade-offs |
| :--- | :--- |
| Catches CSS breakage nothing else can see | False positives from fonts, animation, dates |
| Perfect fit for design systems | Cost per snapshot; review fatigue |

**Use for:** design system primitives and a few key pages. Freeze time, disable animation, and mask
dynamic regions or you'll approve diffs without reading them.

### Contract / API type tests

Generate types from OpenAPI or GraphQL and fail the build when the API changes shape. This catches
the most expensive frontend bug — the backend renamed a field — at build time instead of in
production.

---

## Comparison

| | Static | Unit | Component | Integration | E2E | Visual |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Speed | instant | ms | tens of ms | ~100 ms | seconds | seconds |
| Confidence | low | low | medium | **high** | highest | narrow |
| Flakiness | none | none | low | low | **high** | medium |
| Maintenance | none | medium | low | low | high | medium |
| Suggested share | always on | 15 % | 20 % | **50 %** | 10 % | 5 % |

---

## What to test, and what not to

| Test this | Not this |
| :--- | :--- |
| What the user sees and can do | Internal state values |
| Error, empty, and loading states | Implementation details (`instance.state.x`) |
| Business rules and edge cases | Third-party library behaviour |
| Accessibility: roles, labels, focus | Exact CSS class names |
| Bug reproductions (a test per fix) | Trivial getters and pass-through props |
| Permission boundaries in the UI | Snapshots of large component trees |

**Coverage percentage is a terrible target.** It rewards testing trivial code and says nothing about
whether checkout works. Track instead: escaped bugs per release, and whether every fixed bug got a
regression test.

---

## CI architecture

| Concern | Approach |
| :--- | :--- |
| Speed | Run only affected projects (Turborepo/Nx) + remote cache |
| Parallelism | Shard unit/integration tests; run E2E across parallel workers |
| Staging | Static + unit + integration on every PR; full E2E on merge to main |
| Preview | Deploy a preview URL per PR and run smoke E2E against it |
| Flake | Auto-retry **once** and report it — never retry silently three times |

If the PR suite exceeds ~10 minutes, people stop running it locally and start merging hopefully.
Suite speed is a correctness feature.

---

## Anti-patterns

| Anti-pattern | Why it hurts | Fix |
| :--- | :--- | :--- |
| Snapshot tests of whole trees | Break on every change; nobody reads the diff | Assert specific visible behaviour |
| `jest.mock` on your own modules | Passes while the app is broken | Mock HTTP with MSW |
| `getByTestId` everywhere | Skips the accessibility tree; tests nothing real | Query by role/label |
| E2E for every form | 45-minute suite, constant flake | Integration tests + a few E2E journeys |
| 100 % coverage mandate | Tests written for the metric, not the risk | Target critical paths |
| `waitForTimeout(2000)` | Slow *and* flaky | Wait for state |
| No test with a bug fix | The same bug returns in six months | Regression test per fix, always |

---

## Exercises

**1.** A team has 1 200 unit tests, 95 % coverage, and still ships broken checkouts. What's wrong?

<details><summary>Answer</summary>

They're testing units in isolation while their bugs live in the **wiring**: routing, state
synchronisation, cache invalidation, API shape mismatches, and error handling. Mocked-module unit
tests can all pass while no two real pieces fit together.

Fix the shape, not the count: add MSW-based integration tests that render the whole checkout feature
against realistic HTTP responses, plus 2–3 Playwright journeys covering the actual purchase path.
Then delete unit tests that merely assert a component calls a mock. Expect coverage to *drop* while
confidence rises — which is exactly why coverage is a bad target.
</details>

**2.** An E2E suite takes 40 minutes and fails ~20 % of the time for unrelated reasons. Nobody
trusts it. Recovery plan?

<details><summary>Answer</summary>

Restore trust first; a suite people ignore has negative value.

1. Quarantine every flaky test out of the blocking path today, each with an owner and a fix-or-delete
   date. A green build must mean something by tomorrow.
2. Keep only critical journeys as blocking E2E — usually 5–15 of them.
3. Push the rest down to integration tests with MSW, where they run in seconds without flake.
4. Remove the flake causes: no fixed waits, seed via API, log in via injected session, one isolated
   data set per worker.
5. Shard the remainder across parallel workers to get under 10 minutes.
6. Run the full suite on merge to main and nightly, not on every PR push.
</details>

**3.** Where do you test that a viewer-role user can't see the "Delete" button?

<details><summary>Answer</summary>

At two levels, for different reasons.

**Integration test** — render the feature with a viewer-role session and assert the button is
absent. Fast, precise, and covers every role in a handful of cases.

**One E2E test** — verify the real session and route guard actually work end to end, since that path
involves cookies, middleware, and the real API.

And the critical caveat: this is a **UX** test, not a security test. Hiding a button is not
authorisation — the server must reject the delete regardless of what the client renders. That
belongs in a backend test ([lesson 11](11_Auth_And_Security.md)).
</details>

---

**Next:** [11 — Auth & Security](11_Auth_And_Security.md)
