# Canonical theme manifest

Status: approved and amended on 2026-08-20.

## Purpose

This document is the approved vocabulary and editorial quality contract for the 30-theme MVP. It deliberately does not choose TMDB keyword IDs, movie IDs, or episode keys. Those identifiers belong to a separate research and curation ticket.

## Manifest shape

Each theme has one stable record with:

- `slug`: permanent URL and identity, lowercase kebab case;
- `label`: English interface label;
- `group`: one canonical editorial group;
- `aliases`: English search and matching phrases that do not change the displayed label;
- `description`: one plain-English sentence defining the theme boundary;
- `keywordExpression`: vetted TMDB keyword IDs and their AND or OR relationship;
- `curatedMovieIds`: verified TMDB movie identities;
- `curatedEpisodes`: optional editorial overrides or seeds using canonical TVmaze episode identities;
- `featuredRank`: optional local ordering used only for curated thematic relevance;
- `reviewedAt`: date of the latest editorial verification.

The slug is stable after launch. A label change does not change the slug. An item associated with multiple themes is stored once and references multiple theme slugs.

## Canonical themes

| Group | Slug | Label | Aliases |
| --- | --- | --- | --- |
| Annual observances | `christmas` | Christmas | Christmas Eve, holiday Christmas |
| Annual observances | `halloween` | Halloween | spooky season, trick or treat |
| Annual observances | `valentines-day` | Valentine's Day | Valentine, romance holiday |
| Annual observances | `thanksgiving` | Thanksgiving | Thanksgiving dinner, turkey day |
| Annual observances | `new-years-eve` | New Year's Eve | New Year, countdown, year-end |
| Annual observances | `easter` | Easter | Easter Sunday, Easter holiday |
| Annual observances | `mothers-day` | Mother's Day | mothers, mom celebration |
| Annual observances | `fathers-day` | Father's Day | fathers, dad celebration |
| Personal celebrations | `birthday` | Birthday | birthday party, surprise party |
| Personal celebrations | `wedding` | Wedding | marriage, wedding ceremony |
| Personal celebrations | `graduation` | Graduation | commencement, graduation ceremony |
| Personal celebrations | `new-baby-pregnancy` | New Baby & Pregnancy | pregnancy, childbirth, new baby |
| Seasons and school | `summer-vacation` | Summer Vacation | summer break, school vacation |
| Seasons and school | `winter-snow-day` | Winter & Snow Day | winter, snow day, blizzard |
| Seasons and school | `spring` | Spring | springtime, spring season |
| Seasons and school | `autumn-fall` | Autumn & Fall | autumn, fall season |
| Seasons and school | `back-to-school` | Back to School | first day of school, new school year |
| Seasons and school | `school-festival` | School Festival | cultural festival, school fair |
| Seasons and school | `prom-school-dance` | Prom & School Dance | prom night, school dance, formal dance |
| Travel and outings | `camping-trip` | Camping Trip | camping, campout, campsite |
| Travel and outings | `beach-day` | Beach Day | beach trip, seaside day |
| Travel and outings | `road-trip` | Road Trip | car trip, cross-country drive |
| Travel and outings | `vacation-trip` | Vacation Trip | holiday trip, family vacation, getaway |
| Travel and outings | `festival-fair` | Festival & Fair | fairground, carnival, local festival |
| Relationships and gatherings | `secret-santa-gift-exchange` | Secret Santa & Gift Exchange | Secret Santa, gift swap, gift exchange |
| Relationships and gatherings | `reunion` | Reunion | family reunion, class reunion, homecoming |
| Relationships and gatherings | `breakup` | Breakup | separation, relationship ending |
| Relationships and gatherings | `first-date` | First Date | first romantic date, blind date |
| Thrills and contests | `ghost-haunted-house` | Ghosts & Haunted Houses | ghost story, haunted house, haunting |
| Thrills and contests | `tournament-competition` | Tournament & Competition | contest, championship, competition arc |

## Theme boundary rules

- The item must depict the theme as a material part of its plot, setting, or event. A passing mention, decoration, or background date is insufficient.
- A curated association requires a short editorial rationale specific to that theme.
- A movie or episode may belong to multiple themes when each association independently meets the boundary.
- Anime is an editorial format classification, not a media kind or a TMDB type. Anime movies remain movies; anime episodes remain episodes.
- Season `0` specials are allowed when the compound episode key resolves and the item meets the same thematic standard.
- Multipart broadcasts use the TMDB episode numbering returned by the season response. The catalog does not invent combined episode identities.

## Minimum quality bar per theme

Every theme must meet all of these conditions before the MVP is considered release-ready:

1. At least eight verified thematic items are available on first load.
2. At least five of the first eight are indexed episode references with `high` or `medium` thematic confidence.
3. At least two are verified movie references or vetted movie-discovery results.
4. The first eight items contain episodes from at least two distinct series.
5. No more than two of the first eight items come from the same series or franchise.
6. Every generated episode association has evidence fields and a classifier version; every curated override also has an editorial rationale.
7. Every episode resolves through its canonical TVmaze episode identity; season and episode numbers are descriptive coordinates, and optional TMDB series mappings are verified separately.
8. Missing, moved, or renamed records are flagged by drift guards and excluded until reviewed.
9. Keyword expressions are stored as IDs and spot-checked against at least the first two TMDB result pages for obvious false positives.
10. Items with missing artwork, dates, ratings, or provider data remain valid when the thematic identity resolves; the UI treats those fields as optional.

These thresholds define the release bar, not the maximum catalog size. Themes may contain more items after their first eight entries.

## Catalog-wide quality rules

- All 30 slugs are unique and every referenced slug exists in this table.
- Canonical TVmaze episode identities are unique globally; one record may reference multiple themes.
- Movie and series identities are namespaced so numeric TMDB ID collisions cannot merge records.
- Theme ordering is explicit and stable. TMDB popularity does not define curated thematic relevance.
- Anime classification is reviewed editorially and cannot be inferred solely from Japanese language or origin.
- Provider availability is never part of thematic membership.
- Editorial records are reviewed before release and at least every 180 days afterward.

## Follow-up ticket

After approval, a separate task will populate and validate TMDB keyword IDs and curated movie IDs, then generate episode associations through the approved thematic episode index. That work must use the quality bar above and may not weaken it silently to make a theme appear complete.
