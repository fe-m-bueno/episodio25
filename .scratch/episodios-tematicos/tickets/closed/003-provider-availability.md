---
status: closed
type: research
parent: ../../map.md
---

# Regional provider strategy

## Question

How should the app show streaming availability by country?

## Resolution

Use TMDB watch provider data keyed by the two-letter ISO country code. Read the production country from Vercel request metadata without asking for browser geolocation permission. Use a JustWatch destination only when it can be generated reliably; otherwise use the TMDB watch link.

