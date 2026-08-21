# Thematic episode index API design

Status: approved on 2026-08-20.

## Purpose

Provide 50–200 classified TV and anime episodes for each canonical theme without runtime catalog crawling, a database, a paid episode-search API, or manual curation of the entire catalog. The generated index is served through SvelteKit on Vercel. TVmaze supplies episode metadata, a local open-source model classifies theme relevance offline, AniList verifies anime identity selectively, and TMDB enriches visible results without supplying classifier input.

## Source decision

### TVmaze as the episode corpus

TVmaze is the canonical source for indexed episode identity and classification input because its public API:

- provides a paginated show index intended for local caches;
- returns complete episode lists with titles, summaries, dates, numbering, ratings, images, and source URLs;
- permits public use under CC BY-SA with attribution and ShareAlike;
- requires no application credential for public metadata;
- documents a baseline limit of at least 20 calls per 10 seconds per IP and `429` backoff behavior.

Sources: [TVmaze API](https://www.tvmaze.com/api), [TVmaze plans and licensing](https://www.tvmaze.com/api/plans).

The generated index and any copied or adapted TVmaze metadata must carry clear TVmaze attribution and a compatible CC BY-SA data license.

### TMDB as runtime enrichment

TMDB remains the source for movie discovery, current series artwork, details, regional provider information, and supported outbound watch links. It is not classifier input. Its current API terms prohibit using TMDB content in ML or AI applications, restrict derived indexes, and limit caching to six months without separate permission. The indexer may store an optional TMDB series identity obtained through a non-ML crosswalk, but it must refresh or remove cached TMDB content within the permitted window.

Sources: [TMDB API terms](https://www.themoviedb.org/api-terms-of-use), [TMDB rate limiting](https://developer.themoviedb.org/docs/rate-limiting), [TMDB FAQ and attribution](https://developer.themoviedb.org/docs/faq).

### AniList as targeted anime verification

AniList is used only to verify likely anime series by title, synonyms, year, media type, and format. It is not mirrored or used as a bulk episode corpus. Public metadata queries require no OAuth, but the pipeline must respect response rate-limit headers and the prohibition on data hoarding or mass collection.

Sources: [AniList API documentation](https://github.com/AniList/docs), [AniList rate limiting](https://anilist.gitbook.io/anilist-apiv2-docs/docs/guide/rate-limiting), [AniList terms](https://anilist.gitbook.io/anilist-apiv2-docs/docs/guide/terms-of-use).

### Rejected primary sources

- **TMDB-only index:** rejected because TMDB lacks global episode search and its current terms do not permit the proposed semantic classification or derived index without written authorization.
- **TheTVDB-first index:** technically capable of globally enumerating episodes, but its project registration, API licensing, attribution, and redistribution restrictions add avoidable uncertainty. It remains a fallback only if TVmaze coverage proves inadequate. [TheTVDB API licensing](https://thetvdb.com/api-information), [TheTVDB terms](https://thetvdb.com/tos).
- **Trakt:** rejected for the MVP because registering a new API application currently requires Trakt VIP, despite no separate per-request Public API charge.

## Architecture

The system has separate offline and runtime paths.

### Offline indexing path

1. A local command or scheduled GitHub Action obtains the TVmaze show index.
2. A deterministic selector chooses approximately 3,000 scripted series across popularity, decade, language, country, and format strata.
3. The indexer fetches complete episode lists with specials when useful.
4. It normalizes titles and summaries, generates broad theme candidates, and runs local semantic inference.
5. It selectively checks likely anime identities against AniList.
6. It optionally crosswalks series to TMDB without using TMDB content in classification.
7. It applies quality, confidence, diversity, and size gates.
8. It writes versioned theme shards and a catalog manifest.
9. A scheduled run opens a review PR. It never publishes a partial or unreviewed output directly.

### Runtime path

1. The browser requests an app-owned SvelteKit endpoint.
2. The server loads only the requested theme shard.
3. It validates and applies local filters, sorting, and pagination.
4. It returns 24 indexed episodes and facet counts.
5. A separate bounded request enriches only visible results with current TMDB artwork or provider data where a mapping exists.

No model, full-catalog scan, or TVmaze crawl runs in a Vercel request.

## Corpus selection

The initial corpus targets roughly 3,000 scripted series and enough episode summaries to yield 50–200 published associations per theme. Selection is stratified rather than globally popularity-only so older shows, animation, anime, non-English series, and multiple countries remain represented.

Initial exclusions include news, talk shows, unscripted reality, and other formats where an episode-theme recommendation would not match the product's narrative-media purpose. The selector records why each series entered or left the corpus and remains deterministic for the same source snapshot and configuration.

## Canonical identity

The canonical episode key is namespaced to TVmaze:

```ts
type EpisodeKey = `tvmaze:${number}`;

type IndexedEpisodeIdentity = {
  key: EpisodeKey;
  tvmazeEpisodeId: number;
  tvmazeSeriesId: number;
  tmdbSeriesId?: number;
  seasonNumber?: number;
  episodeNumber?: number;
};
```

Season and episode numbers are descriptive coordinates and may be absent for specials. They are not canonical identity. Numbering conflicts are preserved as metadata rather than guessed. One canonical episode can reference multiple themes.

## Indexed record

An indexed episode contains:

- canonical and source identities;
- series and episode titles;
- source URL and attribution;
- normalized summary or permitted excerpt;
- air date when present;
- episode rating when present;
- inherited series genres, language, country, and source popularity weight;
- editorial format: `live-action`, `animation`, or verified `anime`;
- optional TMDB series mapping with mapping confidence and review state;
- one or more theme matches.

Each theme match contains:

- canonical theme slug;
- normalized theme score;
- confidence: `high`, `medium`, or unpublished `low`;
- matched phrases or evidence spans;
- classifier version and evaluation date.

## Classification pipeline

### Candidate generation

Each theme supplies its canonical definition, aliases, compound expressions, and negative terms. Literal matching creates high-recall candidates but cannot independently publish an episode.

### Semantic inference

An open-source model runs offline against normalized episode title and summary text. It performs inference only; the project does not train a model on third-party metadata. The score combines semantic similarity, explicit phrases, negative context, title-versus-summary support, and description quality.

### Confidence

- `high`: the available text strongly supports the theme as a central plot, setting, or event.
- `medium`: the theme supports a substantial subplot or event but is not the episode's sole focus.
- `low`: weak, incidental, title-only, contradictory, or insufficiently described; retained only for audit.

Only `high` and `medium` records are published. A title match alone cannot exceed `low`.

### Anime verification

Animation associated with Japan is only an anime candidate. The `anime` format requires a confident targeted AniList match using titles, synonyms, start year, type, and format. Ambiguous matches remain `animation` and are reported for review.

### Ranking and diversity

The default order uses thematic score followed by deterministic diversity adjustments and a namespaced identity tie-breaker. The first 24 results must include at least five distinct series. Ranking reduces repeated franchises and preserves useful format, decade, and country variation.

## API contract

### Theme episodes

```text
GET /api/themes/:slug/episodes
```

Supported query parameters:

- `format`;
- `genre`;
- `yearFrom` and `yearTo`;
- `minRating`;
- `confidence`;
- `sort`;
- `page`.

Page size is fixed at 24 for the MVP. Unknown parameters and unsupported values are rejected rather than ignored.

Supported sorts:

- `relevance`: local thematic score;
- `popularity`: inherited TVmaze series weight, labeled **Series popularity**;
- `rating`: episode rating;
- `newest` and `oldest`: episode air date;
- `title`: normalized episode title.

Missing values always sort last. Every comparator ends with the canonical identity so pagination is stable.

The response contains catalog and classifier versions, page metadata, total matching results, facet counts, and normalized episode records.

### Availability

```text
GET /api/availability?tmdbSeriesId=:id&season=:number&country=:code
```

Availability is enriched only for visible results and labeled at its actual TMDB scope: series or season. It never claims episode-specific availability. The MVP does not provide a global provider filter for indexed episodes because runtime enrichment cannot reliably filter the complete local catalog. Movies retain provider filtering through TMDB movie discovery.

## Update workflow

A weekly GitHub Action:

1. loads the last successful index manifest and source checkpoints;
2. detects new or changed TVmaze shows;
3. refreshes only affected episodes where possible;
4. reprocesses all records when classifier or theme definitions change;
5. performs targeted AniList verification for new anime candidates;
6. refreshes optional TMDB mappings within allowed cache windows;
7. generates shards and an audit report in a temporary output directory;
8. runs every validation against that complete output;
9. opens a PR containing changed generated artifacts and metrics.

TVmaze and AniList public reads require no repository secret. The optional crosswalk uses `TMDB_ACCESS_TOKEN` configured directly in GitHub and Vercel secrets. The workflow and application never inspect or copy the developer's local `.env`.

## Versioning

The catalog manifest includes:

```ts
type CatalogManifest = {
  schemaVersion: number;
  catalogVersion: string;
  classifierVersion: string;
  generatedAt: string;
  sourceUpdatedAt: string;
  license: "CC-BY-SA-4.0";
};
```

Every theme response returns these versions. Schema changes require an explicit migration. Generated output is deterministic for the same source snapshot, configuration, and model version.

## Failure behavior

- **TVmaze unavailable:** indexing fails without changing production.
- **AniList unavailable:** new anime candidates remain pending; existing verified classifications are not silently downgraded.
- **TMDB unavailable:** indexed episodes remain visible; optional images and provider sections show an unavailable state.
- **Rate limiting:** jobs honor `Retry-After`, use bounded concurrency, checkpoint progress, and resume safely.
- **Invalid or corrupt shard:** the API returns a safe structured error; the UI offers retry and does not render partial records.
- **Removed or renumbered source episode:** the record is flagged in the PR report before removal or coordinate changes are accepted.
- **Failed validation:** no generated file is promoted.

Production always serves the last valid merged catalog.

## Quality gates

Every published theme must satisfy:

- between 50 and 200 `high` or `medium` associations;
- at least five `high` associations;
- at least five distinct series in the first 24 results;
- no duplicate canonical episode identity;
- valid source URL and TVmaze attribution for every record;
- no title-only acceptance;
- deterministic rank and pagination;
- valid schema, catalog, and classifier versions;
- no unresolved identity collision.

## Testing

### Classifier benchmark

A versioned editorial benchmark includes positive and hard-negative examples for every theme. Initial targets are at least 90% precision for `high` and 75% precision for `medium`. Recall is monitored against known-positive fixtures and coverage drift rather than claimed as global catalog recall.

### Pipeline tests

Tests cover HTML normalization, aliases, compound phrases, negative terms, score thresholds, missing descriptions, specials, missing numbers and dates, multi-theme records, deduplication, deterministic output, checkpoints, `429` recovery, source deletion, identity crosswalk conflicts, licensing metadata, and secret redaction.

### API tests

Tests cover every filter and sort, invalid parameters, stable pagination, missing values, empty results, corrupt shards, unavailable enrichment, facet counts, catalog versions, and safe error objects.

### Operational targets

- local index response below 200 ms under normal conditions;
- compressed theme shard below 500 KB;
- no external call to list indexed episodes;
- enrichment limited to the 24 visible results;
- last valid catalog remains available after any failed refresh;
- no credential appears in generated files, logs, API responses, browser bundles, or rendered HTML.

## Review report

Every update PR includes per-theme counts, confidence changes, additions, removals, coverage regressions, unresolved identities, format distribution, series concentration, classifier version changes, and a review sample from each modified theme. All removals and confidence promotions are explicitly visible.

## MVP boundaries

- No database, vector database, runtime model, or user account.
- No global episode provider filter.
- No claim of episode-specific streaming availability.
- No TMDB content in semantic inference or model training.
- No bulk AniList mirror.
- No automatic merge of generated catalog changes.
- No manual requirement to curate the complete episode catalog.
