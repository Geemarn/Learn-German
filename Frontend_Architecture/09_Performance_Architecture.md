# 09 — Performance Architecture

> **The question this answers:** why is it slow, and which of the ten possible causes is it *this*
> time?

Performance is not a task you do at the end. It is a set of structural decisions plus one habit:
**measure before you change anything.** Most performance work fails because it optimises the wrong
layer with great enthusiasm.

---

## The three metrics that matter

| Metric | Measures | Good | Usually caused by |
| :--- | :--- | :--- | :--- |
| **LCP** | Time until the largest element paints | < 2.5 s | Unoptimised images, slow TTFB, render-blocking JS/CSS |
| **INP** | Responsiveness to interaction | < 200 ms | Long tasks, over-rendering, heavy event handlers |
| **CLS** | Visual stability | < 0.1 | Images without dimensions, late fonts, injected banners |

Learn which cause maps to which metric. It turns "the site feels slow" into a specific
investigation instead of a guessing game.

---

## Start here: the diagnosis order

Work top to bottom. Each step is cheaper than the one below it, and steps 1–3 explain most problems.

1. **Images** — are they sized, modern format, lazy below the fold, and dimensioned to prevent shift?
2. **Third-party scripts** — tag managers, chat widgets, analytics, A/B tools. Frequently larger
   than your entire app.
3. **Bundle composition** — run the analyser. Look for one accidental giant import.
4. **Network waterfall** — staircase shape means sequential requests ([lesson 06](06_Data_Layer.md)).
5. **Hydration / JS execution** — only now is the rendering architecture at fault.
6. **Render performance** — re-render storms, unvirtualised lists.

Switching frameworks is never step 1. It carries every problem above along with it.

---

## Bundle architecture

### Route-level code splitting — do this always

```jsx
const Reports = lazy(() => import('./features/reports'));
```

Each route becomes its own chunk, so visiting the dashboard doesn't download the reports module.
This is the single highest-value split and it costs one line per route.

### Component-level splitting — for heavy, conditional UI

```jsx
const Editor = lazy(() => import('./RichTextEditor'));   // 300 KB, opens on click
```

Good candidates: rich text editors, charting libraries, PDF viewers, map widgets, emoji pickers,
date pickers, anything inside a modal.

### The barrel-file trap

```js
// shared/ui/index.ts
export * from './Button';
export * from './DataTable';   // 90 KB
export * from './Chart';       // 120 KB
```

Importing `Button` from this barrel can pull in the whole file's graph if anything in it has side
effects or your bundler can't prove otherwise. This is one of the most common causes of mysteriously
large chunks.

**Fixes:** import from the source path in hot code, keep barrels free of side effects, set
`"sideEffects": false` in `package.json`, and verify with the analyser rather than trusting theory.

### Dependency choices, ranked by typical damage

| Instead of | Use | Saves |
| :--- | :--- | :--- |
| `moment` | `date-fns` (per-function imports) or `Intl` | ~200 KB |
| `lodash` | `lodash-es` per function, or plain JS | ~70 KB |
| Full icon package | Per-icon imports or inline SVG | 100 KB+ |
| A charting library on every page | Lazy-load per chart page | 120 KB+ |

**Enforce it in CI**, because bundles only ever grow without a gate:

```yaml
# size-limit / bundlesize: fail the PR, don't just warn
- path: 'dist/assets/index-*.js'
  limit: '170 kB'
```

A budget that only warns is a budget nobody reads.

---

## Images and fonts

These two are usually the whole LCP and CLS story.

| Concern | Do this |
| :--- | :--- |
| Format | AVIF/WebP with a fallback |
| Sizing | `srcset` + `sizes` so phones don't download desktop images |
| Dimensions | Always set `width`/`height` or `aspect-ratio` — this is the CLS fix |
| Loading | `loading="lazy"` below the fold; **eager + `fetchpriority="high"`** for the LCP image |
| Fonts | `font-display: swap`, preload the one critical weight, subset the character set |
| Font shift | Tune `size-adjust` on the fallback so swapping doesn't reflow text |

Lazy-loading the hero image is a common own goal: it delays the exact element LCP measures.

---

## Render performance

