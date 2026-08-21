# TMDB watch-provider contract

Research date: 2026-08-20  
Scope: TMDB v3 watch-provider endpoints, Discover filters, link semantics, attribution, and consequences for this application's list and detail surfaces.  
Sources: first-party TMDB documentation and terms, plus first-party JustWatch documentation only where explicitly identified. Context7 was queried first against TMDB's official developer documentation index (`/websites/developer_themoviedb`); the linked official pages below were then used to verify the decision-relevant claims.

## Decision summary

TMDB has documented watch-provider detail endpoints for **movies, TV series, and TV seasons**, but not for individual TV episodes. Episode availability must therefore be represented as series-level or season-level availability, never as a claim that a specific episode is available.

The provider result's `link` is a **TMDB `/watch` page**, not a JustWatch deep link and not a provider playback link. TMDB explicitly says the API does not return full deep links; the supplied TMDB page is where users can reach the actual outbound links.

List-level provider filtering is feasible without per-item fan-out only for lists produced by `discover/movie` and `discover/tv`. Those filters constrain the result set but do not add provider arrays or the TMDB watch URL to each result. Search results, curated movie lists, and curated episode lists need a different contract.

## Facts

### Supported granularities

| Content granularity | Documented endpoint | Supported contract |
| --- | --- | --- |
| Movie | `GET /3/movie/{movie_id}/watch/providers` | Yes. Availability by country and provider. [TMDB movie watch providers](https://developer.themoviedb.org/reference/movie-watch-providers) |
| TV series | `GET /3/tv/{series_id}/watch/providers` | Yes. Availability by country and provider for the show. [TMDB TV series watch providers](https://developer.themoviedb.org/reference/tv-series-watch-providers) |
| TV season | `GET /3/tv/{series_id}/season/{season_number}/watch/providers` | Yes. Availability by country and provider for the season. [TMDB TV season watch providers](https://developer.themoviedb.org/reference/tv-season-watch-providers) |
| TV episode | None in the current documented API reference/OpenAPI | No documented episode-level watch-provider operation. The episode namespace documents episode details and related resources, while the provider operation exists under the season namespace. [TMDB TV episode details](https://developer.themoviedb.org/reference/tv-episode-details), [TMDB OpenAPI](https://developer.themoviedb.org/openapi) |

The absence of an episode endpoint is a statement about the current documented public API, not proof that JustWatch itself lacks episode-level data internally.

### Response shape and categories

The movie, series, and season operations return a top-level resource `id` and a `results` object keyed by ISO 3166-1 country code. A country entry contains:

- `link`: a TMDB watch-page URL for that title and locale;
- zero or more monetization-category arrays;
- provider entries with `logo_path`, `provider_id`, `provider_name`, and `display_priority`.

The category names exposed by TMDB are:

- `flatrate`: subscription streaming;
- `free`: free availability;
- `ads`: ad-supported availability;
- `rent`: rental;
- `buy`: purchase.

TMDB lists exactly these five values for `with_watch_monetization_types` on both Discover endpoints, and its watch-provider documentation describes the underlying data as streaming, rental, and purchase availability. [TMDB Discover Movie](https://developer.themoviedb.org/reference/discover-movie), [TMDB Discover TV](https://developer.themoviedb.org/reference/discover-tv), [TMDB movie watch providers](https://developer.themoviedb.org/reference/movie-watch-providers)

Category arrays are optional per country. The application must not assume that all five keys exist, that a provider appears in only one category, or that every TMDB-supported country has availability for every title. The authoritative list of countries for which TMDB has OTT data is available from `GET /3/watch/providers/regions`. [TMDB available regions](https://developer.themoviedb.org/reference/watch-providers-available-regions)

Provider catalogs can be obtained separately for movies and TV with `GET /3/watch/providers/movie` and `GET /3/watch/providers/tv`; both accept `watch_region`. These endpoints supply the provider IDs needed for Discover filters and region-specific display priority. [TMDB movie provider list](https://developer.themoviedb.org/reference/watch-providers-movie-list), [TMDB TV provider list](https://developer.themoviedb.org/reference/watch-provider-tv-list)

### Link semantics

TMDB explicitly states that watch-provider responses do **not** return full deep links. It instructs clients to link to the provided TMDB URL, which can then provide the actual outbound links to the content. [TMDB movie watch providers](https://developer.themoviedb.org/reference/movie-watch-providers)

The official response examples use URLs shaped like:

```text
https://www.themoviedb.org/movie/{id}-{slug}/watch?locale={country}
https://www.themoviedb.org/tv/{id}-{slug}/watch?locale={country}
```

Therefore:

- `link` is not a JustWatch deep link;
- `link` is not a guaranteed provider playback URL;
- the application must not construct JustWatch URLs from TMDB IDs or slugs;
- the safe outbound action is to use the exact `link` returned by TMDB and label it as a TMDB watch page, such as "View watch options".

This differs from JustWatch's own partner API, which can return a country-specific `full_path` on justwatch.com. That first-party JustWatch contract applies to direct JustWatch integrations and does not change the semantics of TMDB's response. [JustWatch API documentation](https://apis.justwatch.com/docs/api/)

### TMDB attribution and terms

TMDB's API Terms require applications to use the TMDB logo, keep it less prominent than the application's primary branding, avoid implying endorsement, and display a prominent notice identifying TMDB use. The terms provide a notice template of: "This [website, program, service, application, product] uses TMDB and the TMDB APIs but is not endorsed, certified, or otherwise approved by TMDB." [TMDB API Terms of Use, section 3](https://www.themoviedb.org/api-terms-of-use)

TMDB's current FAQ additionally says attribution belongs in an About or Credits type section, requires an approved TMDB logo, and gives the shorter notice: "This product uses the TMDB API but is not endorsed or certified by TMDB." It also prohibits altering the logo's color outside approved colors, aspect ratio, orientation, or rotation. [TMDB FAQ: attribution and branding](https://developer.themoviedb.org/docs/faq), [TMDB logos and attribution](https://www.themoviedb.org/about/logos-attribution)

For this website, the conservative notice that satisfies the more explicit API Terms wording is:

> This website uses TMDB and the TMDB APIs but is not endorsed, certified, or otherwise approved by TMDB.

The notice and an approved, unmodified TMDB logo should appear in an About or Credits surface. Provider UI should not imply that TMDB or JustWatch endorses the application.

The API Terms grant the standard API license subject to attribution, prohibit commercial use without a separate written agreement, prohibit caching TMDB information for longer than six months, and allow TMDB to impose limits or terminate access. The public app must remain non-commercial unless a commercial agreement is obtained, and operational caches must expire well before the six-month maximum. [TMDB API Terms of Use, sections 1, 2, 3, and 5](https://www.themoviedb.org/api-terms-of-use)

### JustWatch attribution

Every TMDB movie, series, and season watch-provider page states that the data is powered by TMDB's partnership with JustWatch and that attribution to JustWatch is mandatory; non-compliance may cause API access to be revoked. [TMDB movie watch providers](https://developer.themoviedb.org/reference/movie-watch-providers), [TMDB TV series watch providers](https://developer.themoviedb.org/reference/tv-series-watch-providers), [TMDB TV season watch providers](https://developer.themoviedb.org/reference/tv-season-watch-providers)

Those TMDB pages do not prescribe exact JustWatch copy, logo artwork, placement, or a link target. A conservative implementation is to display **"Streaming availability data provided by JustWatch"** anywhere provider data is rendered, with a persistent copy in Credits as well. This sentence is a recommendation, not an official mandated phrase.

JustWatch publishes more specific link and attribution rules for data obtained through its own commercial API, but those rules describe direct JustWatch partner integrations. They should not be silently treated as the TMDB contract. [JustWatch API documentation](https://apis.justwatch.com/docs/api/)

### Discover-level filtering

Both `GET /3/discover/movie` and `GET /3/discover/tv` support:

- `watch_region`, used with provider or monetization filters;
- `with_watch_providers`, accepting provider IDs with comma as `AND` and pipe as `OR`;
- `without_watch_providers`;
- `with_watch_monetization_types`, accepting `flatrate`, `free`, `ads`, `rent`, and `buy`, also with comma as `AND` and pipe as `OR`.

TMDB documents comma-separated values as `AND` and pipe-separated values as `OR` for supported Discover filters. [TMDB Discover Movie](https://developer.themoviedb.org/reference/discover-movie), [TMDB Discover TV](https://developer.themoviedb.org/reference/discover-tv)

These capabilities apply to **movie and TV-series discovery**, not seasons or episodes. They are suitable for filtering TMDB-generated discovery lists by the user's country before pagination. They do not return the provider categories, logos, or `link` in each Discover result; displaying those details still requires the corresponding per-title watch-provider operation.

The separate `region` parameter is not a substitute for `watch_region`. On movie Search and Discover, `region` affects regional release-date presentation/filtering, whereas `watch_region` scopes provider and monetization filters. [TMDB region support](https://developer.themoviedb.org/docs/region-support), [TMDB Discover Movie](https://developer.themoviedb.org/reference/discover-movie)

TMDB's title Search endpoints do not document `with_watch_providers`, `with_watch_monetization_types`, or `watch_region` provider filtering. Search can identify candidate movies or shows, but provider filtering cannot be pushed into those Search calls. [TMDB movie search](https://developer.themoviedb.org/reference/search-movie), [TMDB TV search](https://developer.themoviedb.org/reference/search-tv)

## Constraints for this application

1. **No episode-level availability claim.** An `EpisodeItem` may carry `availabilityScope: "season" | "series"`, with the associated series ID and optional season number. UI copy must say "Season availability in {country}" or "Series availability in {country}".
2. **No fabricated JustWatch link.** Preserve and use the exact TMDB `link`; do not derive a justwatch.com URL.
3. **Filtering and display are different operations.** Discover can filter movies and shows without fan-out, but provider badges and categories require detail loading.
4. **Mixed thematic lists have no common provider query.** Movies can be provider-filtered through `discover/movie`; episodes cannot. A global provider filter over curated movies plus curated episodes would require cached per-item/per-season availability or would have incomplete semantics.
5. **Search cannot promise provider filtering across the full result set.** Applying a provider filter only to the currently loaded search page would be misleading and pagination-dependent.
6. **Region is mandatory context for meaningful provider results.** Validate the selected country against available regions and keep provider caches keyed by country, media scope, and resource identity.
7. **Attribution is a release requirement.** TMDB branding/notice and visible JustWatch source attribution must be acceptance criteria, not deferred polish.
8. **Commercialization changes the license.** Ads, paid access, affiliate revenue, or another revenue-generating use should trigger legal/licensing review and a commercial agreement with TMDB before launch.

## Recommendations

### List surfaces

- For TMDB-generated movie or series theme lists, apply `watch_region` plus `with_watch_providers` and/or `with_watch_monetization_types` directly in Discover. This gives correct server-side filtering and pagination without one provider request per result.
- Treat the Discover result as "matches availability criteria", not as provider detail. Do not render provider logos or a provider-specific category until provider details have been loaded.
- For the MVP, omit provider filtering from curated episode lists. Show season or series availability in the episode detail view instead.
- If a mixed `All` list combines movies and episodes, either disable provider filters for `All` or explicitly filter only the movie subset and label that limitation. The cleaner contract is to require `mediaKind=movie` before enabling provider filters.
- Obtain selectable provider IDs from the region-filtered movie/TV provider-list endpoints. Keep movie and TV provider catalogs separate because coverage and priority can differ.

### Detail surfaces

- Movie detail: fetch movie watch providers and select the user's country entry.
- Series detail: fetch series watch providers and select the user's country entry.
- Episode detail: prefer the season watch-provider endpoint. If the season has no country entry, optionally fall back to series-level data, clearly changing the label from season to series availability.
- Render available categories in TMDB's `display_priority` order and tolerate absent category arrays, absent logos, duplicate providers across categories, and absent country entries.
- Use the returned TMDB `link` for the outbound "View watch options" action. Do not label the action "Open JustWatch".
- Place "Streaming availability data provided by JustWatch" adjacent to the provider block and include TMDB plus JustWatch credits in the site's Credits/About surface.

### Suggested normalized contract

```ts
type AvailabilityScope = "movie" | "series" | "season";

type ProviderCategory = "flatrate" | "free" | "ads" | "rent" | "buy";

type Provider = {
  id: number;
  name: string;
  logoPath: string | null;
  displayPriority: number;
};

type RegionalAvailability = {
  country: string;
  scope: AvailabilityScope;
  tmdbWatchUrl: string;
  categories: Partial<Record<ProviderCategory, Provider[]>>;
  attribution: "JustWatch";
};
```

For an episode, the containing application model should attach `RegionalAvailability` obtained from the season or series and must retain its actual `scope`.

## Explicit unknowns

1. TMDB does not define an official exact phrase, artwork requirement, placement, or link target for JustWatch attribution on the cited watch-provider pages. The recommended visible sentence should be validated with TMDB support before a high-traffic or commercial launch.
2. The public docs do not promise that each provider category is exhaustive, mutually exclusive, updated on a particular schedule, or equivalent to entitlement on every plan/device. The UI should present the data as availability information, not a contractual guarantee.
3. The public docs do not specify whether season data always overrides series data, how conflicts are reconciled, or whether an absent season country entry means unavailable versus unknown. The proposed series fallback must be visibly labeled as broader-scope information.
4. Discover documentation confirms filtering semantics but does not state that provider arrays are embedded in Discover results. Current result schemas do not expose them; this should be guarded by contract tests in case the API evolves.
5. The TMDB terms are legal terms that can change. This note is engineering research, not legal advice; licensing and attribution should be rechecked before production launch or monetization.
