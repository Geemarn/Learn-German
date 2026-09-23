# 01 — Rendering Architectures

> **The question this answers:** where is the HTML produced, and when?

There are only three places HTML can be built: at **build time**, on a **server at request time**,
or in the **browser**. Everything else is a combination of those three.

---

## The vocabulary you need first

| Term | Meaning |
| :--- | :--- |
| **TTFB** | Time to first byte — how fast the server starts responding |
| **FCP** | First contentful paint — when the user sees *something* |
| **TTI** | Time to interactive — when clicks actually work |
| **Hydration** | Attaching JavaScript event handlers to server-produced HTML |
| **Waterfall** | Request A must finish before request B can start — the main cause of slowness |

The gap between FCP and TTI is where users rage-click. Every rendering architecture below is
fundamentally an attempt to shrink that gap.

---

## 1. CSR — Client-Side Rendering

The server sends a near-empty HTML file plus a JavaScript bundle. The browser builds everything.

```html
<!-- what the server actually sends -->
<div id="root"></div>
<script src="/bundle.js"></script>
```

**How it works:** download HTML → download JS → run JS → fetch data → render. Four sequential
steps before the user sees content.

| Advantages | Trade-offs |
| :--- | :--- |
| Dead-simple hosting (any static file host / CDN) | Slow first paint; blank screen while JS loads |
| Instant navigation after first load | Poor SEO without extra work |
| Clean split: API + client, no shared runtime | Whole app ships to every user |
| No server to scale or keep warm | Bad on low-end phones and slow networks |

**Use it when:** authenticated internal tools, dashboards, admin panels, editors, anything behind a
login where SEO is irrelevant and users keep the tab open for a long time.

**Avoid it when:** your traffic comes from search or social links, or your users are on mobile data.

---

## 2. SSG — Static Site Generation

HTML is produced once at build time and served as files from a CDN.

**How it works:** `build` renders every page to HTML → files land on a CDN → the user gets finished
HTML on the first byte.

| Advantages | Trade-offs |
| :--- | :--- |
| Fastest possible TTFB and FCP | Content is frozen until the next build |
| Nearly free hosting, trivially cacheable | Build time grows with page count (10k+ pages hurts) |
| Extremely resilient — nothing to crash | Cannot personalise per user |
| Perfect SEO | Every content edit needs a deploy |

**Use it when:** documentation, marketing sites, blogs, changelogs, landing pages — content that
changes on a human schedule, not a machine one.

**Avoid it when:** prices, stock levels, or user-specific content appear on the page.

---

## 3. SSR — Server-Side Rendering

HTML is built per request on a server, then hydrated in the browser.

| Advantages | Trade-offs |
| :--- | :--- |
| Fresh data *and* complete HTML on first byte | You now own a server: scaling, cold starts, cost |
| Good SEO and social previews | Slower TTFB than static (server has to think) |
| Can personalise (cookies, geo, A/B tests) | Code must run in two environments — subtle bugs |
| Secrets stay server-side | Hydration cost still hits the client |

**Use it when:** e-commerce product pages, news, marketplaces, any page that must be both indexable
and current.

**The classic SSR trap:** the page *looks* ready but is not interactive yet, because hydration
hasn't finished. Users click and nothing happens. Measure TTI, not just FCP.

---

## 4. ISR / Revalidation — Static with an expiry date

Static pages that are regenerated in the background on a schedule or on demand.

```js
// Next.js App Router: static, refreshed at most once a minute
export const revalidate = 60;
```

**How it works:** serve the cached HTML instantly; if it is older than the window, rebuild it in the
background and serve the fresh copy to the *next* visitor.

| Advantages | Trade-offs |
| :--- | :--- |
| Static speed with acceptable freshness | Someone always sees slightly stale content |
| Build time stays flat regardless of page count | Cache invalidation becomes a real design problem |
| Cheap: one render serves thousands of hits | Hard to reason about "which version am I seeing?" |

**Use it when:** large catalogues, product listings, content sites where "up to a minute old" is
fine. This is the best default for most public content-heavy sites.

---

## 5. Streaming SSR + React Server Components

The server sends HTML in chunks as it becomes ready, and some components **never ship JavaScript at
all** because they only ever render on the server.

