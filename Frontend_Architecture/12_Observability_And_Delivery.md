# 12 — Observability & Delivery

> **The question this answers:** it's broken for someone I can't see, on a device I don't own. Now
> what?

The frontend is the only part of your system that runs on hardware you don't control, on networks
you can't test, in browsers you've never opened. Backend observability is a nice-to-have; frontend
observability is the **only** way you find out what actually happened.

---

## The four signals

| Signal | Answers | Tooling |
| :--- | :--- | :--- |
| **Errors** | What broke, for whom, in which release? | Sentry, Rollbar |
| **Traces / spans** | Where did the time go across the request? | Sentry tracing, OpenTelemetry |
| **RUM / vitals** | Is it slow for real users, and which ones? | web-vitals + your analytics |
| **Logs** | What was the app doing just before it broke? | Structured logs shipped to one place |

You need all four eventually. Start with errors — highest value for the least setup.

---

## Error tracking

Three configuration details separate useful error tracking from noise:

| Requirement | Why it's non-negotiable |
| :--- | :--- |
| **Source maps uploaded per build** | Without them, every stack trace is minified garbage |
| **Release + commit tagged** | "Started 20 minutes ago, in this deploy" is the whole diagnosis |
| **User/tenant context attached** | Distinguishes "one weird user" from "everyone on Safari 16" |

```js
// instrumentation-client.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NEXT_PUBLIC_ENV,   // dev | test | prod, kept separate
  release: process.env.NEXT_PUBLIC_COMMIT_SHA,
  tracesSampleRate: 0.1,                      // sample traces; keep 100% of errors
  _experiments: { enableLogs: true },
});
```

Catch exceptions where you already handle failure:

```js
try {
  await submitInvoice(draft);
} catch (error) {
  Sentry.captureException(error);             // keep the stack + context
  showToast('Could not submit. Please retry.');
}
```

Then wrap meaningful work in spans, so slow paths are attributable rather than anecdotal:

```js
async function fetchInvoices(tenantId) {
  return Sentry.startSpan(
    { op: 'http.client', name: 'GET /api/invoices' },
    async (span) => {
      const res = await fetch(`/api/invoices?tenant=${tenantId}`);
      span.setAttribute('status', res.status);
      return res.json();
    },
  );
}
```

Spans are worth adding to the three or four interactions the business cares about — checkout submit,
search, report generation — not to every function.

### Keeping the signal clean

An error tracker nobody looks at is worse than none, because it creates the illusion of coverage.

| Problem | Fix |
| :--- | :--- |
| Browser extension and bot noise | `ignoreErrors` / `denyUrls`, and filter non-app frames |
| `ResizeObserver loop limit exceeded` and friends | Known-benign allowlist |
| One bug producing 40 000 events | Group by fingerprint, alert on *new* issues and spikes |
| Alert fatigue | Alert on new issues, error-rate spikes, and crash-free-session drops — nothing else |

Track **crash-free sessions** as your headline reliability number. Raw error counts scale with
traffic; the percentage of users who had a good session is what you actually care about.

### Errors need boundaries to be catchable

```jsx
<ErrorBoundary fallback={<SectionFailed onRetry={retry} />}>
  <RevenueChart />
</ErrorBoundary>
```

Per-section boundaries, not one at the root. A failing chart should cost the user a chart, not the
page ([lesson 06](06_Data_Layer.md)).

---

## Structured logs

Log **events with fields**, not sentences. Sentences can't be queried.

```js
const { logger } = Sentry;

logger.info('Invoice submitted', { invoiceId, tenantId, amount });
logger.warn('Rate limit reached for endpoint', { endpoint: '/api/invoices', retryAfter });
logger.error('Payment authorisation failed', { orderId, provider, code });
```

| Advantages | Trade-offs |
| :--- | :--- |
| Queryable and aggregatable: "all failures for tenant X" | Volume costs money; needs sampling |
| Fields correlate with traces and errors | Easy to leak PII into a third-party system |
| `logger.fmt` keeps variables structured, not stringified | Requires team convention to stay consistent |

**Two rules:** never log tokens, passwords, or personal data (logs travel further than you think,
and [lesson 11](11_Auth_And_Security.md) applies to them), and log **decisions and outcomes** —
"chose cached response", "retry 2 of 3", "fell back to default plan" — not the fact that a function
started.

---

## Delivery: decoupling deploy from release

This is the architectural idea that makes everything else safe.

```
deploy  = the code is in production
release = users can see it
```

Keep those separate with flags, and shipping stops being frightening.

### Feature flags

```js
if (flags.newCheckout) return <CheckoutV2 />;
return <CheckoutV1 />;
```

| Advantages | Trade-offs |
| :--- | :--- |
| Merge to main daily; release when ready | Every flag is a branch in the code **and** in your tests |
| Instant kill switch — no rollback deploy needed | Stale flags become permanent dead code |
| Percentage rollouts and per-tenant enablement | Combinatorial explosion if flags interact |
| Decouples teams from each other's schedules | Needs a flag service or config pipeline |

