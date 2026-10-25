# TODO.md — workflow budowy pełnego snapshotu PsychonautWiki

**Definicja ukończenia (Definition of Done):** każda strona artykułowa
PsychonautWiki (namespace 0, ~4124 strony wg oficjalnej sitemapy) istnieje w
tym repozytorium jako `.wikitext` + `.md`, wszystkie dane strukturalne z
GraphQL są w `data/*.json`, indeksy są kompletne i spójne, a weryfikacja k.7
przechodzi bez błędów. Wtedy — i tylko wtedy — ten plik jest w całości
odhaczony i produkt jest gotowy do użycia przez agenta AI.

## Użyte biblioteki i repozytoria — co, skąd, po co

| Narzędzie / biblioteka | Skąd | Do czego w pipeline |
|---|---|---|
| `pypsychonaut.Transport` + `CloudflareSession` (unblock_requests, curl_cffi) | repo **pypsychonaut** (dev) | transport HTTP dla wszystkich żądań do wiki: TLS impersonation Chrome + fallbacki FlareSolverr/Wayback; `get_html()` do sitemapy |
| `pypsychonaut` GraphQL client (`Transport.graphql`) | repo **pypsychonaut** (dev) | pobranie wszystkich substancji z `api.psychonautwiki.org` (query `substances(limit, offset)` z pełnymi polami) |
| `pypsychonaut.fetch_page_text()` (action=parse) | repo **pypsychonaut** (dev) | fallback renderowania markdownu dla stron z ciężkimi szablonami, gdy konwersja statyczna zawodzi |
| `wiki-scripts/wiki_dump_downloader` | repo **wiki-scripts** (master) | wzorzec: batchowy `Special:Export` przez POST (`pages` newline-separated, `curonly=1`); nie uruchamiamy perla — odtwarzamy ten sam protokół w Pythonie |
| `wiki-scripts/wiki_dump_parser` | repo **wiki-scripts** (master) | wzorzec parsowania dumpa XML; finalnie używamy `mwxml`/`xml.etree` (prościej) |
| `urllib`/stdlib + `gzip` | Python stdlib | sitemapa (XML gzip) i `Special:Export` — zweryfikowane: działa z UA przeglądarki, bez bibliotek zewnętrznych |
| `wikitextparser` | PyPI | parsowanie wikitextu → markdown (szablony, linki, sekcje) |
| `pypandoc` (opcja) | PyPI | alternatywna konwersja wikitext→md przez pandoc |
| `mwxml` | PyPI | szybki strumieniowy parser MediaWiki dump XML |
| `tqdm` | PyPI | pasek postępu |

Uwaga: `pypsychonaut.list_wiki_pages()` (pakiet `sitemapper`) jest zepsute —
API `sitemapper` się zmieniło. Krok 1 czyta sitemapę bezpośrednio stdlibem;
naprawa `list_wiki_pages()` jest osobną pozycją (k. 8).

## Workflow — kolejność wykonania

### Krok 1. Wyliczenie wszystkich stron (sitemap) — `scripts/01_sitemap.py`
- [ ] Pobrać `https://psychonautwiki.org/sitemap/sitemap-index-mediawiki-pw_.xml`
- [ ] Rozpakować pliki per-namespace (gzip), wyparsować `<loc>` i `<lastmod>`
- [ ] Zapisać `meta/sitemap.json` (title, ns, url, lastmod); baseline: **NS_0**
- [ ] Sanity: liczba URL-i NS_0 ≈ 4124; brak duplikatów tytułów

### Krok 2. Pobranie treści (Special:Export, batche) — `scripts/02_fetch.py`
- [ ] POST `https://psychonautwiki.org/wiki/Special:Export`, `pages` = 50 tytułów (newline), `curonly=1`
- [ ] Retry z backoffem (3×), pauza 2–3 s między batchami, UA przeglądarki
- [ ] Zapis: `dump/batch-NNN.xml.gz` + `meta/progress.json` (do wznowienia po przerwaniu)
- [ ] Sanity: liczba `<page>` w dumpach == liczba tytułów z sitemapy

