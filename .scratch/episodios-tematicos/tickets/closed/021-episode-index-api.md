---
status: closed
type: grilling
parent: ../../map.md
blocked_by:
  - 009-episode-discovery-feasibility.md
  - 013-canonical-theme-manifest.md
claimed_by: episode-index-design-2026-08-20
---

# Thematic episode index API

## Question

How should the product discover, classify, update, and serve 50–200 thematic TV and anime episodes per theme without runtime catalog crawling, a paid episode-search API, or complete manual curation?

## Resolution

Approved the [Thematic episode index API design](../../../../docs/plans/2026-08-20-episode-index-api-design.md). TVmaze supplies the licensed episode corpus, local semantic inference produces versioned confidence-scored theme associations, AniList selectively verifies anime identity, and TMDB enriches only visible results without supplying classifier input. SvelteKit serves reviewed static shards, and a scheduled GitHub Action proposes transactional updates through pull requests.