| Problem | Symptom | Fix |
| :--- | :--- | :--- |
| Long list rendered fully | Scroll jank, slow mount | Virtualise (TanStack Virtual) |
| Re-render storms | Typing in one field lags the page | Narrow subscriptions ([lesson 03](03_State_Management.md)) |
| Unstable keys (`key={index}`) | Wrong rows update; inputs lose state | Stable IDs |
| New object/array props each render | Memoisation silently does nothing | `useMemo` the value, or restructure |
| Expensive work in a handler | Poor INP | Defer with `startTransition`, or move to a worker |
| Layout thrash | Jank while scrolling or resizing | Batch DOM reads before writes |

**On memoisation:** `memo`, `useMemo`, and `useCallback` are not free — they cost comparison work
and memory. Apply them where a profile shows a problem, not prophylactically. Fixing *why* the
component re-renders usually beats memoising the fact that it does.

---

## Perceived performance

Often cheaper and more effective than real performance.

| Technique | Effect |
| :--- | :--- |
| Skeletons that match the final layout | Feels like progress; prevents layout shift |
| Prefetch on hover/focus | The page is often already loaded when clicked |
| Optimistic updates | Interaction feels instant ([lesson 06](06_Data_Layer.md)) |
| Streaming the shell | Content appears while slow data is still in flight |
| Keep old data during refetch | No spinner flash between filter changes |

**The inverse trap:** a spinner that shows for 80 ms reads as a flicker and feels *worse* than
showing nothing. Delay spinners by ~200 ms, and once shown, keep them for a minimum duration.

---

## Measurement: lab vs field

| | Lab (Lighthouse, local profiling) | Field / RUM (real users) |
| :--- | :--- | :--- |
| Advantage | Reproducible, debuggable, pre-merge | Tells you the *truth* |
| Trade-off | Your laptop is not your user's phone | Noisy; aggregate, not step-by-step |
| Use for | Catching regressions in CI | Deciding what to fix, and proving it worked |

Track the **p75**, not the average — averages hide the users who are suffering. Segment by device
and country, because one slow market can be invisible in the global number.

Wire it up once:

```js
import { onLCP, onINP, onCLS } from 'web-vitals';
[onLCP, onINP, onCLS].forEach((fn) => fn((m) => analytics.send(m.name, m.value)));
```

---

## Anti-patterns

| Anti-pattern | Why it hurts |
| :--- | :--- |
| Optimising without a profile | You spend a week on 2 % and miss the 40 % |
| No performance budget in CI | Every feature adds weight; nobody is accountable |
| Memoising everything | Added complexity and memory for no measured gain |
| Lazy-loading the hero image | Directly delays LCP |
| Third-party scripts in `<head>`, synchronous | Blocks the page for a chat widget |
| Framework migration as a performance plan | Months of cost; the real causes migrate with you |
| Celebrating a Lighthouse score | Lab number improved; real users unchanged |

---

## Exercises

**1.** Bundle analysis shows a 240 KB chunk on the login page. The team wants to split the login
form. Good idea?

<details><summary>Answer</summary>

Almost certainly not — a login form is a few KB. The 240 KB is something else that got pulled in,
most likely through a barrel file: `shared/ui/index.ts` re-exporting a `DataTable` or chart, or a
top-level import of an icon set or date library.

Open the analyser and find the *actual* module. Fix the import path, not the form. Splitting the
login form would add complexity while leaving the real 235 KB in place.
</details>

**2.** LCP is 1.9 s (good) but users say the site "doesn't respond". Which metric, and what causes
it?

<details><summary>Answer</summary>

**INP.** The page paints fast but the main thread is blocked when users interact.

Likely causes, in order: a large hydration pass still running at first click; a long task in an
event handler (filtering 10 000 rows synchronously, parsing a big JSON payload); an unvirtualised
list re-rendering on every keystroke; or a third-party script occupying the main thread.

Investigate with the Performance panel's long-task markers. Structural fixes: reduce how much
hydrates ([lesson 01](01_Rendering_Architectures.md)), virtualise lists, wrap non-urgent updates in
`startTransition`, and move heavy computation to a web worker.
</details>

**3.** A PR adds a 90 KB dependency for one admin-only screen. Block it?

<details><summary>Answer</summary>

Don't block — scope it. If it's dynamically imported on an admin-only route, no regular user ever
downloads it, and the budget for the main entry chunk is untouched. That is exactly what
route-level splitting is for.

Block it only if it is imported at the top level (entering the shared chunk), if a much smaller
alternative covers the use case, or if it duplicates something already bundled — two date libraries
or two icon sets in one app is the real failure here.
</details>

---

**Next:** [10 — Testing Architecture](10_Testing_Architecture.md)
