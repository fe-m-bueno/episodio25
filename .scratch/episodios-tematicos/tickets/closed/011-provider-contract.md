---
status: closed
type: research
parent: ../../map.md
claimed_by: research-provider-contract-2026-08-20
---

# Provider granularity, links, and attribution

## Question

At which media granularities does TMDB provide regional watch-provider data, what does the returned link represent, which JustWatch and TMDB attributions are required, and which list-level provider filters are feasible without per-item fan-out?

## Resolution

TMDB exposes regional provider data for movies, series, and seasons, but not episodes. Episode UI must label season or series availability. The returned outbound URL is a TMDB watch page, not a JustWatch deep link. Provider filtering without fan-out is available only on movie and TV discover endpoints. TMDB branding and visible JustWatch data attribution are release requirements. See [TMDB watch-provider contract](../../../../docs/research/tmdb-provider-contract.md).

