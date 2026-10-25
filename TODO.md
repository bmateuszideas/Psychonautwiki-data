# TODO.md — workflow zeskrapowania całej wiedzy z www.psychonautwiki.org

**Cel końcowy (Definition of Done):** cała treść artykułowa strony
`www.psychonautwiki.org` (każda strona z oficjalnej sitemapy, namespace
artykułów, ~4124 strony) jest zapisana w tym repozytorium — surowy wikitext
plus czytelny render — wraz z danymi strukturalnymi z publicznego API GraphQL
serwisu, kompletnymi indeksami i raportem weryfikacji, który przechodzi bez
błędów. Dopiero wtedy ten plik jest w całości odhaczony.

**Źródło danych: wyłącznie serwis www.psychonautwiki.org** (jego sitemapa,
Special:Export i API GraphQL). Żadne repozytorium kodu nie jest źródłem danych —
`pypsychonaut` i `wiki-scripts` to tylko narzędzia/wzorce.

## Zweryfikowane fakty o źródle (testy na żywo)

| Fakt | Status |
|---|---|
| `robots.txt` wskazuje sitemapę: `https://psychonautwiki.org/sitemap/sitemap-index-mediawiki-pw_.xml` | ✅ |
| Sitemapa zawiera 25 plików per-namespace; artykuły (NS_0) = **4124 URL-e** | ✅ |
| MediaWiki API `list=allpages` → HTTP 418 (anti-bot) — nie używamy | ✅ |
| `Special:Export` POST (`pages` newline-separated, `curonly=1`) → pełny XML `export-0.11` z wikitextem (test 5 stron) | ✅ |
| API GraphQL `api.psychonautwiki.org` → 373 substancje ze strukturą (dawki, ROA, efekty, tolerancja) | ✅ |
| Licencja treści serwisu: **CC BY-SA 4.0** (stopka każdej strony) | ✅ |

## Narzędzia — co, skąd, do czego

| Narzędzie | Skąd | Rola w pipeline |
|---|---|---|
| Python stdlib (`urllib`, `gzip`, `xml.etree`) | stdlib | sitemapa i Special:Export — zweryfikowane, wystarczą bez zewnętrznych bibliotek |
| `pypsychonaut.Transport` / `CloudflareSession` | Twoje repo pypsychonaut | gotowy transport HTTP z obejściem Cloudflare — użyty, gdyby wiki zaczęła blokować zwykłe żądania |
| `pypsychonaut` GraphQL client | Twoje repo pypsychonaut | pobranie substancji z api.psychonautwiki.org |
| `pypsychonaut.fetch_page_text()` | Twoje repo pypsychonaut | fallback renderu markdown dla stron z ciężkimi szablonami |
| `wiki-scripts/wiki_dump_downloader` | Twoje repo wiki-scripts | wzorzec protokołu Special:Export (odtworzony w Pythonie, nie uruchamiam perla) |
| `wikitextparser` | PyPI | konwersja wikitext → markdown |
| `mwxml` | PyPI | strumieniowe parsowanie dumpa XML |
| `tqdm` | PyPI | pasek postępu |

Uwaga: `pypsychonaut.list_wiki_pages()` jest obecnie zepsute (API pakietu
`sitemapper` się zmieniło) — krok 1 czyta sitemapę bezpośrednio stdlibem.

## Workflow — kroki do wykonania w tej kolejności

### 1. Wyliczenie wszystkich stron serwisu
- [ ] Pobrać indeks sitemapy + pliki namespace, rozpakować, wyparsować URL-e i `lastmod`
- [ ] Zapisać pełną listę (title, url, ns, lastmod) do pliku roboczego `meta/sitemap.json`
- [ ] Sanity: liczba stron NS_0 ≈ 4124, brak duplikatów

### 2. Pobranie pełnej treści stron (Special:Export)
- [ ] Batche po 50 tytułów → POST `Special:Export` z `curonly=1`
- [ ] Retry 3× z backoffem, pauza 2–3 s między batchami; wznowienie z `meta/progress.json`
- [ ] Zapis surowych dumpów XML (gzip) w `dump/`
- [ ] Sanity: liczba stron w dumpach == liczba z sitemapy

### 3. Ekstrakcja do plików per strona
- [ ] Parsowanie XML → dla każdej strony plik z **surowym wikitextem**
- [ ] Indeks: tytuł, URL źródłowy na psychonautwiki.org, rewizja, timestamp, checksum
- [ ] Sanity: brak pustych plików; zbiór tytułów == sitemap

### 4. Render czytelny (markdown)
- [ ] Konwersja wikitext → markdown (`wikitextparser`); fallback `fetch_page_text()` dla trudnych szablonów
- [ ] Każdy plik z nagłówkiem: tytuł, URL źródłowy, licencja CC BY-SA 4.0, timestamp rewizji
- [ ] Sanity: każdy wikitext ma render; brak pustych renderów

### 5. Dane strukturalne z API GraphQL serwisu
- [ ] Stronowanie po wszystkich substancjach (~373): dawki, ROA, czas trwania, tolerancja, klasy, efekty, toksyczność
- [ ] Zapis do plików JSON
- [ ] Sanity: brak błędów w odpowiedziach; kompletność listy substancji

### 6. Indeksy i metadane runu
- [ ] Kompletny indeks stron → plik JSON (z source_url per strona)
- [ ] Indeks substancji i efektów
- [ ] `meta/run.json`: data runu, lastmod sitemapy, liczby, checksumy

### 7. Weryfikacja całości (brama ukończenia)
- [ ] Trzy liczby muszą się zgadzać: sitemap == wyekstrahowane strony == pliki na dysku
- [ ] Losowa próba 20 stron: checksum wikitextu vs żywy Special:Export
- [ ] Wszystkie pliki niepuste, indeksy kompletne
- [ ] Raport `meta/verify-report.json` — **fail któregokolwiek checku = dump nieukończony**

### 8. Publikacja
- [ ] Nota licencyjna CC BY-SA 4.0 + atrybucja w README
- [ ] Commit pełnego snapshotu do main, PR, merge

### 9. Rozszerzenia (po merge'u, opcjonalne)
- [ ] Pozostałe namespace serwisu (dyskusje, szablony)
- [ ] Refresh przyrostowy po `lastmod`
- [ ] Naprawa `pypsychonaut.list_wiki_pages()`

## Zasady pobierania

- ~85 żądań na całą treść NS_0 (batche 50-stronowe), pauzy 2–3 s, retry z backoffem
- Tylko bieżące rewizje (`curonly=1`), bez historii edycji
- Bez scrapowania HTML — treść z natywnego eksportu MediaWiki

## Postęp

- [x] Weryfikacja ścieżek dostępu do serwisu (sitemap, Special:Export, GraphQL, licencja)
- [x] README.md — stan obecny, cel, metodologia, licencja (bez obiecywania nieistniejącej struktury)
- [x] TODO.md — ten workflow
- [ ] Kroki 1–8: wykonanie i odhaczanie
- [ ] Gotowy produkt: cała wiedza psychonautwiki.org w repo
