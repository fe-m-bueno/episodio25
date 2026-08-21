# Episódios temáticos

## Destination

Produce an approved, buildable MVP specification for an English-language thematic media discovery app built with SvelteKit and deployed to Vercel. The app lets people browse themes, search movies and TV episodes, filter and sort results, inspect details, and see regional provider availability without exposing TMDB credentials.

## Notes

- Domain: thematic discovery for movies, TV series, and anime episodes.
- Primary skills: Svelte 5, SvelteKit, shadcn-svelte, Context7, Vercel deployment.
- The TMDB credential must never be read by the browser or committed to the repository.
- The design uses a dark cinematic catalog language with an English interface.
- The app has no account, persistence, Stremio integration, or direct JustWatch API dependency in this effort.
- The initial design review found unresolved feasibility and contract questions. The map remains open until every research and decision ticket is closed.

## Decisions so far

- [MVP destination and client experience](tickets/closed/001-mvp-scope.md) - Explore themes and search content in a client-focused app, with a minimal server proxy for private TMDB access.
- [Content item model](tickets/closed/002-content-item-model.md) - Use `movie` and `episode` as the two catalog item types.
- [Regional provider strategy](tickets/closed/003-provider-availability.md) - Use TMDB provider data by country and link outward when a reliable JustWatch destination exists.
- [Filter and sort model](tickets/closed/004-filters-and-sorting.md) - Make type, genre, year, region, provider, rating, vote count, popularity, and date first-class controls.
- [Theme discovery strategy](tickets/closed/005-theme-discovery.md) - Combine local theme configuration and curated seeds with TMDB keyword discovery.
- [Interface language](tickets/closed/006-interface-language.md) - Use English for the UI and English TMDB responses with original titles as fallback metadata.
- [Visual direction](tickets/closed/007-visual-direction.md) - Use a dark cinematic catalog with zinc surfaces, one amber accent, restrained motion, and accessible light mode.
- [Stremio scope boundary](tickets/closed/008-stremio-out-of-scope.md) - Remove playlist export and Stremio integration from this effort.
- [Episode discovery feasibility](tickets/closed/009-episode-discovery-feasibility.md) - Individual thematic episodes require a finite local catalog of compound episode keys.
- [Search, filter, and sort capabilities](tickets/closed/010-search-filter-capabilities.md) - Search and discover support movies and series; episodes and mixed lists require separate local contracts.
- [Provider granularity, links, and attribution](tickets/closed/011-provider-contract.md) - Providers are movie, series, or season scoped; the safe outbound URL is TMDB's watch page with required attribution.
- [SvelteKit BFF, cache, and abuse protection](tickets/closed/012-bff-cache-security.md) - A public deployment needs an allowlisted Node BFF with bounded fan-out, regional cache keys, deadlines, and abuse controls.
- [Canonical theme manifest and quality bar](tickets/closed/013-canonical-theme-manifest.md) - Fix all 30 English themes and require at least eight verified items, including five episode items and two movies, per theme.
- [Thematic episode index API](tickets/closed/021-episode-index-api.md) - Generate 50–200 episode matches per theme from a licensed TVmaze corpus, classify them offline, and serve versioned shards through SvelteKit with optional TMDB enrichment.

## Not yet specified

- The final home and theme-page information hierarchy may change after the result and filter capability matrix is resolved.
- The implementation sequence remains fog until the remaining result, filter, provider, and domain-model decisions are resolved.

## Out of scope

- Stremio lists, playlist downloads, addon manifests, or library integration.
- Authentication, accounts, favorites, database persistence, and cross-device sync.
- Scraping JustWatch or streaming providers.
- Direct integration with a private or undocumented JustWatch API.
- AI-generated recommendations.
- [Manual curation of the complete episode catalog](tickets/closed/020-curated-episode-research.md) - Superseded because it would remain too small and labor-intensive; three completed slices are retained only as benchmark and audit material.
