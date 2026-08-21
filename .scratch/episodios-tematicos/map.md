# Episódios temáticos

## Destination

Produce an approved, buildable MVP specification for an English-language thematic media discovery app built with SvelteKit and deployed to Vercel. The app lets people browse themes, search movies and TV episodes, filter and sort results, inspect details, and see regional provider availability without exposing TMDB credentials.

## Notes

- Domain: thematic discovery for movies, TV series, and anime episodes.
- Primary skills: Svelte 5, SvelteKit, shadcn-svelte, Context7, Vercel deployment.
- The TMDB credential must never be read by the browser or committed to the repository.
- The design uses a dark cinematic catalog language with an English interface.
- The app has no account, persistence, Stremio integration, or direct JustWatch API dependency in this effort.

## Decisions so far

- [MVP destination and client experience](tickets/closed/001-mvp-scope.md) - Explore themes and search content in a client-focused app, with a minimal server proxy for private TMDB access.
- [Content item model](tickets/closed/002-content-item-model.md) - Use `movie` and `episode` as the two catalog item types.
- [Regional provider strategy](tickets/closed/003-provider-availability.md) - Use TMDB provider data by country and link outward when a reliable JustWatch destination exists.
- [Filter and sort model](tickets/closed/004-filters-and-sorting.md) - Make type, genre, year, region, provider, rating, vote count, popularity, and date first-class controls.
- [Theme discovery strategy](tickets/closed/005-theme-discovery.md) - Combine local theme configuration and curated seeds with TMDB keyword discovery.
- [Interface language](tickets/closed/006-interface-language.md) - Use English for the UI and English TMDB responses with original titles as fallback metadata.
- [Visual direction](tickets/closed/007-visual-direction.md) - Use a dark cinematic catalog with zinc surfaces, one amber accent, restrained motion, and accessible light mode.
- [Stremio scope boundary](tickets/closed/008-stremio-out-of-scope.md) - Remove playlist export and Stremio integration from this effort.

## Not yet specified

- The exact curated seed IDs and keyword IDs for each of the 30 themes.
- The exact existing TMDB environment variable name, because the `.env` file is intentionally unread.
- The exact shape of a direct JustWatch URL for each title. The MVP must fall back to the TMDB watch link when a reliable JustWatch destination cannot be constructed.
- Whether provider data should be prefetched for result cards or loaded only when a detail view opens. Start with detail-time loading unless performance evidence requires otherwise.

## Out of scope

- Stremio lists, playlist downloads, addon manifests, or library integration.
- Authentication, accounts, favorites, database persistence, and cross-device sync.
- Scraping JustWatch or streaming providers.
- Direct integration with a private or undocumented JustWatch API.
- AI-generated recommendations.

