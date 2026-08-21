# TMDB episode discovery feasibility

Research date: 2026-08-20  
Scope: official TMDB API v3 capabilities for discovering individual thematic TV and anime episodes

## Decision summary

TMDB does not expose a global text-search or discover operation whose result type is an individual TV episode. Its documented search and discover flows find TV series, not episodes. Individual episode data is addressed under a known series and season using the compound key `(series_id, season_number, episode_number)`.

The application should therefore treat thematic episodes as a finite, locally curated catalog. TMDB should hydrate and refresh known references, not discover episodes by scanning arbitrary series and seasons during a user request. Automatic TMDB discovery remains useful for finding candidate series for offline editorial work, but it cannot guarantee thematic episode results by itself.

## Method and evidence standard

Context7 was queried first against its high-reputation TMDB documentation corpus (`/websites/developer_themoviedb`). Claims below are linked to TMDB's own current API reference, guide, or OpenAPI description. No authenticated API calls were made, and no local environment variables were read.

The phrase "no global episode search/discover" means that no such operation appears in TMDB's documented API surface as of the research date. It is not a claim about undocumented internal TMDB services.

## Verified facts

### 1. Global discovery returns series, not individual episodes

TMDB describes three general ways to find data: `/search` for text, `/discover` for filtered movies and TV shows, and `/find` for known external identifiers. The guide describes search/discover targets as movies, TV shows, and people, not individual episodes. [TMDB: Finding Data](https://developer.themoviedb.org/docs/finding-data)

The TV text-search operation is:

```text
GET /3/search/tv
```

It searches TV shows by original, translated, and alternative names and returns TV-show results. Its `year` parameter may consider episode air dates, but that parameter changes which shows match; it does not change the result type to episodes. [TMDB: Search TV Shows](https://developer.themoviedb.org/reference/search-tv)

The TV discover operation is:

```text
GET /3/discover/tv
```

It finds TV shows using filters and sort options, including series-level keywords, genres, origin country, original language, ratings, and watch-provider filters. It returns TV-show results. [TMDB: Discover TV](https://developer.themoviedb.org/reference/discover-tv)

Multi-search does not add episodes: it searches movies, TV shows, and people. [TMDB: Multi Search](https://developer.themoviedb.org/reference/search-multi)

`GET /3/find/{external_id}` can return a TV episode when the caller already has a supported episode external ID, including IMDb, TheTVDB, or Wikidata identifiers. This is identity resolution, not thematic or textual discovery. [TMDB: Find By ID](https://developer.themoviedb.org/reference/find-by-id)

There is no separate anime search, discover, season, or episode resource in the documented API. Anime candidates are TV or movie records that an application classifies using available metadata such as genre, original language, and origin country. `discover/tv` exposes those series-level filters, but no documented filter establishes that a particular episode is anime or matches a theme. [TMDB: Discover TV](https://developer.themoviedb.org/reference/discover-tv)

### 2. Exact hierarchy needed to reach episode data

The minimum hierarchy for enumerating episodes is:

```text
GET /3/tv/{series_id}
GET /3/tv/{series_id}/season/{season_number}
GET /3/tv/{series_id}/season/{season_number}/episode/{episode_number}
```

Series details return a known TV show's metadata and support `append_to_response`. The series is addressed by `series_id`. [TMDB: TV Series Details](https://developer.themoviedb.org/reference/tv-series-details)

Season details require both `series_id` and `season_number`. The response includes an `episodes` array with episode identity and display metadata such as `id`, `episode_number`, `name`, `overview`, `air_date`, and still/vote fields. This is the operation that can enumerate the episodes of one already-known season. [TMDB: TV Season Details](https://developer.themoviedb.org/reference/tv-season-details)

Episode details require all three coordinates: `series_id`, `season_number`, and `episode_number`. A standalone TMDB episode ID is not accepted by the episode-detail path. [TMDB: TV Episode Details](https://developer.themoviedb.org/reference/tv-episode-details)

TMDB does expose a standalone-ID route for recent episode changes:

```text
GET /3/tv/episode/{episode_id}/changes
```

That route reports change history and is not an alternate episode-details or discovery endpoint. [TMDB: TV Episode Changes](https://developer.themoviedb.org/reference/tv-episode-changes-by-id)

### 3. Relevant namespace endpoints

The following read endpoints can enrich a known series, season, or episode. They do not discover arbitrary individual episodes.

#### Series-level operations relevant to the catalog

```text
GET /3/tv/{series_id}
GET /3/tv/{series_id}/external_ids
GET /3/tv/{series_id}/keywords
GET /3/tv/{series_id}/translations
GET /3/tv/{series_id}/watch/providers
GET /3/tv/{series_id}/changes
```

The details operation is the source of the series hierarchy. External IDs and translations help verify and localize curated references. Keywords are attached to the series, not to an individual episode. Watch providers are also exposed at TV-series scope. [TMDB: TV Series Details](https://developer.themoviedb.org/reference/tv-series-details), [External IDs](https://developer.themoviedb.org/reference/tv-series-external-ids), [Translations](https://developer.themoviedb.org/reference/tv-series-translations), [Watch Providers](https://developer.themoviedb.org/reference/tv-series-watch-providers), [Changes](https://developer.themoviedb.org/reference/tv-series-changes)

#### Season-level operations relevant to the catalog

```text
GET /3/tv/{series_id}/season/{season_number}
GET /3/tv/{series_id}/season/{season_number}/aggregate_credits
GET /3/tv/{series_id}/season/{season_number}/credits
GET /3/tv/{series_id}/season/{season_number}/external_ids
GET /3/tv/{series_id}/season/{season_number}/images
GET /3/tv/{series_id}/season/{season_number}/translations
GET /3/tv/{series_id}/season/{season_number}/videos
GET /3/tv/{series_id}/season/{season_number}/watch/providers
GET /3/tv/season/{season_id}/changes
```

Season details are the important fan-out operation because one response contains that season's episode summaries. The other endpoints enrich an already-known season. The changes endpoint is exceptional: it uses the season's standalone TMDB `season_id`, not the series/season-number pair. [TMDB: TV Season Details](https://developer.themoviedb.org/reference/tv-season-details), [Aggregate Credits](https://developer.themoviedb.org/reference/tv-season-aggregate-credits), [Credits](https://developer.themoviedb.org/reference/tv-season-credits), [External IDs](https://developer.themoviedb.org/reference/tv-season-external-ids), [Images](https://developer.themoviedb.org/reference/tv-season-images), [Translations](https://developer.themoviedb.org/reference/tv-season-translations), [Videos](https://developer.themoviedb.org/reference/tv-season-videos), [Watch Providers](https://developer.themoviedb.org/reference/tv-season-watch-providers), [Changes](https://developer.themoviedb.org/reference/tv-season-changes-by-id)

#### Episode-level operations relevant to the catalog

```text
GET /3/tv/{series_id}/season/{season_number}/episode/{episode_number}
GET /3/tv/{series_id}/season/{season_number}/episode/{episode_number}/credits
GET /3/tv/{series_id}/season/{season_number}/episode/{episode_number}/external_ids
GET /3/tv/{series_id}/season/{season_number}/episode/{episode_number}/images
GET /3/tv/{series_id}/season/{season_number}/episode/{episode_number}/translations
GET /3/tv/{series_id}/season/{season_number}/episode/{episode_number}/videos
GET /3/tv/episode/{episode_id}/changes
```

These operations all require a known episode identity. The compound-key operations provide details and enrichment; the standalone-ID operation provides recent changes only. [TMDB: TV Episode Details](https://developer.themoviedb.org/reference/tv-episode-details), [Credits](https://developer.themoviedb.org/reference/tv-episode-credits), [External IDs](https://developer.themoviedb.org/reference/tv-episode-external-ids), [Images](https://developer.themoviedb.org/reference/tv-episode-images), [Translations](https://developer.themoviedb.org/reference/tv-episode-translations), [Videos](https://developer.themoviedb.org/reference/tv-episode-videos), [Changes](https://developer.themoviedb.org/reference/tv-episode-changes-by-id)

### 4. `append_to_response` capabilities and limits

TMDB documents `append_to_response` on movie, TV-show, TV-season, TV-episode, and person detail methods. It accepts comma-separated child endpoints from the same namespace and appends each child response as another JSON object in the base response. Child-specific query parameters still apply. [TMDB: Append To Response](https://developer.themoviedb.org/docs/append-to-response)

The TV series, season, and episode detail references each document an explicit maximum of 20 comma-separated appended items:

```text
GET /3/tv/{series_id}?append_to_response=...
GET /3/tv/{series_id}/season/{season_number}?append_to_response=...
GET /3/tv/{series_id}/season/{season_number}/episode/{episode_number}?append_to_response=...
```

[TMDB: TV Series Details](https://developer.themoviedb.org/reference/tv-series-details), [TV Season Details](https://developer.themoviedb.org/reference/tv-season-details), [TV Episode Details](https://developer.themoviedb.org/reference/tv-episode-details)

For a known episode, a request such as the following can combine its base details with several episode child resources:

```text
GET /3/tv/{series_id}/season/{season_number}/episode/{episode_number}
    ?append_to_response=external_ids,images,translations,videos,credits
```

This reduces HTTP round trips for enrichment of that one episode. It does not provide a wildcard, a text predicate, or a batch of unrelated episode identities. Because each episode detail URL has its own series/season/episode coordinates, `append_to_response` does not eliminate episode-level fan-out when complete episode details are requested for many episodes.

TMDB's general append guide does not explicitly document dynamic child paths such as appending multiple numbered seasons to a series details call. This research does not assume that behavior, and the recommended contract does not depend on it.

## Request-shape constraints

Let:

- `P` be the number of pages requested from `search/tv` or `discover/tv`;
- `C` be the number of candidate series inspected;
- `S_i` be the number of seasons scanned for candidate series `i`;
- `E` be the number of episodes for which full episode details are fetched.

An online scan has the following lower-bound HTTP shape:

```text
series discovery + hierarchy enumeration = P + C + sum(S_i)
with full details for candidate episodes = P + C + sum(S_i) + E
```

Why:

1. Search/discover returns series, costing `P` requests.
2. Each candidate series needs details to establish its season hierarchy, costing `C` requests.
3. Each season must be fetched to obtain episode titles and overviews, costing `sum(S_i)` requests.
4. Each selected episode needs its own detail request if fields beyond the season summary are required, costing `E` requests. Appended child resources can ride on that episode's detail request but cannot combine different episodes.

Illustrative shapes, not measurements of the live catalog:

| Assumptions | Requests before per-episode details | If every enumerated episode also needs details |
| --- | ---: | ---: |
| 1 discover page, 20 series, 8 seasons per series | `1 + 20 + 160 = 181` | add one request for every episode found |
| 2 discover pages, 50 series, 5 seasons per series | `2 + 50 + 250 = 302` | add one request for every episode found |

If the first example averaged 10 episodes per season, fetching every episode's full details would add 1,600 requests for a total of 1,781. This number is deliberately hypothetical; it demonstrates how the hierarchy multiplies requests and is not a claim about typical TMDB series sizes.

Scanning title and overview from season responses avoids the `+E` term, but it still requires arbitrary `C + sum(S_i)` fan-out and transfers all episode summaries for every candidate season. Theme matching would then be application-authored text classification, not a TMDB-supported thematic episode query.

## Constraints for this application

### Verified constraints

- Theme keywords accepted by `discover/tv` select TV series. They do not select individual episodes. [TMDB: Discover TV](https://developer.themoviedb.org/reference/discover-tv)
- Episode details cannot be fetched from only an episode TMDB ID; the documented detail route requires series ID, season number, and episode number. [TMDB: TV Episode Details](https://developer.themoviedb.org/reference/tv-episode-details)
- One season-details request can provide the episode summaries needed for list cards and basic text matching. [TMDB: TV Season Details](https://developer.themoviedb.org/reference/tv-season-details)
- Provider data is documented at TV-series and TV-season scope, not episode scope. [TMDB: TV Series Watch Providers](https://developer.themoviedb.org/reference/tv-series-watch-providers), [TV Season Watch Providers](https://developer.themoviedb.org/reference/tv-season-watch-providers)
- TMDB's watch-provider data does not return full provider deep links; consumers should use the supplied TMDB URL, and JustWatch attribution is required. [TMDB: TV Season Watch Providers](https://developer.themoviedb.org/reference/tv-season-watch-providers)

### Product and engineering consequences

The following are reasoned consequences, not TMDB statements:

- Request-time series/season scanning has unbounded cost unless the application imposes arbitrary limits on candidate series, season count, episode count, and concurrency.
- Any such limit creates unstable recall: a thematic episode may disappear because its series ranked outside the inspected candidate window, not because it is absent from TMDB.
- Runtime text matching against episode titles and overviews will produce language-dependent false positives and false negatives.
- A provider badge on an episode must be labeled as series- or season-level availability, never guaranteed episode availability.
- A finite local catalog makes result quality reviewable, deterministic, cacheable, and independent of upstream ranking changes.

## Recommendation: finite local episode catalog

### Canonical contract

Store curated episode references in version-controlled data. The canonical identity is the compound key required by TMDB:

```ts
type ThemeSlug = string;

type CuratedEpisodeReference = {
  seriesId: number;
  seasonNumber: number;
  episodeNumber: number;
  themes: ThemeSlug[];

  // Editorial evidence and maintenance metadata
  rationale: string;
  sourceNote?: string;
  priority?: number;
  addedAt: string;
  verifiedAt?: string;

  // Drift guards, not canonical identity
  expectedSeriesName?: string;
  expectedEpisodeName?: string;

  // Editorial classification, not a TMDB media type
  format?: "live-action" | "animation" | "anime";
};

type EpisodeCatalog = {
  schemaVersion: 1;
  episodes: CuratedEpisodeReference[];
};
```

`seriesId + seasonNumber + episodeNumber` must be unique across the catalog. Multiple themes should be attached to the same record rather than duplicating it. `expectedSeriesName` and `expectedEpisodeName` should be used only to flag unexpected upstream drift during validation; names are not stable identifiers.

`rationale` is intentionally required. It gives editorial reviewers a concrete reason that an episode belongs to each declared theme and prevents the catalog from becoming an unexplained ID list.

### Runtime hydration contract

For a theme page:

1. Select local references containing the requested `ThemeSlug`.
2. Group references by `seriesId` and `(seriesId, seasonNumber)`.
3. Fetch each unique series once when series-level metadata is needed.
4. Fetch each unique season once, then select the referenced episode numbers from its `episodes` array.
5. Fetch the individual episode-detail endpoint only when opening a detail page or when a required field is absent from the season summary.
6. Fetch provider data only at the labeled series or season scope and cache it by country.
7. If a reference cannot be resolved, omit or mark it unavailable and emit a maintenance signal; never compensate by scanning adjacent seasons.

For `U_series` unique series and `U_seasons` unique series/season pairs on one theme page, hydration costs at most:

```text
U_series + U_seasons
```

before optional provider calls. If the list UI needs only episode fields already present in season details and precomputed local series labels, it can reduce to `U_seasons`. Opening one episode detail adds one request. These bounds are determined by the local catalog size instead of the size of arbitrary TMDB search results.

### Editorial discovery workflow

Use TMDB's series search/discover only to propose candidate series during an offline or explicit maintenance workflow. A curator can then inspect selected seasons, verify episodes, and commit exact compound references. Do not run that workflow as part of a public theme-page request.

Automated title/overview matching may rank candidates for a curator, but it should not publish episodes without review. This preserves a clear distinction between TMDB-provided metadata and the application's thematic judgment.

### Validation rules

Recommended validation for every catalog change:

- all three identity numbers are non-negative integers, with an explicit policy on season `0` specials;
- every theme slug exists in the canonical theme manifest;
- compound identities are unique;
- every reference resolves through its season-details response;
- returned series and episode names are compared with drift guards;
- every theme meets a separately defined editorial minimum and has no unresolved entries;
- anime classification is reviewed editorially rather than inferred solely from `ja` or origin country `JP`;
- availability is labeled with `series` or `season` scope.

## Explicit unknowns

- TMDB does not document a semantic definition or dedicated classifier for "anime". The project must own that classification policy.
- The completeness and language coverage of episode overviews, stills, votes, translations, external IDs, and provider data vary by record and region; this research did not measure live coverage.
- The general append guide documents child endpoints in the same namespace, but does not explicitly guarantee numbered dynamic paths such as appending `season/1,season/2` to a series call. The proposal intentionally does not rely on this.
- TMDB's documentation states the 20-item append limit but does not state in the cited pages how appended subrequests are counted internally for rate limiting or quotas.
- The optimal number of curated episodes per theme is a product/editorial quality decision. It should be set after sampling the 30 themes, not inferred from API mechanics.
- The policy for specials in season `0`, multipart episodes, double-length broadcasts, reordered anime seasons, and alternate episode groups remains to be specified.
- A refresh cadence for detecting deleted, renumbered, merged, or renamed records remains to be specified.

## Final feasibility determination

Global thematic episode discovery through the official TMDB API is not a viable runtime contract because the documented discovery surface returns series and reaching episodes requires hierarchical fan-out. The viable MVP contract is a finite local catalog keyed by `(seriesId, seasonNumber, episodeNumber)`, hydrated from season details and enriched on demand. This guarantees bounded request shapes and makes thematic relevance an explicit, reviewable product decision.
