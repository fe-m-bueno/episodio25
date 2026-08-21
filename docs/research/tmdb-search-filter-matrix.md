# TMDB search and filter capability matrix

Research date: 2026-08-20  
Scope: TMDB API v3 search and discover capabilities for movies, TV series, seasons, and episodes  
Claim: `research-search-capabilities-2026-08-20`

## Executive finding

TMDB exposes text search for movies and TV series, and filter-driven discovery for movies and TV series. It does not expose a global season or episode text-search or discover endpoint. Seasons and episodes are addressable only after a TV series and season are known. Therefore, a thematic catalog that promises individual episodes needs a finite episode source, such as locally curated `{ seriesId, seasonNumber, episodeNumber }` references; TMDB can enrich those references but cannot discover the complete episode catalog by theme. This follows from TMDB's documented finding model, which limits `/search` and `/discover` to movies, TV shows, and people, plus the identity-based season and episode detail routes ([Finding Data](https://developer.themoviedb.org/docs/finding-data), [TV season details](https://developer.themoviedb.org/reference/tv-season-details), [TV episode details](https://developer.themoviedb.org/reference/tv-episode-details)).

TMDB does not publish a cross-media relevance score. Search results have an upstream order but no `relevance` field, while discover exposes media-specific sorts. A deterministic MVP should preserve upstream order within each source and avoid claiming that popularity, vote totals, or relevance are directly comparable across movies, TV series, and episodes.

## Facts

### Endpoint capability matrix

| Media kind | Text search | Filtered discover | Keyword-driven discover | Direct detail identity | Pagination |
| --- | --- | --- | --- | --- | --- |
| Movie | `GET /3/search/movie` searches original, translated, and alternative titles. | `GET /3/discover/movie` supports more than 30 filters and sort options. | Resolve a keyword with `GET /3/search/keyword`, then pass its ID to `with_keywords` on discover. | `movie_id` | Search and discover return `page`, `results`, `total_pages`, and `total_results`. Requested pages are globally limited to 1-500. |
| TV series | `GET /3/search/tv` searches original, translated, and also-known-as names. | `GET /3/discover/tv` supports more than 30 filters and sort options. | Resolve a keyword, then pass its ID to `with_keywords` on discover. | `series_id` | Search and discover use the same paged envelope and 1-500 request limit. |
| TV season | None documented. | None documented. | None documented. | `series_id` + `season_number` | Season detail is one resource, not a paged discovery result. It contains the season's episode array. |
| TV episode | None documented. | None documented. | None documented. | `series_id` + `season_number` + `episode_number` | Episode detail is one resource, not a paged discovery result. |

