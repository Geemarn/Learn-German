# 15 — Layout Recipes

> **The question this answers:** I know flex and grid. What do I actually write for the layout in
> front of me?

Copy-paste-ready patterns, each with the reason it's built that way. All of them are modern CSS —
no floats, no clearfix, no `position: absolute` gymnastics.

---

## 1. App shell — header, sidebar, content, footer

```css
.app {
  display: grid;
  grid-template-columns: 240px 1fr;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header  header"
    "sidebar main"
    "footer  footer";
  min-height: 100dvh;
}
.app__header  { grid-area: header; }
.app__sidebar { grid-area: sidebar; }
.app__main    { grid-area: main; overflow-y: auto; }
.app__footer  { grid-area: footer; }

@media (width < 768px) {
  .app {
    grid-template-columns: 1fr;
    grid-template-areas: "header" "main" "footer";
  }
  .app__sidebar { display: none; }   /* becomes a drawer */
}
```

`1fr` on the middle row pushes the footer to the bottom on short pages with no `position: fixed`.
`100dvh` rather than `100vh` avoids the mobile browser chrome bug where `100vh` overflows the
visible area.

---

## 2. Responsive card grid — no media queries

```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 24px;
}
```

The column count follows the available width automatically. Add a full-width row inside the same
grid whenever you need a section break:

```css
.cards__heading { grid-column: 1 / -1; }
```

---

## 3. Equal-height cards with bottom-aligned actions

The classic problem: cards in a row have different text lengths, and you want the buttons to line
up along the bottom.

```css
.cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(260px, 1fr)); gap: 20px; }

.card {
  display: flex;
  flex-direction: column;   /* grid gives equal height, flex arranges within */
}
.card__body   { flex: 1; }            /* absorbs the slack */
.card__footer { margin-top: auto; }   /* belt and braces   */
```

This is the canonical grid-outside, flex-inside composition: grid equalises the heights across the
row, flex distributes space inside each card.

---

## 4. The media object — fixed thumbnail, fluid text

```css
.media { display: flex; gap: 12px; align-items: flex-start; }
.media__figure { flex: 0 0 56px; }     /* never grows, never shrinks */
.media__body   { flex: 1; min-width: 0; }  /* min-width: 0 enables truncation */
```

Without `min-width: 0`, a long unbroken string in the body overflows instead of truncating — see
[lesson 13](13_Flexbox.md).

---

## 5. Centring, four ways

```css
/* Shortest — both axes */
.center { display: grid; place-items: center; }

/* Flex equivalent, when you also need direction control */
.center { display: flex; justify-content: center; align-items: center; }

/* Horizontal only, for a fixed-width block */
.container { max-width: 72ch; margin-inline: auto; }

/* Centre one child inside a grid without touching its siblings */
.child { justify-self: center; align-self: center; }
```

`max-width: 72ch` is worth knowing: `ch` units tie the measure to the font size, which keeps line
length readable across breakpoints without a single media query.

---

## 6. Sidebar that collapses into a stack — no breakpoints

```css
.with-sidebar {
  display: flex;
  flex-wrap: wrap;
  gap: 24px;
}
.with-sidebar__side { flex: 1 1 220px; }        /* ideal 220px         */
.with-sidebar__main { flex: 999 1 60%; }        /* huge grow, 60% min  */
```

When there isn't room for the main area to stay at 60%, the line wraps and the two become a stack.
The absurd `flex-grow: 999` makes the main area claim essentially all the free space while they're
side by side. This is container-driven, so it works inside any column width — something media
queries can't do, since they only know about the viewport.

For the modern equivalent, use container queries:

```css
.panel { container-type: inline-size; }
@container (width > 600px) {
  .panel__layout { display: grid; grid-template-columns: 220px 1fr; }
}
```

---

## 7. Full-bleed sections inside a constrained page

Letting one section break out of a centred column, without wrapper divs:

```css
.prose {
  display: grid;
  grid-template-columns:
    1fr
    min(72ch, 100% - 48px)
    1fr;
}
.prose > * { grid-column: 2; }        /* everything in the readable column */
.prose > .full-bleed { grid-column: 1 / -1; }   /* except these            */
```

`min(72ch, 100% - 48px)` gives a readable measure on wide screens and a 24px gutter on small ones,
in one declaration.

