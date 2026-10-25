# Psychonautwiki-data

A complete, versioned mirror of the article content of
[PsychonautWiki](https://psychonautwiki.org) — the open, community-run
encyclopaedia of psychoactive substances, their effects, doses, durations,
tolerance, and harm-reduction information.

This repository exists to provide a **100%-coverage, machine-readable snapshot**
of the wiki that can be cloned, indexed, diffed, and used offline for research,
ASR/NER datasets, and harm-reduction tooling — without scraping HTML.

## What is stored here

| Directory | Contents |
|---|---|
| `dump/` | Raw MediaWiki XML dumps (`Special:Export`, `export-0.11`), batched and gzipped — the authoritative, lossless source |
| `pages/<namespace>/` | One page per file: `.wikitext` (raw source) and `.md` (rendered markdown) |
| `data/` | Structured data from the official [GraphQL API](https://api.psychonautwiki.org): substances, RoAs, doses, durations, effects, tolerance, reagents |
| `index/` | `pages.json` (title, namespace, timestamp, checksum), `substances.json`, `effects.json` |
| `meta/` | Dump timestamp, sitemap snapshot, per-run statistics |

Every page in the wiki's article namespace is captured with its **full raw
wikitext** (lossless) plus a markdown rendering for readability.

## Source and how the dump is taken

- **Page list**: the wiki's official MediaWiki sitemap index
  (`/sitemap/sitemap-index-mediawiki-pw_.xml`, announced in `robots.txt`)
  is the authoritative enumeration of all pages. It avoids the
  Cloudflare/anti-bot protection (HTTP 418) that blocks the `list=allpages`
  API module.
- **Page content**: fetched in batches (50–100 pages per request) via
  `Special:Export` with `curonly=1`, which returns standard MediaWiki XML
  (`export-0.11`) including full wikitext — no HTML scraping involved.
- **Structured data**: the public GraphQL API at `api.psychonautwiki.org`
  (~373 substances with doses, durations, effects, classes, tolerance).

Fetching is deliberately slow and batched (a small number of requests per
run), to be a good citizen of the wiki's infrastructure.

Tooling for the full pipeline lives in
[pypsychonaut](https://github.com/bmateuszideas/pypsychonaut) (transport with
Cloudflare bypass, GraphQL client, markdown corpus dumper). See `PLAN.md` for
the build pipeline and current progress.

## License and attribution

All wiki content in this repository originates from
**PsychonautWiki** and is licensed by its authors under the
**Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)**
license, as stated on every page of the wiki.

Redistribution of this data must:

1. credit PsychonautWiki and link to `https://psychonautwiki.org`;
2. state the license (CC BY-SA 4.0) and link to it;
3. be released under the same (or compatible) license.

Code and tooling in this repository are licensed separately by the repository
owner (see `LICENSE` for scripts if added).

## Refreshing the dump

The dump is refreshable at any time by re-running the pipeline (see
`PLAN.md`). Each run records its timestamp and the sitemap `lastmod` dates in
`meta/`, so the age of the snapshot is always explicit.