### Krok 3. Ekstrakcja stron z XML — `scripts/03_extract.py`
- [ ] Streamować `dump/*.xml.gz` (mwxml lub ElementTree iterparse)
- [ ] Dla każdej strony: `pages/ns0/<title>.wikitext` (sanityzacja nazw plików) + rekord w `index/pages.json` (title, ns, source_url, revision_id, revision_timestamp, sha256)
- [ ] Sanity: brak pustych plików; tytuły wyekstrahowane == tytuły z sitemapy

### Krok 4. Render wikitext → markdown — `scripts/04_render.py`
- [ ] `wikitextparser` → markdown (sekcje, linki, listy, tabele)
- [ ] Frontmatter YAML w każdym `.md`: title, source_url, license (CC BY-SA 4.0), revision_timestamp
- [ ] Fallback dla stron szablono-ciężkich: `fetch_page_text()` (action=parse) i zapis wyniku z flagą `render: fallback-parse`
- [ ] Sanity: każdy `.wikitext` ma odpowiadający `.md`; brak pustych `.md`

### Krok 5. Dane strukturalne z GraphQL — `scripts/05_graphql.py`
- [ ] Stronowanie `substances(limit: 100, offset: n)` do wyczerpania (~373)
- [ ] Pełne pola: name, class (psychoactive/chemical), roas (dose brackets, duration, bioavailability), tolerance, effects, addictivePotential, toxicity, crossTolerances
- [ ] Zapis: `data/substances.json`; efekty per substancja → `data/effects.json`; `query_reagents` z pypsychonaut → `data/reagents.json`
- [ ] Sanity: liczba substancji == ostatnia strona offsetów; brak `errors` w odpowiedziach

### Krok 6. Indeksy i metadane — `scripts/06_index.py`
- [ ] `index/pages.json` (z k.3) posortowany, z checksumami
- [ ] `index/substances.json`: substancja → pliki (`pages/ns0/X.md`, `data/substances.json` entry), klasy chemiczne/psychoaktywne
- [ ] `index/effects.json`: efekt → plik strony + substancje zgłaszające
- [ ] `meta/run.json`: timestamp runu, lastmod sitemapy, liczby (strony/substancje/efekty), wersja pipeline'u, statystyki

### Krok 7. Weryfikacja całości — `scripts/07_verify.py`
- [ ] Zestawienie trzech liczb: sitemap titles == pages wyekstrahowane == pliki `.wikitext`; raport różnic
- [ ] Każdy `.md` ma frontmatter i `source_url`
- [ ] Losowa próba 20 stron: porównanie treści `.wikitext` z żywym `Special:Export` (checksum)
- [ ] Wszystkie rekordy GraphQL mają poprawny JSON i niepustą nazwę
- [ ] Wynik: `meta/verify-report.json` (pass/fail per check) — **fail blokuje uznanie dumpa za ukończony**

### Krok 8. Publikacja i domknięcie
- [ ] Licencja: nota CC BY-SA 4.0 w README (jest) + `source_url` w każdym rekordzie indeksu
- [ ] Commit pełnego snapshotu, PR, review, merge
- [ ] `meta/run.json` wskazuje datę i świeżość snapshotu

### Krok 9. (po merge'u, opcjonalne rozszerzenia)
- [ ] Naprawić `pypsychonaut.list_wiki_pages()` (czytać sitemapę stdlibem, bez `sitemapper`)
- [ ] Pozostałe namespace (talk, szablony, pliki) — po decyzji scope'u
- [ ] Refresh przyrostowy po `lastmod` z sitemapy

## Zasady bezpieczeństwa pobierania

- Batche 50 stron, pauza 2–3 s, retry 3× z backoffem — całkowity koszt NS_0 ≈ 85 żądań
- `curonly=1` (tylko bieżąca rewizja — bez historii edycji)
- Anulowanie/wznowienie przez `meta/progress.json`

## Postęp

- [x] Weryfikacja ścieżek dostępu (sitemap, Special:Export, GraphQL, licencja CC BY-SA 4.0)
- [x] README.md z docelową strukturą, metodologią i zasadami użycia przez AI
- [x] TODO.md — ten workflow
- [ ] Kroki 1–7 (odhaczane po realizacji, z numerem runu)
- [ ] Krok 8: gotowy produkt — pełny snapshot w main