---

## 8. Overlapping elements — same grid cell

Grid can stack items, which replaces most `position: absolute` usage:

```css
.hero {
  display: grid;
}
.hero > * {
  grid-area: 1 / 1;      /* every child in the same cell */
}
.hero__caption {
  place-self: end start;
  z-index: 1;
}
```

Unlike absolute positioning, the container still sizes itself to the tallest child, so nothing
collapses to zero height.

---

## 9. Sticky table header inside a scroll area

```css
.table-wrap { max-height: 60dvh; overflow: auto; }
thead th {
  position: sticky;
  top: 0;
  background: var(--color-surface);   /* required — sticky cells are transparent */
  z-index: 1;
}
```

The background is not optional: without it, scrolled rows show through the header.

---

## 10. Aspect-ratio media that never causes layout shift

```css
.thumb {
  aspect-ratio: 16 / 9;
  object-fit: cover;
  width: 100%;
}
```

Reserving the space before the image loads is a direct CLS fix
([lesson 09](09_Performance_Architecture.md)). Setting the `width` and `height` attributes in HTML
does the same job and works even before CSS loads — do both.

---

## Debugging layout: a fast checklist

| Symptom | Likely cause |
| :--- | :--- |
| Horizontal scrollbar appears | A `1fr` grid track or flex item that won't shrink → `minmax(0, 1fr)` / `min-width: 0` |
| `justify-content` does nothing | No free space on the main axis, or you needed `align-items` |
| Item won't centre vertically | The container has no height to centre within |
| Gap appears under an image | `display: inline` baseline space → `display: block` |
| Sticky element won't stick | An ancestor has `overflow: hidden` or `auto` |
| `height: 100%` ignored | Every ancestor needs a height; use `dvh` or grid `1fr` instead |
| Grid item escapes its area | `grid-area` name typo — misspelled names silently create a new implicit area |

**Use the browser's layout inspector.** Chrome and Firefox both overlay grid line numbers and flex
alignment guides on hover, which turns most of these into a five-second diagnosis. In Firefox the
grid inspector also shows area names, which catches the last row in that table instantly.

---

## Exercises

**1.** A three-column card grid uses `@media` breakpoints at 600px and 900px. Inside a 500px-wide
dashboard panel, it still renders three cramped columns. Why, and what fixes it?

<details><summary>Answer</summary>

Media queries respond to the **viewport**, not the element. On a 1400px monitor the 900px query
matches, so the grid uses three columns even though its own container is only 500px wide.

Two fixes. The simplest is intrinsic sizing — no queries at all:

```css
grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
```

The explicit option is container queries, when you need different *rules*, not just different
counts:

```css
.panel { container-type: inline-size; }
@container (width > 700px) { .cards { grid-template-columns: repeat(3, 1fr); } }
```
</details>

**2.** A modal is centred with `position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%)`.
What's the modern replacement, and what does it fix?

<details><summary>Answer</summary>

```css
.overlay { position: fixed; inset: 0; display: grid; place-items: center; padding: 24px; }
```

It fixes three things the transform approach gets wrong: a tall modal no longer overflows off the
top of the screen (the grid centres but respects the padding), the `translate` no longer conflicts
with transform-based animations, and half-pixel offsets that blur text disappear.

Better still, use the native `<dialog>` element with `showModal()` — you get focus trapping, an
`::backdrop`, and Escape-to-close for free, which is a meaningful accessibility win over a
hand-built overlay ([lesson 04](04_Component_Patterns.md)).
</details>

**3.** Your card grid looks fine until one card has a long unbroken product code, which widens
every column. Diagnose.

<details><summary>Answer</summary>

`1fr` is `minmax(auto, 1fr)`, so each track's minimum is its content's intrinsic minimum width. The
unbreakable string sets a large minimum for its track, and because all tracks are `1fr` they stay
equal — so every column grows to match.

```css
.cards { grid-template-columns: repeat(auto-fit, minmax(0, 1fr)); }
.card__code { overflow-wrap: anywhere; }   /* or: text-overflow: ellipsis */
```

Fix both sides: `minmax(0, …)` stops the track from being held open, and the wrapping rule decides
what the long string should do instead. Without the second line the text merely overflows its own
card rather than the grid.
</details>

---

**Back to:** [Course index](README.md)
