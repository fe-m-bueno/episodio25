# Episódios temáticos MVP design

## Destination

Build an English-language thematic media discovery app for movies and TV or anime episodes. Users browse a fixed set of themes, search TMDB-backed content, filter and sort results, open details, and see provider availability for their country. The app is deployed to Vercel and keeps TMDB credentials server-side.

## Product scope

The first release is a client-focused SvelteKit application with a minimal server-side proxy. It has no accounts, database, favorites, cross-device persistence, Stremio integration, playlist download, or direct JustWatch API dependency.

The 30 requested themes are included as local configuration. The home page highlights a smaller selection, while a theme browser exposes the complete set.

## Domain model

- `Theme`: slug, English label, description, editorial group, supported media types, TMDB keyword identifiers, episode matching terms, and curated seed references.
- `ContentItem`: discriminated union of `MovieItem` and `EpisodeItem`.
- `MovieItem`: TMDB movie identity, title, overview, poster and backdrop paths, release date, genres, score, vote count, popularity, and provider data when loaded.
- `EpisodeItem`: series identity, season number, episode number, episode title, overview, air date, still path, score, vote count, and parent series metadata.
- `ProviderAvailability`: country code, provider name, provider logo, monetization type, and external watch link.
- `CountryContext`: normalized two-letter ISO country code plus a source indicator for production header or local fallback.
- `ThemeMatch`: curated or inferred association between an item and a theme.

## User flow

### Home

The home page presents the search field, a concise product statement, a visual selection of TMDB imagery, featured themes, and a route to the complete theme browser.

### Theme page

The theme page presents the English theme title and description, active filters, sorting, and a result grid. Controls include media type, genre, year range, origin country, current-country availability, provider, rating, vote count, and popularity. The default sort is popularity.

### Search

Search uses TMDB multi-search for movies and TV shows, with explicit type filters. The default sort is relevance. Search input is debounced and the query plus controls are encoded in the URL.

### Content details

Movie details show title, year, overview, genres, ratings, popularity, providers, and an external watch action. TV and anime details show seasons and thematic episodes. Episode details show series, season and episode numbers, title, overview, air date, ratings, and provider information where available.

## Discovery strategy

Local theme definitions establish the editorial vocabulary for the 30 themes. Curated seeds guarantee useful initial content. TMDB keyword discovery expands the results. For TV and anime, episode data is loaded after a series is selected; title and overview terms are used to rank thematic matches, with curated episode references taking precedence.

The anime filter uses animation and Japanese-origin signals as a starting point. It does not claim that every Japanese animation title is anime, so the local theme configuration remains the source of editorial precision.

## Filters and sorting

Supported filters:

- `type`: all, movie, episode, anime;
- `genre`;
- `yearFrom` and `yearTo`;
- `originCountry`;
- `availableInCountry`;
- `provider`;
- `minRating`;
- `minVoteCount`;
- `minPopularity`.

Supported sorts:

- relevance;
- popularity;
- highest rated;
- newest;
- oldest;
- most voted.

Every control is serializable in URL search parameters. Desktop uses a compact toolbar and filter groups. Mobile uses a shadcn-svelte Sheet with a visible reset action and removable active-filter chips.

## Server data flow

The browser calls only app-owned endpoints. SvelteKit server routes validate known query parameters, select allowed TMDB operations, attach the private credential, normalize the result, and return only the fields required by the UI.

Planned app endpoints:

- `/api/search` for free search;
- `/api/themes/[slug]` for theme discovery;
- `/api/media/[type]/[id]` for details;
- `/api/media/tv/[id]/season/[season]` for season episodes;
- `/api/media/[type]/[id]/providers` for regional provider data;
- `/api/region` for the detected country context.

The proxy must not accept arbitrary upstream URLs. It must enforce pagination bounds, reject unknown media types, and return consistent error objects. TMDB responses use short cache headers to reduce repeated requests without making content feel stale.

## Country and providers

Production requests read `x-vercel-ip-country` and normalize the result to uppercase ISO format. Local development falls back to `BR`. If no valid country is available, the interface shows a clear unavailable-region state and can offer a manual country override without requesting browser geolocation.

TMDB provider data is shown by monetization category: subscription, free or ad-supported, rental, and purchase. The UI uses an external watch action when a reliable JustWatch destination is available and otherwise falls back to the TMDB watch link. The app does not scrape JustWatch or call an undocumented JustWatch API.

## Visual system

- Dark cinematic default with accessible light mode.
- Charcoal background, zinc surfaces, one amber accent.
- Consistent 12px corner radius.
- Sans-serif typography with a strong but restrained display hierarchy.
- Real TMDB posters and backdrops as the primary imagery.
- Asymmetric theme atlas rather than a generic row of equal feature cards.
- Motion intensity 4: short result transitions, restrained poster hover, and no perpetual decoration.
- Reduced-motion fallback for every non-essential motion.
- English interface copy, English TMDB language preference, original title as secondary metadata.

## States and accessibility

The product includes skeleton loaders matching result and detail layouts, empty states with recovery suggestions, retryable API errors, unavailable-provider messaging, undetected-country handling, and missing-translation fallback.

All controls are keyboard reachable with visible focus. Inputs have real labels. Icon actions have accessible names. Images have title-based alt text. Contrast targets WCAG AA. Mobile filter sheets preserve focus and provide a clear close action.

## Testing decisions

Tests should exercise external behavior at the highest available seam:

- pure normalization and sorting functions receive fixture responses and produce stable domain items;
- server routes validate parameters, pass country context, and map upstream failures to safe error responses;
- theme matching ranks curated and inferred matches correctly;
- UI tests cover search, filter changes, sort changes, detail opening, retry, empty, and provider states;
- the production build and Svelte autofixer run before handoff.

## Success criteria

- A visitor can open the app, choose any of the 30 themes, and see a useful first result set.
- A visitor can search for a movie or series and filter or sort the result without losing state on reload.
- A TV or anime detail view can expose relevant seasons and thematic episodes.
- Provider availability reflects the production country header and is clearly labeled.
- TMDB credentials never appear in browser code, rendered HTML, or committed files.
- The app builds with the Vercel adapter and passes type checking and Svelte validation.

