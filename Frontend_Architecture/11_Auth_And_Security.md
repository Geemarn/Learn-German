# 11 — Auth & Security

> **The question this answers:** where do I keep the token, and what can an attacker do with my
> frontend?

One principle governs this entire lesson:

> **The frontend cannot enforce security. It can only present it.**

Everything you hide in the UI is still reachable by anyone with devtools and curl. The UI decides
what is *convenient*; the server decides what is *allowed*. Architectures that forget this produce
beautiful, hackable applications.

---

## Where to store the session

This is the decision people get wrong most often.

| Storage | XSS-safe | CSRF-safe | Verdict |
| :--- | :--- | :--- | :--- |
| `localStorage` | ❌ any script can read it | ✅ | **Avoid** |
| `sessionStorage` | ❌ same problem | ✅ | **Avoid** |
| JS variable / memory | ⚠️ better, lost on reload | ✅ | OK for short-lived access tokens |
| **httpOnly + Secure + SameSite cookie** | ✅ JS cannot read it | needs care | **Default choice** |

```
Set-Cookie: session=…; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=1209600
```

**Why this matters:** with a token in `localStorage`, a single XSS — including one from a compromised
npm dependency — exfiltrates every user's credentials silently. With httpOnly cookies, the same XSS
can still act as the user in that browser session, but cannot steal a portable token. That's the
difference between an incident and a catastrophe.

`SameSite=Lax` blocks most CSRF automatically. Use `Strict` where no cross-site navigation must stay
logged in, and if you need `None` (cross-domain), add explicit CSRF tokens.

**"But my API needs a Bearer header."** Then use a **BFF**: the browser holds only the cookie, the
BFF exchanges it for the Bearer token server-side ([lesson 05](05_Scale_And_Micro_Frontends.md)).
This is the standard pattern for SPAs talking to OAuth-protected APIs.

---

## Session vs JWT

| | Server session (opaque ID) | Self-contained JWT |
| :--- | :--- | :--- |
| Revocation | **Instant** — delete the record | Hard — valid until expiry |
| Server state | Needs a store (Redis/DB) | Stateless, scales trivially |
| Size | Tiny cookie | Larger; grows with claims |
| Freshness | Always current permissions | Claims are stale until refresh |

**Choose sessions** for normal products — the ability to log someone out immediately is worth a Redis
instance. Fired employees and stolen laptops are not hypothetical.

**Choose JWTs** for genuinely stateless, high-scale, service-to-service cases — and then keep access
tokens **short-lived** (5–15 min) with refresh-token rotation, so the revocation window is small.

---

## OAuth / OIDC on the frontend

Use **Authorization Code flow with PKCE**. The implicit flow is deprecated; never use it.

```
app → /authorize?…&code_challenge=…      (redirect to the IdP)
IdP → /callback?code=…                   (back to your app/BFF)
BFF → POST /token (code + verifier)      → tokens stay server-side
BFF → Set-Cookie: session=…              (browser gets only a cookie)
```

| Advantages | Trade-offs |
| :--- | :--- |
| Tokens never touch browser JavaScript | Requires a server component |
| Works with any IdP; SSO for free | Redirect flows are fiddly to debug |
| Central logout and session policy | More moving parts than a login form |

The rule to remember: **tokens should not be readable by your own JavaScript.** If they are, any
script on the page — yours, a dependency's, or an injected one — has them too.

---

## Authorisation in the UI

Two layers, with distinct jobs:

```jsx
// 1. Route guard — a UX affordance, never a security control
if (!session) redirect('/login?next=' + encodeURIComponent(path));

// 2. Capability-driven UI — render from permissions the server sent
{can('invoice:delete') && <DeleteButton />}
```

**Design tip:** have the server return **capabilities**, not roles.

| Server sends | Frontend does | Problem |
| :--- | :--- | :--- |
| `role: 'admin'` | `if (role === 'admin' \|\| role === 'billing_admin')` | Every permission change ships a frontend release |
| `can: ['invoice:delete']` | `if (can('invoice:delete'))` | Policy lives in one place, on the server |

Role checks scattered through components are permission logic duplicated in the least trustworthy
place in your system. Capabilities keep the rules server-side where they're enforced anyway.

---

## The attack surface you actually own

### XSS — the one that matters most

| Vector | Mitigation |
| :--- | :--- |
| `dangerouslySetInnerHTML` / `v-html` | Avoid; if unavoidable, sanitise with DOMPurify **and** allowlist tags |
| User-supplied URLs in `href`/`src` | Reject anything not `https:`/`mailto:` — `javascript:` URLs execute |
| Markdown rendering | Use a sanitising renderer; never `marked` output straight into HTML |
| Third-party scripts | Subresource Integrity + a strict CSP; audit what you embed |
| SVG uploads rendered inline | SVG can carry scripts — sanitise or serve as an image, not inline |

React escapes interpolated text by default. Nearly every real frontend XSS comes from one of the
rows above — the places where you explicitly opted out of that protection.