**The discipline that makes flags work:** every flag gets an owner and a removal date at creation.
A codebase with 200 forgotten flags has 2^200 theoretical states and no one who understands any of
them. Audit and delete quarterly.

### Rollout strategies

| Strategy | How | Best for |
| :--- | :--- | :--- |
| **Canary** | 1 % → 10 % → 50 % → 100 %, watching metrics | Risky changes with good telemetry |
| **Blue-green** | Two environments, switch traffic atomically | Instant rollback, infra-level changes |
| **Percentage flag** | Same code, enabled for a slice of users | Product experiments, gradual launches |
| **Ring** | Internal → beta users → everyone | Changes needing qualitative feedback |

All of them require the previous section's telemetry. Rolling out to 10 % is only meaningful if you
can tell whether those 10 % are having a worse time.

### Frontend-specific deploy hazards

| Hazard | Mitigation |
| :--- | :--- |
| Users on the old bundle request a deleted chunk | Keep old assets for 24–48 h; never delete on deploy |
| Long-lived tabs break after a deploy | Version check → "New version available, reload" prompt |
| Service worker serves a stale shell forever | Versioned caches and a tested update path |
| `index.html` cached by a CDN | HTML `no-cache`, hashed assets `immutable, max-age=31536000` |
| API and frontend deploy out of sync | Backward-compatible API changes; expand then contract |

The stale-chunk one bites nearly every team exactly once, and it presents as random `ChunkLoadError`
reports from users you can't reproduce. Catch it by handling that error class with a reload prompt.

---

## Anti-patterns

| Anti-pattern | Why it hurts |
| :--- | :--- |
| No source maps in production | Every stack trace is unreadable; you debug by guessing |
| Untagged releases | Can't tell whether your deploy caused the spike |
| Alerting on every error | Fatigue; the real incident scrolls past unnoticed |
| `console.log` as the observability strategy | You can't see the user's console |
| Flags with no expiry | Permanent dead branches nobody dares delete |
| Deploying with no rollback path | The only option under pressure is a panicked fix-forward |
| Mixing environments in one project | Dev noise buries production signal |
| Logging PII to a third party | A compliance incident waiting to be discovered |

---

## Exercises

**1.** Users tweet that checkout is broken. Your error tracker is quiet. Where do you look?

<details><summary>Answer</summary>

A silent tracker during a real outage usually means the failure happens somewhere it can't report:

- **The bundle never loaded** — a failed deploy, a bad CDN path, or a CSP violation blocking the
  script. No JS means no error reporting. Check the CDN, HTML response, and CSP reports.
- **It's not an exception** — the API returns 200 with an unexpected shape, or a failed request is
  caught and swallowed by a `catch` that only shows a toast. Handled errors need explicit
  `captureException`.
- **Reports are being filtered** — over-aggressive `ignoreErrors`, sampling set too low, or
  ad-blockers dropping the tracker's requests.
- **The failure is server-side** — check the backend and BFF, not the browser.

Prevention: a synthetic checkout journey running every few minutes against production, and alerting
on *business* metrics (orders per minute falling off a cliff) rather than only on errors. Absence of
errors is not evidence of health.
</details>

**2.** A team wants to hold a risky redesign on a long-lived branch for six weeks, then deploy it
all at once. Counter-proposal?

<details><summary>Answer</summary>

Six-week branches produce brutal merge conflicts, a single terrifying deploy, and no way to isolate
which of hundreds of changes broke something.

Instead: merge to main daily behind a `redesign` flag, off by default. The code is continuously
integrated and tested, but invisible. Then release progressively — internal users, 1 %, 10 %, 100 %
— watching crash-free sessions and Core Web Vitals at each step. Any problem is a flag flip away
from resolved, with no rollback deploy.

Cost to acknowledge: both code paths exist for six weeks, so tests must cover both, and the flag
must be deleted the week after full rollout. That's real work, but far less than a six-week merge.
</details>

**3.** After every deploy you get a burst of `ChunkLoadError` from users you can't reproduce.
Diagnose.

<details><summary>Answer</summary>

Users with the page already open are running the **previous** bundle. When they navigate, it
requests a lazily-loaded chunk by its old hashed filename — and your deploy replaced or deleted it.
Unreproducible locally because you always load fresh.

Two fixes, apply both: (a) **retain old assets** for 24–48 hours instead of purging on deploy — the
old chunk stays fetchable; (b) detect the error class and prompt a reload ("A new version is
available"), optionally with a build-version poll so long-lived tabs update gracefully.

Also verify caching headers: `index.html` must be `no-cache` while hashed assets are `immutable`. If
the HTML is cached, users receive an old document pointing at assets that no longer exist.
</details>

---

**Next:** Part III begins with [13 — Flexbox](13_Flexbox.md) · **Index:** [Course index](README.md)
