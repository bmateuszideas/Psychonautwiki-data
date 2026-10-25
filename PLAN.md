# Build plan — PsychonautWiki full snapshot

Goal: a **100%-coverage snapshot** of PsychonautWiki article content in this
repository, refreshable on demand. This file tracks the pipeline design,
verified facts, and progress.

## Verified facts (checked live, 2026-10)

| Assumption | Verification |
|---|---|
| MediaWiki `action=query` (`list=allpages`) | ❌ blocked — HTTP 418 anti-bot, unusable |
| MediaWiki `action=query&titles=X&prop=revisions` (single page content) | ✅ works with a browser User-Agent |
| MediaWiki `action=parse` (rendered HTML) | ✅ works (used by `pypsychonaut.fetch_page_text`) |
| Official sitemap index in `robots.txt` | ✅ `https://psychonautwiki.org/sitemap/sitemap-index-mediawiki-pw_.xml`, 25 namespace files |
| Article namespace (NS_0) page count via sitemap | ✅ **4124 URLs** (wiki statistics: 1911 "articles", 18700 pages total incl. talk/user/template) |
| Sitemap `lastmod` | ✅ present per namespace file — freshness signal for incremental refresh |
| `Special:Export` batch POST (`pages` newline-separated, `curonly=1`) | ✅ verified with 5 pages: returns `export-0.11` XML with full wikitext (~370 KB for 5 pages) |
| GraphQL API `api.psychonautwiki.org` | ✅ 373 substances with structured data; note: no top-level `effects` query field — effect data comes nested per substance |
| Wiki license | ✅ **CC BY-SA 4.0** (footer of every page) — redistribution allowed with attribution, share-alike |
| `pypsychonaut` transport (curl_cffi + FlareSolverr + Wayback fallback) | ✅ works; covers Cloudflare-protected HTML pages |
| `pypsychonaut.list_wiki_pages()` (sitemapper) | ⚠️ broken — `sitemapper` API changed (`discover` gone); the plan reads the sitemap directly instead |
| `wiki-scripts/wiki_dump_downloader` | ✅ same `Special:Export` approach; reference implementation (Perl) |

## Pipeline (order of execution)

1. **Enumerate** — download sitemap index + all namespace files; parse URLs →
   `meta/sitemap.json` (title, namespace, lastmod). Baseline scope: **NS_0**
   (4124 pages); optional later: talk/template/file namespaces.
2. **Fetch** — batch titles (50/request) → `Special:Export` POST → append to
   `dump/batch-NNN.xml.gz` with retry/backoff and a crawl delay.
3. **Extract** — parse dump XML → `pages/<ns>/<Title>.wikitext` + metadata
   (revision id, timestamp, sha256) → `index/pages.json`.
4. **Render** — convert wikitext → markdown
   (`pypandoc`/`wikitextparser` or `action=parse` for templates-heavy pages) →
   `pages/<ns>/<Title>.md`.
5. **GraphQL** — page through all 373 substances → `data/substances.json`
   (doses, durations, RoAs, classes, tolerance, effects per substance).
6. **Index & stats** — build `index/substances.json`, `meta/run.json`
   (timestamp, counts, checksums, sitemap lastmod).
7. **Verify** — cross-check: sitemap titles == extracted pages == indexed
   pages; no empty files; spot-check 10 random pages against live wiki.
8. **Document & publish** — README (done), license notice, commit dump.

## To-do list

- [x] Verify access paths (sitemap, Special:Export, GraphQL, license)
- [x] Write README.md (repository purpose, structure, license, attribution)
- [x] Write PLAN.md (this file)
- [ ] 1. Sitemap enumeration script → `meta/sitemap.json` (NS_0, 4124 pages)
- [ ] 2. Batch fetcher (`Special:Export`, 50/req, delay, retries) → `dump/*.xml.gz`
- [ ] 3. XML → per-page `.wikitext` + `index/pages.json`
- [ ] 4. Wikitext → markdown renderer → `pages/<ns>/*.md`
- [ ] 5. GraphQL dump of all 373 substances → `data/substances.json`
- [ ] 6. Build `meta/run.json` with stats and checksums
- [ ] 7. Verification pass (counts match, spot checks, no empty files)
- [ ] 8. Commit full snapshot + open PR; wire refresh tooling into `pypsychonaut`
- [ ] 9. (Optional) extend to non-article namespaces (talk, templates, files)
- [ ] 10. (Optional) fix `pypsychonaut.list_wiki_pages()` to read the official sitemap directly

## Design decisions

- **Sitemap over `allpages`**: the only reliable, unblocked enumeration; also
  gives `lastmod` for future incremental (only-changed-pages) refreshes.
- **Raw wikitext over HTML**: lossless, diffable, independent of skin/layout.
  Markdown is generated as a convenience layer, never the source of truth.
- **XML dumps kept**: `dump/` is the authoritative artifact; everything in
  `pages/` is derivable from it, so a broken render never loses data.
- **Batching**: 50 titles per export request → the whole NS_0 snapshot costs
  roughly 85 requests; with a 2–3 s delay that is a few minutes of gentle load.
- **License**: CC BY-SA 4.0 requires attribution + share-alike; README carries
  the notice and the repo keeps per-page source URLs in `index/pages.json`.
