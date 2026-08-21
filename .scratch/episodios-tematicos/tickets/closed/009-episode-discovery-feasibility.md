---
status: closed
type: research
parent: ../../map.md
claimed_by: research-episode-discovery-2026-08-20
---

# Episode discovery feasibility

## Question

Which official TMDB operations can produce individual thematic episodes, what fan-out would automatic discovery require, and which finite local catalog shape can guarantee useful episode results without scanning arbitrary series and seasons at request time?

## Resolution

TMDB has no documented global episode search or discovery endpoint. Runtime discovery would require unbounded hierarchical fan-out through candidate series and seasons. The viable MVP contract is a finite, version-controlled episode catalog keyed by `(seriesId, seasonNumber, episodeNumber)`, hydrated from unique season responses and enriched with individual episode details only on demand. See [TMDB episode discovery feasibility](../../../../docs/research/tmdb-episode-discovery.md).