```jsx
// Server Component — runs on the server, zero bytes shipped to the browser
async function ProductPage({ id }) {
  const product = await db.product.find(id);   // no API layer needed
  return (
    <>
      <ProductHeader product={product} />      {/* static, no JS */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews id={id} />                    {/* streams in later */}
      </Suspense>
      <AddToCartButton id={id} />              {/* 'use client' — needs JS */}
    </>
  );
}
```

| Advantages | Trade-offs |
| :--- | :--- |
| Shell paints immediately; slow data never blocks it | Genuinely new mental model; easy to misuse |
| Big bundle reductions — non-interactive UI ships no JS | Strong framework coupling (Next.js, Remix, etc.) |
| Data fetching lives next to the component, no waterfall | Server/client boundary errors are confusing |
| Secrets and heavy deps stay on the server | Ecosystem libraries still catching up |

**Use it when:** starting a new full-stack product where both SEO and rich interaction matter, and
you accept framework lock-in in exchange for large performance wins.

**The key idea to internalise:** interactivity becomes **opt-in** (`'use client'`) instead of
default. That inversion is the whole performance story.

---

## 6. Islands Architecture / Partial Hydration

The page is static HTML. Only small, explicitly marked "islands" get JavaScript.

```astro
<!-- Astro: static by default -->
<ArticleBody />                       <!-- 0 KB JS -->
<CommentBox client:visible />         <!-- hydrates when scrolled into view -->
```

| Advantages | Trade-offs |
| :--- | :--- |
| Often 90 %+ less JavaScript than an SPA | Cross-island shared state is awkward |
| Excellent Core Web Vitals, great on cheap phones | Not suited to app-like, stateful UI |
| Can mix React + Vue + Svelte islands on one page | Full-page client-side navigation needs extra work |

**Use it when:** content-first sites with a few interactive widgets — documentation with a search
box, a blog with a newsletter form, marketing with a pricing calculator.

---

## Side-by-side summary

| | CSR | SSG | SSR | ISR | RSC + streaming | Islands |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| HTML built | browser | build | request | build + refresh | request, streamed | build |
| First paint | slow | fastest | fast | fastest | fast | fastest |
| Data freshness | live | frozen | live | windowed | live | frozen |
| JS shipped | all | some | all | some | minimal | almost none |
| SEO | poor | great | great | great | great | great |
| Server needed | no | no | yes | yes (light) | yes | no |
| Personalisation | yes | no | yes | limited | yes | limited |

---

## You can mix these — and you should

The real answer for most products is per-route:

| Route | Strategy | Why |
| :--- | :--- | :--- |
| `/` , `/pricing` | SSG | Never changes per user, must be instant |
| `/blog/[slug]` | SSG or ISR | Content, edited occasionally |
| `/products/[id]` | ISR + client-side price/stock | Indexable, mostly static, one live field |
| `/checkout` | SSR | Personal, must be current, not indexed |
| `/dashboard/*` | CSR | Behind login, highly interactive |

If someone asks "should we use SSR or CSR?", the professional answer is "per route, here's the map".

---

## Exercises

**1.** A recipe site with 50 000 recipes and ~20 edits per day. Which strategy?

<details><summary>Answer</summary>

**ISR.** SSG would mean a 50 000-page build for 20 changes — build times measured in hours. SSR
would pay a server render for content that is identical for everyone. ISR with on-demand
revalidation on publish gives static speed, flat build times, and near-instant updates.
</details>

**2.** A bank's internal fraud-review console. 400 users, all logged in, heavy tables and keyboard
shortcuts.

<details><summary>Answer</summary>

**CSR.** No SEO value, small known audience on good hardware, and long sessions where instant
client-side navigation matters far more than first paint. Adding SSR here buys nothing and costs
you a server plus dual-environment bugs.
</details>

**3.** A product page currently does SSR, but TTI is 4.2 s because a 900 KB bundle must hydrate.
Name two structural fixes.

<details><summary>Answer</summary>

(a) Move to **Server Components / islands** so the description, images, and specs ship no JS, and
only the cart button and gallery hydrate. (b) **Stream** the reviews section behind `Suspense` so
the slow query stops blocking the shell. Code-splitting alone treats the symptom; both of these
reduce how much must hydrate at all.
</details>

---

**Next:** [02 — Code Organisation](02_Code_Organization.md)
