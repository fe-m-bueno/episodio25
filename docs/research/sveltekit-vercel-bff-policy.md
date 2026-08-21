# SvelteKit and Vercel BFF policy for TMDB

Date: 2026-08-20  
Claim: `research-bff-policy-2026-08-20`  
Ticket: `.scratch/episodios-tematicos/tickets/open/012-bff-cache-security.md`

## Decision summary

Use SvelteKit `+server.ts` routes as a small, allowlisted backend-for-frontend (BFF). Keep `TMDB_ACCESS_TOKEN` in a server-only module, send it to TMDB as an `Authorization: Bearer` header, and never accept an arbitrary upstream URL from the browser. Deploy the BFF on the Vercel Node.js runtime, use explicit upstream abort deadlines, and configure a short function duration as a final guardrail.

For the MVP, cache successful public GET responses at Vercel's CDN, with country-dependent responses keyed by `Vary: X-Vercel-IP-Country`. Keep browser caching conservative. Do not cache client errors, upstream errors, timeouts, rate-limit responses, or `/api/region`. Apply a Vercel WAF rate limit to `/api/**` if the selected plan supports it. Application-memory counters are not a valid substitute in a horizontally scaled serverless deployment.

The recommended concrete starting limits are:

- GET and HEAD only for read routes; return `405` for every other method.
- Search query: 2-100 Unicode characters after trimming.
- Page: integer 1-20; one upstream page per BFF request.
- At most 20 normalized results per response.
- Country: exactly two ASCII letters, uppercased and checked against the app's supported ISO country set.
- No more than 3 TMDB calls per incoming request, with concurrency capped at 3.
- 4.5 second upstream deadline and 10 second Vercel `maxDuration`.
- Edge rate limit starting point: 60 requests per minute per IP for `/api/**`, with a tighter 20 requests per minute per IP rule for `/api/search`.
- Successful neutral search/detail responses: Vercel CDN TTL 5 minutes, stale-while-revalidate 10 minutes.
- Successful curated theme responses: Vercel CDN TTL 15 minutes, stale-while-revalidate 30 minutes.
- Successful provider responses: Vercel CDN TTL 1 hour, stale-while-revalidate 6 hours, varied by country.

All numerical limits above are design recommendations, not limits documented by SvelteKit, Vercel, or TMDB. They should be revised from production traffic and quota observations.

## Documented facts

### Secret boundary

SvelteKit's `$env/static/private` and `$env/dynamic/private` modules can only be imported by server-only code. Files under `$lib/server` and files with a `.server` suffix are also server-only. SvelteKit checks direct and indirect import chains and fails the build when public-facing code imports them. Illegal-import detection is disabled under tests, so a production build is still required as a secret-leak check. [SvelteKit server-only modules](https://svelte.dev/docs/kit/server-only-modules)

