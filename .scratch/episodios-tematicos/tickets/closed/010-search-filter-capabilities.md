---
status: closed
type: research
parent: ../../map.md
claimed_by: research-search-capabilities-2026-08-20
---

# Search, filter, and sort capabilities

## Question

What search, discover, filter, sorting, pagination, and field capabilities does TMDB expose for movies, series, seasons, and episodes, and which capabilities can be compared safely when results from different media kinds are displayed together?

## Resolution

TMDB search and discover operate on movies and TV series, not seasons or episodes. Free-text search cannot use discover's advanced filters. Popularity is unavailable for episodes, and ratings or vote counts are not naturally comparable across kinds. Separate movie and series source groups preserve upstream ordering and pagination; mixed thematic lists need explicit local ranking and stable namespaced tie-breakers. See [TMDB search and filter capability matrix](../../../../docs/research/tmdb-search-filter-matrix.md).