Sources: [Search movie](https://developer.themoviedb.org/reference/search-movie), [Search TV](https://developer.themoviedb.org/reference/search-tv), [Search keyword](https://developer.themoviedb.org/reference/search-keyword), [Discover movie](https://developer.themoviedb.org/reference/discover-movie), [Discover TV](https://developer.themoviedb.org/reference/discover-tv), [TV season details](https://developer.themoviedb.org/reference/tv-season-details), [TV episode details](https://developer.themoviedb.org/reference/tv-episode-details), and the API-wide invalid-page error stating that pages start at 1 and max at 500 ([Errors, status code 22](https://developer.themoviedb.org/docs/errors)).

The OpenAPI document and endpoint references do not promise a fixed result count per page. The application must consume the returned envelope rather than infer `total_results` from a presumed page size ([TMDB OpenAPI](https://developer.themoviedb.org/openapi/tmdb-api.json)).

### Search capabilities

| Endpoint | Inputs | Sorting and filtering | Result kinds and important list fields |
| --- | --- | --- | --- |
| `/3/search/movie` | Required `query`; optional `include_adult` (default `false`), `language` (default `en-US`), `page` (default `1`), `primary_release_year`, `region`, and `year`. | No `sort_by`, genre, rating, vote-count, keyword, provider, or popularity filter is documented. Returned order must be treated as TMDB search order. | Movie list objects expose `id`, `title`, `original_title`, `original_language`, `overview`, `adult`, `genre_ids`, `release_date`, `popularity`, `vote_average`, `vote_count`, `poster_path`, `backdrop_path`, and `video`. |
| `/3/search/tv` | Required `query`; optional `first_air_date_year`, `include_adult` (default `false`), `language` (default `en-US`), `page` (default `1`), and `year`. `first_air_date_year` searches only the first-air year; `year` searches the first air date and all episode air dates. Both document valid years as 1000-9999. | No `sort_by`, genre, rating, vote-count, keyword, provider, or popularity filter is documented. | TV list objects expose `id`, `name`, `original_name`, `original_language`, `origin_country`, `overview`, `adult`, `genre_ids`, `first_air_date`, `popularity`, `vote_average`, `vote_count`, `poster_path`, and `backdrop_path`. |
| `/3/search/multi` | Required `query`; optional `include_adult` (default `false`), `language` (default `en-US`), and `page` (default `1`). | No media-kind filter or `sort_by` is documented. | Returns movies, TV shows, and people in one response. Results use `media_type`; person results have person-shaped fields and may include a heterogeneous `known_for` list. A consumer that supports only media must explicitly reject `media_type: person`. |
| `/3/search/keyword` | Required `query`; optional `page` (default `1`). | Searches keyword names; it is not content search. | Returns keyword `id` and `name`, which can feed movie or TV `with_keywords`. |

Sources: [Search movie](https://developer.themoviedb.org/reference/search-movie), [Search TV](https://developer.themoviedb.org/reference/search-tv), [Search multi](https://developer.themoviedb.org/reference/search-multi), [Search keyword](https://developer.themoviedb.org/reference/search-keyword), and the response schemas in the official [TMDB OpenAPI](https://developer.themoviedb.org/openapi/tmdb-api.json).

TMDB describes `/search` as text matching over original, translated, and alternative names or titles, and `/discover` as filtering over definable values. Search is therefore not a discover query with a text parameter, and discover is not a full-text search endpoint ([Finding Data](https://developer.themoviedb.org/docs/finding-data)).

### Movie discover

`GET /3/discover/movie` documents these filter families ([Discover movie](https://developer.themoviedb.org/reference/discover-movie)):

| Family | Parameters and semantics |
| --- | --- |
| Localization and paging | `language` (default `en-US`), `page` (default `1`), `region`, `include_adult` (default `false`), `include_video` (default `false`). |
| Dates and years | `primary_release_year`, `year`, `primary_release_date.gte`, `primary_release_date.lte`, `release_date.gte`, `release_date.lte`. If `region` is supplied, TMDB says the regional release date is used instead of the primary release date; the returned date can depend on ordered `with_release_type` values. |
| Certification | `certification`, `certification.gte`, `certification.lte`, and `certification_country`; the reference documents their required relationship with `region` or `certification_country`. |
| Quality and duration | `vote_average.gte`, `vote_average.lte`, `vote_count.gte`, `vote_count.lte`, `with_runtime.gte`, `with_runtime.lte`. |
| People and organizations | `with_cast`, `with_crew`, `with_people`, `with_companies`, `without_companies`. |
| Classification | `with_genres`, `without_genres`, `with_keywords`, `without_keywords`, `with_origin_country`, `with_original_language`, `with_release_type`. Release types are 1-6. |
| Availability | `watch_region`, `with_watch_providers`, `without_watch_providers`, and `with_watch_monetization_types`. Monetization types are `flatrate`, `free`, `ads`, `rent`, and `buy`. The reference says provider and monetization filters are used with `watch_region`. |

Movie `sort_by` supports `original_title`, `popularity`, `revenue`, `primary_release_date`, `title`, `vote_average`, and `vote_count`, each ascending or descending. The default is `popularity.desc` ([Discover movie](https://developer.themoviedb.org/reference/discover-movie)).

### TV series discover

`GET /3/discover/tv` documents these filter families ([Discover TV](https://developer.themoviedb.org/reference/discover-tv)):

| Family | Parameters and semantics |
| --- | --- |
| Localization and paging | `language` (default `en-US`), `page` (default `1`), `include_adult` (default `false`). |
| Dates | `air_date.gte`, `air_date.lte`, `first_air_date_year`, `first_air_date.gte`, `first_air_date.lte`, `timezone`, and `include_null_first_air_dates` (default `false`). `screened_theatrically` is also available. |
| Quality and duration | `vote_average.gte`, `vote_average.lte`, `vote_count.gte`, `vote_count.lte`, `with_runtime.gte`, `with_runtime.lte`. |
| Classification and ownership | `with_companies`, `without_companies`, `with_genres`, `without_genres`, `with_keywords`, `without_keywords`, `with_networks`, `with_origin_country`, `with_original_language`, `with_status` (0-5), and `with_type` (0-6). |
| Availability | `watch_region`, `with_watch_providers`, `without_watch_providers`, and `with_watch_monetization_types`; monetization types are `flatrate`, `free`, `ads`, `rent`, and `buy`. |

TV `sort_by` supports `first_air_date`, `name`, `original_name`, `popularity`, `vote_average`, and `vote_count`, each ascending or descending. The default is `popularity.desc` ([Discover TV](https://developer.themoviedb.org/reference/discover-tv)).

On both discover endpoints, the reference states that supported comma-separated values mean logical AND and pipe-separated values mean logical OR. This rule must only be applied to parameters whose documentation says they support those separators ([Discover movie](https://developer.themoviedb.org/reference/discover-movie), [Discover TV](https://developer.themoviedb.org/reference/discover-tv)).

`region` and `watch_region` are different contracts: `region` affects movie release-date and certification behavior, while `watch_region` scopes provider and monetization filters. The application should not substitute one for the other ([Discover movie](https://developer.themoviedb.org/reference/discover-movie), [Discover TV](https://developer.themoviedb.org/reference/discover-tv)).

### Seasons and episodes

A season is fetched with `GET /3/tv/{series_id}/season/{season_number}`. The documented season response includes `_id`, `id`, `name`, `overview`, `air_date`, `poster_path`, `season_number`, `vote_average`, and an `episodes` array. Episode objects in the official schema include identity and display fields such as `id`, `episode_number`, `season_number`, `name`, `overview`, `air_date`, `runtime`, `still_path`, `vote_average`, and `vote_count`; the schema does not expose an episode-level popularity field ([TV season details](https://developer.themoviedb.org/reference/tv-season-details), [TMDB OpenAPI](https://developer.themoviedb.org/openapi/tmdb-api.json)).

An episode detail requires `series_id`, `season_number`, and `episode_number`; an episode's numeric `id` alone is not the documented route identity. Its detail schema includes `air_date`, `episode_number`, `season_number`, `id`, `name`, `overview`, `production_code`, `runtime`, `still_path`, `vote_average`, `vote_count`, crew, and guest stars, but not genres, origin country, keywords, providers, or episode popularity ([TV episode details](https://developer.themoviedb.org/reference/tv-episode-details), [TMDB OpenAPI](https://developer.themoviedb.org/openapi/tmdb-api.json)). Genres and origin displayed for an episode must therefore be explicitly labeled as inherited from its parent series.

Season and episode detail methods support `append_to_response`, with no more than 20 appended subrequests. This reduces HTTP round trips but does not create episode search or discover functionality ([Append To Response](https://developer.themoviedb.org/docs/append-to-response), [Errors, status code 27](https://developer.themoviedb.org/docs/errors)).

TMDB documents watch-provider data for a TV season at `GET /3/tv/{series_id}/season/{season_number}/watch/providers`. It is season-scoped, provides availability per country, does not provide full provider deep links, and requires JustWatch attribution. The supplied TMDB URL is the supported outbound link ([TV season watch providers](https://developer.themoviedb.org/reference/tv-season-watch-providers)). It must not be presented as proof of episode-specific availability.

### Documented null and missing-value behavior

The one explicit null-control in the reviewed search/discover surface is TV discover's `include_null_first_air_dates`, which defaults to `false` ([Discover TV](https://developer.themoviedb.org/reference/discover-tv)). Movie discover has no equivalent documented `include_null_release_dates` parameter.

The official response examples contain nullable image paths, but the OpenAPI schemas do not consistently declare all operationally nullable or blank fields as nullable. The search and discover references do not define ordering semantics for missing dates, zero-vote ratings, blank overviews, or missing artwork ([TMDB OpenAPI](https://developer.themoviedb.org/openapi/tmdb-api.json)). A client must validate these values instead of assuming every declared string is non-empty.

## Mixed-list comparison matrix

| Candidate field or sort | Movie vs TV series | Movie vs episode | Safe claim in a mixed UI |
| --- | --- | --- | --- |
| Identity | IDs can collide across namespaces. | IDs can collide, and episode routing needs the compound episode key. | Safe only as a namespaced key such as `movie:123`, `tv:123`, or `episode:{seriesId}:{season}:{episode}`. |
| Title | `title` vs `name` can be normalized. | `title` vs episode `name` can be normalized. | Safe for display and deterministic locale-aware alphabetical sort after normalization. |
| Date | `release_date` vs `first_air_date` represent different lifecycle events. | Movie release vs episode air date are both publication events but not identical concepts. | Safe only when labeled generically as `date` and documented per kind. Missing or invalid dates sort last. |
| `vote_average` | Same nominal field and scale, but different populations and vote counts. | Episode ratings exist, but episodes usually have a different voting population from movies. | Displayable. A cross-kind ranking is only a product heuristic, not a TMDB relevance or quality truth; require a vote-count floor and show vote count. |
| `vote_count` | Mechanically sortable, but audience sizes differ by media kind. | Movie and episode totals are especially scale-skewed. | Displayable, but not safely interpretable as cross-kind quality or relevance. Prefer per-kind sort. |
| `popularity` | Present on movie and TV list objects, but TMDB does not document cross-media calibration. | No episode popularity field is documented. | Do not offer mixed popularity sort. Keep it within movie-only or TV-only upstream results. |
| Search relevance | Search returns ordered results but no score. | No episode search exists. | Preserve source order and call it `TMDB search order`; do not merge by a fabricated relevance number. |
| Genre | Movie and TV use list-level `genre_ids`; genre vocabularies are media-specific endpoints. | Episode details have no genre; it can inherit series genres. | Filter each media kind with its own IDs, normalize to labels for display, and identify inherited episode metadata. |
| Origin country | Movie discover can filter `with_origin_country`; movie list objects do not expose `origin_country`. TV list objects expose it. | Episodes can only inherit series origin. | Not a uniformly available list-result field. Filtering must happen per source or through enrichment. |
| Provider availability | Movie and TV discover can filter by provider and watch region. | Episode list/detail does not carry provider data; season provider data can be fetched separately. | Mixed filtering is possible only with an explicit availability scope (`movie` or `season`) and enrichment. Never imply episode-level precision. |

The safety judgments above are application conclusions derived from the documented field and endpoint asymmetries. TMDB does not state that popularity or relevance is normalized across media kinds ([Search multi](https://developer.themoviedb.org/reference/search-multi), [Discover movie](https://developer.themoviedb.org/reference/discover-movie), [Discover TV](https://developer.themoviedb.org/reference/discover-tv), [TV episode details](https://developer.themoviedb.org/reference/tv-episode-details)).

## Constraints for this product

1. A query cannot combine arbitrary text search with discover's advanced filters in one documented TMDB request. Search and discover are separate products with different parameter sets.
2. Multi-search admits people and has no documented media-kind filter. Removing people after retrieval can produce short pages and makes TMDB's `total_results` unsuitable as a media-only total.
3. There is no global episode query by title, overview, genre, keyword, rating, popularity, provider, or date. Crawling all seasons from discovered series would be an application-side fan-out, not a TMDB search capability.
4. Keyword lookup and keyword-based discovery are two steps. `search/keyword` resolves names to IDs; `with_keywords` filters movies or TV series by those IDs. The deprecated keyword-movies endpoint should not be used; TMDB directs clients to movie discover with `with_keywords` ([Keyword movies](https://developer.themoviedb.org/reference/keyword-movies)).
5. Provider filters are available during movie and TV discovery only when paired with a watch region. Season provider data is a separate enrichment request.
6. The API accepts pages 1-500. Even if `total_pages` is greater than 500, pages beyond 500 are not requestable under the documented global error contract ([Errors](https://developer.themoviedb.org/docs/errors)).
7. Search order, discover order, curated episode order, and application-defined thematic relevance are different concepts and must retain provenance.

## Recommendation: deterministic MVP contract

This section is a product recommendation, not a statement of TMDB behavior.

### 1. Free-text search

- Call `/search/movie` and `/search/tv` separately with the same normalized `query`, `language`, page, and adult-content policy.
- Do not use `/search/multi`; separate endpoints exclude person results and preserve meaningful per-source pagination totals.
- Return two source groups (`movies`, `series`) and preserve TMDB's order within each group. Label the default order `Relevance` in the UI only if product copy clarifies that it means TMDB search order within each media kind.
- Allow only media-kind and supported year filters on the search surface. When the user requests genre, rating, vote, provider, origin, or popularity filtering, switch to a discover surface and do not pretend the text query is still being applied by TMDB.
- If a single visual stream is required, interleave source groups by stable round-robin using `(sourcePage, sourceRank, mediaKind)` and retain those values. Do not sort the merged stream by popularity or rating.

### 2. Thematic discovery

- Resolve and store vetted TMDB keyword IDs in the local theme manifest. Do not resolve theme keywords on every user request.
- Query `/discover/movie` and `/discover/tv` independently with each theme's keyword expression and the filters supported by that media kind.
- Keep pagination, totals, and upstream sorting separate per kind. A combined `All` view should be an interleaving presentation over independently paged sources, not a claim of global TMDB ranking.
- Support these first-class upstream sorts per movie or TV group: `popularity`, `vote_average`, `vote_count`, newest, and oldest. For rating sorts, apply a product-chosen minimum vote threshold to reduce zero- or low-sample artifacts.
- Use provider filters only when `watch_region` is valid and present. Keep `region` as a separate movie-release context.

### 3. Thematic episodes

- Source episode membership from a local curated manifest using `EpisodeKey = { seriesId, seasonNumber, episodeNumber }`.
- Enrich keys from season details, caching one season response for all curated episodes in that season. Do not perform an unbounded crawl of discovered series.
- Normalize episode fields from the episode object and explicitly attach inherited series fields (`genres`, `originCountry`, anime classification) with provenance.
- Represent availability as `availabilityScope: "season"`; fetch season providers only when availability is requested or needed for a filter.
- Do not expose episode popularity. Episode date, rating, and vote count may be displayed, but rating and votes should sort episode-only groups by default.

### 4. Mixed thematic list sorts

Use the following deterministic contract:

| UI sort | Mixed movies + episodes | Tie-breakers |
| --- | --- | --- |
| Curated relevance | Allowed only from an explicit local `themeRank` or rank tier. It is not TMDB relevance. | `themeRank`, then `mediaKind`, then namespaced identity. |
| Newest / oldest | Allowed using movie `release_date` and episode `air_date`, with the generic label `Date`. | Valid date, then `themeRank`, then identity; missing dates always last. |
| Highest rated | Recommended only in per-kind views. If retained in `All`, require a documented minimum vote count and label it as a cross-kind heuristic. | `vote_average`, `vote_count`, `themeRank`, identity; missing values last. |
| Most voted | Recommended only in per-kind views because exposure differs substantially. | `vote_count`, `themeRank`, identity. |
| Popularity | Disabled in mixed movie + episode lists because episodes have no documented popularity field. | Not applicable. |

All sort implementations should use a final namespaced identity tie-breaker so results are stable across requests and JavaScript engines.

## Explicit unknowns

- TMDB does not document the relevance algorithm or expose a numeric search relevance score.
- TMDB does not document that movie and TV popularity values are calibrated for cross-media comparison.
- The reviewed references do not specify a guaranteed page size, only the page envelope and requestable page range.
- The reviewed references do not fully specify whether missing dates are returned as `null`, an empty string, or omitted across every endpoint and localization state.
- The references do not define how `vote_average` behaves statistically at very low `vote_count`; a minimum-vote rule is an application policy.
- `include_null_first_air_dates` documents inclusion control but not the ordering of null dates under every TV discover sort.
- The search movie reference accepts `region` but does not explain its complete ranking or filtering effect there. Do not infer discover's regional-release behavior for search.
- The API reference does not define a complete cross-endpoint snapshot-consistency guarantee. Popularity, votes, availability, translations, totals, and result order may change between independently fetched pages.
- Provider completeness and freshness are not guaranteed by the reviewed reference. The data should be described as TMDB/JustWatch availability data, not an exhaustive market guarantee.