`$env/static/private` injects values at build time. `$env/dynamic/private` reads values at runtime. Neither is importable by client-side code. [SvelteKit static private environment variables](https://svelte.dev/docs/kit/%24env-static-private), [SvelteKit dynamic private environment variables](https://svelte.dev/docs/kit/%24env-dynamic-private)

Vercel environment variables are encrypted at rest. Variables marked Sensitive are write-only in the dashboard after creation. This controls storage and dashboard visibility, not what application code may print at runtime. [Vercel environment variables](https://vercel.com/docs/environment-variables), [Vercel sensitive environment variables](https://vercel.com/docs/environment-variables/sensitive-environment-variables)

### SvelteKit endpoint and method behavior

SvelteKit `+server.ts` files run only on the server and can export handlers for `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, and `HEAD`. A `fallback` export catches otherwise unhandled methods. When `GET` exists, SvelteKit uses it for `HEAD` unless an explicit `HEAD` handler takes precedence. [SvelteKit routing and `+server`](https://svelte.dev/docs/kit/routing#server)

SvelteKit's `json`, `error`, and `redirect` helpers are optional conveniences around standard `Response` behavior. Unhandled errors become negotiated error responses based on `Accept`; API code should therefore deliberately normalize expected upstream failures instead of leaking framework or upstream error details. [SvelteKit `+server` error behavior](https://svelte.dev/docs/kit/routing#server)

### Adapter and runtime

`@sveltejs/adapter-vercel` supports deployment configuration globally or on individual `+server.ts`, `+page.server.ts`, and `+layout.server.ts` files. Its documented options include `runtime`, `regions`, `split`, and, for serverless functions, `maxDuration`. The adapter documentation currently marks its `runtime` option as deprecated in favor of the Node version configured for the Vercel project. [SvelteKit adapter-vercel](https://svelte.dev/docs/kit/adapter-vercel#deployment-configuration)

The adapter currently documents `edge`, `nodejs20.x`, and `nodejs22.x`. Node runtime has full Node.js API coverage; Edge supports a smaller Web API surface. Vercel request cancellation is only available for Node functions and requires `supportsCancellation: true` in `vercel.json`. [SvelteKit adapter-vercel](https://svelte.dev/docs/kit/adapter-vercel#deployment-configuration), [Vercel Functions API](https://vercel.com/docs/functions/functions-api-reference#cancel-requests), [Vercel function limits](https://vercel.com/docs/functions/limitations)

Vercel's current function-duration defaults depend on plan and whether Fluid compute is enabled. If a function exceeds its duration, Vercel returns `504 FUNCTION_INVOCATION_TIMEOUT`. The current Vercel limits page is authoritative for the deployed account and can differ from older defaults quoted by adapter documentation. [Vercel function limits](https://vercel.com/docs/functions/limitations#max-duration)

### Country detection

Vercel adds `x-vercel-ip-country`, a two-character ISO 3166-1 country code inferred from the requester's public IP, to deployment requests. It can be read from the standard `Request.headers`. It is contextual geolocation data, not proof of identity or an authorization boundary. [Vercel request headers](https://vercel.com/docs/headers/request-headers#x-vercel-ip-country)

### CDN caching and `Vary`

Vercel Functions can cache complete dynamic responses by returning cache directives such as `s-maxage` and `stale-while-revalidate`. `Vercel-CDN-Cache-Control` has priority for Vercel's cache and is consumed rather than forwarded to the browser; `CDN-Cache-Control` is next; standard `Cache-Control` has the lowest priority. Function response headers override route-level headers for the same response. [Vercel CDN cache](https://vercel.com/docs/caching/cdn-cache), [Vercel cache-control headers](https://vercel.com/docs/caching/cache-control-headers)

Vercel's CDN includes headers named by `Vary` in its cache key when a cache-enabling directive is present. Vercel explicitly documents `Vary: X-Vercel-IP-Country` for country-specific variants. Adding many varying headers increases cache fragmentation and misses. [Vercel `Vary` behavior](https://vercel.com/docs/caching/cdn-cache#vary-header)

The target URI participates in the HTTP cache key. Therefore a validated `country=BR` override is already a distinct URI from `country=US`; `Vary` is still required whenever the response may fall back to `x-vercel-ip-country` for the same URI. [RFC 9111 cache operation](https://www.rfc-editor.org/rfc/rfc9111.html#section-2)

### Timeouts and cancellation

Node provides `AbortSignal.timeout(milliseconds)` and `AbortSignal.any(signals)`. These allow an upstream `fetch` to stop on either an application deadline or another abort signal. [Node.js `AbortSignal`](https://nodejs.org/api/globals.html#static-method-abortsignaltimeoutdelay)

Vercel Node functions can observe client cancellation through `Request.signal` when `supportsCancellation` is enabled. Vercel's example forwards that cancellation to an upstream fetch with an `AbortController`. [Vercel Functions request cancellation](https://vercel.com/docs/functions/functions-api-reference#cancel-requests)

### Rate limiting and TMDB 429 responses

Vercel provides automatic DDoS mitigation on all plans. Its WAF supports custom rules, while rate limiting is a separately priced, plan-dependent feature. WAF rate-limit rules can terminate with a `429` response, and the `@vercel/firewall` SDK can check a configured rule from application code. The SDK still requires a corresponding Firewall rule configured in the project. [Vercel Firewall](https://vercel.com/docs/vercel-firewall), [WAF custom rules](https://vercel.com/docs/vercel-firewall/vercel-waf/custom-rules), [Rate Limiting SDK](https://vercel.com/docs/vercel-firewall/vercel-waf/rate-limiting-sdk), [WAF usage and pricing](https://vercel.com/docs/vercel-firewall/vercel-waf/usage-and-pricing)

TMDB no longer publishes the old fixed limit of 40 requests per 10 seconds, but says it retains upper limits around 40 requests per second for bulk-scraping protection. TMDB says that limit may change and clients must respect `429` responses. [TMDB rate limiting](https://developer.themoviedb.org/docs/rate-limiting)

### Logging boundaries

Vercel Runtime Logs include Function output such as `console.log`, request path and search parameters, and information about outgoing requests. Logs are visible to project users with suitable access and may also be exported through Log Drains. [Vercel Runtime Logs](https://vercel.com/docs/logs/runtime)

Vercel documents automatic redaction for marked Sensitive Environment Variable values in build logs when the value is at least 32 characters, but this is not a general promise that arbitrary runtime log messages, request URLs, headers, error objects, or Log Drain payloads are secret-safe. [Vercel build-log redaction](https://vercel.com/changelog/build-logs-now-redact-sensitive-environment-variable-values)

## Constraints derived from the facts

1. **Server-only is necessary but not sufficient.** It prevents the token from entering browser bundles. It does not stop clients from consuming TMDB quota through the public BFF.
2. **The BFF cannot be a generic proxy.** Allowing arbitrary paths, query keys, or upstream URLs would expose TMDB's surface and turn the project into a quota relay.
3. **Country-dependent caching must be explicit.** A response derived from `x-vercel-ip-country` without `Vary` can be served to the wrong country. A manual country override must be validated and canonicalized before use.
4. **Function duration is not an upstream deadline.** A 10-second function cap prevents indefinite billing, but the TMDB fetch must abort earlier so the BFF can still return a controlled response.
5. **In-memory rate limiting is unreliable.** Serverless instances scale horizontally, restart, and do not share durable state. A process-local map cannot enforce a project-wide quota.
6. **Cache keys can become attacker-controlled cardinality.** Unbounded search strings, page numbers, country values, sort keys, or ignored query parameters create cache misses and quota amplification.
7. **Logs are a disclosure surface.** A bearer token in a query string can appear in outgoing-request URLs. Raw incoming URLs contain search terms. Logging request headers, full upstream errors, or serialized `Request` objects can expose credentials and user data.
8. **TMDB's ceiling is intentionally not contractual.** The app must treat `429` as normal operational backpressure, not as an exceptional impossible state.

## Recommended MVP policy

### 1. Secret storage and upstream client

- Require a Vercel Sensitive Environment Variable named `TMDB_ACCESS_TOKEN` for Preview and Production. Document the name without inspecting or committing any local value.
- Import the token only from a module under `$lib/server`, using `$env/static/private` unless runtime rotation without rebuilding becomes a requirement. Use `$env/dynamic/private` only when that runtime behavior is intentionally needed.
- Send the token only as `Authorization: Bearer <token>`. Do not append `api_key` or the bearer token to a URL.
- Centralize the upstream origin as the literal `https://api.themoviedb.org/3`. Map internal operations to fixed paths and allowlisted query keys. Never accept an upstream origin, path, or complete URL from a request.
- Fail startup/build clearly when the variable is absent, but never include its value in the error.

### 2. Route and method restrictions

- Expose purpose-specific `+server.ts` routes, for example search, theme results, media details, providers, and region.
- Export `GET` only for read routes. Accept implicit `HEAD`. Add a shared `fallback` that returns `405 Method Not Allowed` with `Allow: GET, HEAD` and `Cache-Control: no-store`.
- Do not enable CORS. Same-origin browser calls need no CORS headers, and omitting them prevents ordinary cross-origin JavaScript from reading the API. This is not authentication and does not stop direct HTTP clients.
- Reject request bodies on read routes and reject unknown query parameters with `400`, rather than silently accepting high-cardinality cache keys that do not affect the response.

### 3. Input, fan-out, and response limits

| Input or operation | MVP limit | Failure |
| --- | ---: | --- |
| Search text | 2-100 trimmed Unicode characters | `400 INVALID_QUERY` |
| Page | integer 1-20 | `400 INVALID_PAGE` |
| Country override | exactly 2 ASCII letters and supported ISO code | `400 INVALID_COUNTRY` |
| Sort/filter enum | exact allowlisted values only | `400 INVALID_FILTER` |
| Unknown query key | none accepted | `400 UNKNOWN_PARAMETER` |
| TMDB calls per request | maximum 3 | internal invariant, otherwise `500 POLICY_VIOLATION` |
| Concurrent TMDB calls | maximum 3 | queue within the request; never unbounded `Promise.all` |
| Results returned | maximum 20 | truncate and expose bounded pagination metadata |
| Upstream response body | reject above 2 MiB when measurable | `502 UPSTREAM_RESPONSE_TOO_LARGE` |

The BFF must not scan arbitrary seasons or enumerate an unbounded set of series to discover episodes. Episode hydration must start from a finite local set of episode keys decided elsewhere in the product plan.

### 4. Runtime and deadlines

- Use Vercel Node.js, configured as Node 22 in the Vercel project, and omit the adapter's deprecated `runtime` override. Node is the conservative choice for a BFF because it has full Node API coverage and supports Vercel request cancellation.
- Set `maxDuration: 10` for BFF routes. This is a cost and runaway-execution ceiling, not the normal response target.
- Give each TMDB operation a 4.5 second deadline. Combine that deadline with request cancellation where Vercel supports it. Reserve the remaining function time for normalization and a controlled error response.
- Do not automatically retry general TMDB failures in the request path. A retry multiplies quota use and latency. One retry may be considered later only for an idempotent GET that failed before headers with a transient network error, and only within the same total deadline.
- Enable `supportsCancellation` for the generated BFF function pattern after verifying the adapter's emitted function paths in a preview deployment. This setting lives in Vercel configuration and is only supported by Node functions.

### 5. Country selection and cache key

Use this precedence:

1. A valid explicit `country` query parameter.
2. A valid `x-vercel-ip-country` value.
3. `BR` in local development. In production, use an explicit unknown sentinel and omit provider claims when the header is absent or invalid.

Return the resolved country in the normalized response so the UI never has to infer it. Canonicalize explicit country values to uppercase and redirect or reject noncanonical input before a cacheable response.

For any cacheable response whose content uses the Vercel country header, include:

```http
Cache-Control: public, max-age=0, must-revalidate
Vercel-CDN-Cache-Control: public, s-maxage=<ttl>, stale-while-revalidate=<swr>
Vary: X-Vercel-IP-Country
```

Do not add `Cookie`, `User-Agent`, full IP headers, or arbitrary client headers to `Vary`. They would fragment or effectively disable the shared cache.

### 6. Cache matrix

| Response class | Browser policy | Vercel policy | `Vary` |
| --- | --- | --- | --- |
| Neutral search/detail success | `max-age=0, must-revalidate` | `s-maxage=300, stale-while-revalidate=600` | none unless country affects output |
| Curated theme success | `max-age=0, must-revalidate` | `s-maxage=900, stale-while-revalidate=1800` | country only if providers are embedded |
| Provider success | `max-age=0, must-revalidate` | `s-maxage=3600, stale-while-revalidate=21600` | `X-Vercel-IP-Country` when header-derived |
| `/api/region` | `no-store` | none | none |
| Any `4xx`, `429`, `5xx`, timeout, or malformed upstream response | `no-store` | none | none |

Do not use `stale-if-error` in the first MVP release. It can be valuable for availability data, but it makes failure testing and freshness semantics harder to verify. Add it only after preview tests prove the intended Vercel behavior.

### 7. Rate limiting and abuse response

- Preferred deployment policy: Vercel WAF rules scoped by request path and source IP, initially 60 requests per minute for `/api/**` and 20 requests per minute for `/api/search`.
- Return `429` with a small JSON error. When the application calls the `@vercel/firewall` SDK and constructs the response itself, include `Retry-After: 60`. A dashboard WAF response must be tested because the official documentation does not promise that it includes `Retry-After`.
- Test each WAF rule in log mode on Preview, then publish the limiting action to Production after checking false positives.
- If the project's Vercel plan does not include suitable rate limiting, do not implement an in-memory limiter. Launch only with strict input/fan-out limits, CDN caching, Vercel usage alerts, and monitoring, or add a durable external rate-limit store as a separately approved dependency.
- Cache hits still consume edge traffic and may be evaluated by the WAF, but they avoid TMDB calls. Cache policy is therefore the primary TMDB quota defense; the WAF is the burst/abuse defense.

### 8. Error contract and upstream 429 handling

Every error response should use a stable public schema:

```json
{
  "error": {
    "code": "UPSTREAM_TIMEOUT",
    "message": "The catalog service did not respond in time.",
    "retryable": true,
    "requestId": "public-correlation-id"
  }
}
```

Recommended mapping:

| Condition | BFF status | Public code | Retry policy |
| --- | ---: | --- | --- |
| Invalid input or unknown parameter | `400` | specific validation code | do not retry unchanged |
| Unknown local resource | `404` | `NOT_FOUND` | do not retry |
| BFF/WAF rate limit | `429` | `RATE_LIMITED` | honor `Retry-After` when present; otherwise wait 60 seconds |
| TMDB `401` or `403` | `502` | `UPSTREAM_AUTH_FAILURE` | operator action; never expose TMDB body |
| TMDB `404` for requested media | `404` | `NOT_FOUND` | do not retry |
| TMDB `429` | `503` | `UPSTREAM_RATE_LIMITED` | forward a valid bounded `Retry-After`, otherwise use 30 seconds |
| TMDB `5xx` or invalid JSON/schema | `502` | `UPSTREAM_FAILURE` | client may retry later |
| Application upstream deadline | `504` | `UPSTREAM_TIMEOUT` | client may retry later |
| Unexpected BFF failure | `500` | `INTERNAL_ERROR` | operator investigation |

Do not synchronously retry TMDB `429`. Returning `503` distinguishes shared upstream exhaustion from a per-client BFF limit. Preserve only a syntactically valid `Retry-After`, capped to 300 seconds; otherwise emit 30 seconds. All error responses use `Cache-Control: no-store`.

### 9. Logging and redaction

Log only a bounded structured record:

- public correlation/request ID;
- route template, not a reconstructed full URL;
- normalized operation name such as `tmdb.search.movie`, never the bearer token or full upstream URL;
- resolved country, response status, cache-relevant category, duration, upstream status, and timeout/rate-limit classification;
- counts such as result count and upstream call count.

Never log:

- `Authorization`, cookies, all request/response headers, environment objects, or the token;
- raw `Request`, `Response`, `URL`, or fetch options objects;
- the full incoming search string or complete query string;
- TMDB response bodies or raw upstream error objects;
- stack traces in public JSON responses;
- full client IP. If temporary abuse diagnostics require an IP-derived key, use it only inside the rate-limit provider and do not emit it to application logs.

Use the bearer header instead of a token-bearing TMDB URL because Vercel Runtime Logs expose outgoing-request metadata. Treat Vercel's Sensitive-variable and build-log redaction as defense in depth, not as the application's redaction policy.

## Verification criteria for implementation tickets

1. A production build fails if browser-facing code imports the TMDB client or private env module.
2. Built client assets and rendered HTML contain neither the env variable name's value nor an authorization header value.
3. Every BFF route rejects unhandled methods with `405` and the correct `Allow` header.
4. Unknown parameters, oversized search text, invalid pages, and invalid countries fail before any TMDB call.
5. A simulated slow TMDB request is aborted near 4.5 seconds and returns normalized `504`, before Vercel's function timeout.
6. Simulated TMDB `401`, `403`, `404`, `429`, `5xx`, invalid JSON, and oversized responses map to the documented public errors without upstream bodies.
7. Country-dependent responses contain `Vary: X-Vercel-IP-Country`; Preview tests show distinct cache entries for at least `BR` and `US` using `x-vercel-cache` observations.
8. Error and `/api/region` responses are never CDN-cached. Successful endpoints match the cache matrix.
9. Runtime logs contain operation metadata but no token, authorization header, full query string, raw search term, TMDB body, or full IP.
10. Preview smoke tests verify Node runtime, `maxDuration`, client cancellation, and the actual adapter-emitted function path used by `supportsCancellation`.
11. Load tests verify configured WAF rules return `429` without invoking TMDB after the threshold; if the SDK path is used, verify `Retry-After: 60`.

## Explicit unknowns

1. **Vercel plan and Fluid compute status:** these determine available function-duration defaults and whether WAF rate limiting is available and billable. Confirm in the target project's dashboard before implementation.
2. **Adapter output path for cancellation:** `supportsCancellation` matches generated Vercel function paths, not necessarily source route paths. Inspect a Preview build before fixing the glob.
3. **TMDB `Retry-After` behavior:** TMDB requires clients to respect `429`, but its public rate-limit page does not promise that every `429` includes `Retry-After` or define its format.
4. **TMDB quota scope:** public documentation does not state a durable per-token contractual ceiling or exact burst algorithm. The approximate 40 requests/second figure may change.
5. **Vercel WAF rate-limit availability:** current documentation describes rate limiting as plan-dependent and priced, while custom WAF rules themselves are available on all plans. Confirm the exact entitlement for the account.
6. **Unknown-country representation:** the policy requires an explicit unknown state with no provider claim, but the public code and response shape for that state still need to be chosen.
7. **Provider freshness requirement:** the recommended one-hour TTL is a design starting point. Neither TMDB nor JustWatch publishes a freshness SLA for this app's use case.
8. **Cache-key normalization:** verify in Preview how Vercel treats query parameter order. The app should emit one canonical order regardless, to avoid depending on undocumented normalization.
9. **Outgoing request metadata redaction:** Vercel documents that outgoing requests appear in Runtime Logs but does not provide a complete field-level guarantee for bearer headers. The application must assume URLs and metadata are observable and must never place secrets in URLs.
10. **WAF `Retry-After`:** Vercel documents a `429` follow-up action but does not promise a `Retry-After` header for dashboard-enforced limits. Preview tests must determine the observed response; clients need a 60-second fallback.

## Sources consulted

- [SvelteKit server-only modules](https://svelte.dev/docs/kit/server-only-modules)
- [SvelteKit private environment variables](https://svelte.dev/docs/kit/%24env-static-private)
- [SvelteKit routing and `+server`](https://svelte.dev/docs/kit/routing#server)
- [SvelteKit adapter-vercel](https://svelte.dev/docs/kit/adapter-vercel)
- [Vercel request headers](https://vercel.com/docs/headers/request-headers)
- [Vercel CDN cache](https://vercel.com/docs/caching/cdn-cache)
- [Vercel cache-control headers](https://vercel.com/docs/caching/cache-control-headers)
- [Vercel function limits](https://vercel.com/docs/functions/limitations)
- [Vercel Functions API](https://vercel.com/docs/functions/functions-api-reference)
- [Vercel Firewall](https://vercel.com/docs/vercel-firewall)
- [Vercel WAF custom rules](https://vercel.com/docs/vercel-firewall/vercel-waf/custom-rules)
- [Vercel Rate Limiting SDK](https://vercel.com/docs/vercel-firewall/vercel-waf/rate-limiting-sdk)
- [Vercel Runtime Logs](https://vercel.com/docs/logs/runtime)
- [Vercel sensitive environment variables](https://vercel.com/docs/environment-variables/sensitive-environment-variables)
- [Node.js `AbortSignal`](https://nodejs.org/api/globals.html#class-abortsignal)
- [TMDB rate limiting](https://developer.themoviedb.org/docs/rate-limiting)