### Content Security Policy

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{random}';
  object-src 'none';
  base-uri 'self';
  frame-ancestors 'none';
```

| Advantages | Trade-offs |
| :--- | :--- |
| Turns many XSS bugs into blocked, reported events | Inline scripts/styles need nonces or hashes |
| `frame-ancestors` prevents clickjacking | Third-party tools frequently violate it |
| Report-only mode lets you roll out safely | Real work to maintain as dependencies change |

Deploy in `Report-Only` first, watch the reports for a week, then enforce. And avoid
`'unsafe-inline'` in `script-src` — it removes most of the protection you're paying for.

### Supply chain

Your `node_modules` is production code written by strangers.

| Risk | Mitigation |
| :--- | :--- |
| Malicious or hijacked package | Lockfiles, `npm audit` in CI, Dependabot/Renovate, pin exact versions for critical deps |
| Install scripts | `--ignore-scripts` in CI where feasible |
| Typosquatting | Review new dependencies in PR like any other code |
| Compromised CDN script | Subresource Integrity hashes |

A dependency can read anything your JavaScript can read — which is the deepest reason tokens don't
belong in `localStorage`.

---

## Other essentials

| Concern | Approach |
| :--- | :--- |
| Secrets | Never in client bundles. Anything prefixed `NEXT_PUBLIC_`/`VITE_` is public |
| Multi-tenancy | Server scopes every query by tenant; never trust a tenant ID from the client |
| IDOR | Don't rely on unguessable IDs — authorise every request server-side |
| Rate limiting | On the server; client-side throttling is UX only |
| File uploads | Validate type and size server-side; serve user files from a separate origin |
| PII in URLs | Don't — URLs reach analytics, logs, and referrer headers |
| Error messages | Generic to the user, detailed to your logs; never leak stack traces or SQL |
| Dev/test/prod | Separate credentials and IdP tenants per environment; never share a database |

---

## Anti-patterns

| Anti-pattern | Why it's dangerous |
| :--- | :--- |
| JWT in `localStorage` | One XSS or bad dependency exfiltrates every session |
| Hiding a button as the authorisation control | The API call still works from curl |
| Client-side-only route guards on paid features | View source, call the endpoint, get it free |
| Trusting a client-supplied `userId`/`tenantId` | Trivial privilege escalation |
| API keys in the bundle "because the API needs them" | It's published to the world; proxy through a BFF |
| `'unsafe-inline'` in the CSP | Neutralises most of the policy |
| Role strings duplicated across components | Policy drift; every change is a release |
| Verbose errors shown to users | Free reconnaissance for an attacker |

---

## Exercises

**1.** A React SPA stores a JWT in `localStorage` and sends it as a Bearer header. The team says
"we're safe, we sanitise inputs". Respond.

<details><summary>Answer</summary>

Input sanitisation doesn't address the real exposure. The token is readable by **all JavaScript on
the page**, including every transitive npm dependency and any third-party script. One compromised
package — no XSS in your own code required — silently exfiltrates every user's token, and because
JWTs are self-contained, the attacker can use them from anywhere until expiry. Sanitising inputs
covers one of several vectors.

Migration: move to an httpOnly + Secure + SameSite cookie; add a BFF that holds the Bearer token and
proxies API calls; shorten access-token lifetime with refresh rotation; add a CSP in report-only
mode. During transition, accept both cookie and header at the API, then drop header support on a
published date.
</details>

**2.** A pricing page hides the "Export to CSV" button for free-tier users. A user finds the export
endpoint and calls it directly. Whose bug?

<details><summary>Answer</summary>

The backend's — this is the lesson's core principle in practice. The hidden button is a UX
affordance; entitlement must be enforced where the data leaves the system.

Fix: the export endpoint checks the caller's plan and returns 403. Keep the UI hiding too, but
recognise it as presentation. Also worth auditing: the same mistake usually exists on several other
endpoints, since it reflects a belief about where authorisation lives rather than a one-off
oversight. Add a test asserting a free-tier session gets 403 ([lesson 10](10_Testing_Architecture.md)).
</details>

**3.** Product wants a third-party analytics script "just add the snippet". Your CSP is strict.
Evaluate.

<details><summary>Answer</summary>

Don't weaken the CSP with `'unsafe-inline'` — that would trade real protection for one vendor's
convenience.

Options, best to worst: (a) load it from a specific allowlisted host with a **nonce** on the loader
tag and SRI where the vendor supports stable hashes; (b) proxy it through your own domain so it
stays first-party and under your control; (c) a server-side or edge analytics approach with no
client script at all — best for both privacy and performance.

Also ask what it costs beyond security: third-party scripts are a top cause of poor LCP and INP
([lesson 09](09_Performance_Architecture.md)), and this one gets access to everything on the page,
including anything in the DOM or in `localStorage`. Require a data-processing review, `async`
loading, and a measured performance budget before it ships.
</details>

---

**Next:** [12 — Observability & Delivery](12_Observability_And_Delivery.md)
