# Issue tracker

This repository has no remote issue tracker configured. Use local Markdown under `.scratch/<effort>/`.

For the thematic episodes effort:

- Map: `.scratch/episodios-tematicos/map.md`
- Child tickets: `.scratch/episodios-tematicos/tickets/`
- Closed decisions: `.scratch/episodios-tematicos/tickets/closed/`

Decision tickets use a `## Question` section and record their answer under `## Resolution`.

## Wayfinding operations

- Open tickets live in `.scratch/<effort>/tickets/open/` with `status: open` front matter.
- Closed tickets live in `.scratch/<effort>/tickets/closed/` with `status: closed` front matter.
- A ticket is claimed by setting `claimed_by` to the session identifier, then rereading the file to verify ownership.
- Dependencies use a `blocked_by` front-matter list of ticket filenames. An empty list means the ticket is on the frontier.
- Resolve a ticket by writing its decision under `## Resolution`, setting `status: closed`, moving it to `tickets/closed/`, and adding a one-line link to the map.
- Query the frontier by listing open tickets whose `blocked_by` entries all refer to closed tickets and whose `claimed_by` value is empty.
