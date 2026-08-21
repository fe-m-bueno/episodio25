---
status: closed
type: research
parent: ../../map.md
claimed_by: research-bff-policy-2026-08-20
---

# SvelteKit BFF, cache, and abuse protection

## Question

Which current SvelteKit and Vercel primitives should define the server-only TMDB boundary, runtime, regional cache keys, request timeouts, rate limiting, 429 handling, method restrictions, and log redaction for a public deployment?

## Resolution

Use allowlisted SvelteKit `+server.ts` routes and a `$lib/server` TMDB client with `TMDB_ACCESS_TOKEN` imported from `$env/static/private`. The conservative deployment target is Vercel Node 22 with explicit upstream aborts, bounded fan-out, country-aware CDN caching, normalized errors, and log redaction. Durable or WAF rate limiting is plan-dependent; in-memory serverless limiting is not reliable. See [SvelteKit and Vercel BFF policy](../../../../docs/research/sveltekit-vercel-bff-policy.md).

