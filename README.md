# Psychonautwiki-data

**Kompletna, ustrukturyzowana kopia całej wiedzy z [PsychonautWiki](https://psychonautwiki.org)** — naukowej, otwartej encyklopedii substancji psychoaktywnych (dawki, efekty, czas trwania, tolerancja, klasy chemiczne, harm reduction).

> **Status repo:** w budowie — postęp realizacji w [`TODO.md`](TODO.md).
> Gdy TODO.md będzie w całości odhaczone, to repozytorium zawiera **100% treści artykułowej PsychonautWiki** w formacie bezpośrednio czytelnym dla agentów AI.

## Dlaczego istnieje to repo

PsychonautWiki to jedyna w swoim rodzaju, uporządkowana baza wiedzy o substancjach psychoaktywnych, ale istnieje tylko jako strona WWW. Celem tego repo jest dostarczenie jej treści jako **datasetu**: plików, które można sklonować, indeksować, przeszukiwać pełnotekstowo i podać agentowi AI jako źródło wiedzy — bez scrapowania HTML, bez parsowania strony przy każdym pytaniu.

## Co dokładnie znajduje się w repo (docelowa struktura)

```
Psychonautwiki-data/
├── README.md                  # ten plik
├── TODO.md                    # workflow + postęp prac
├── LICENSE                    # licencja na kod narzędzi
├── dump/                      # autorytatywny zapis: surowe XML (MediaWiki export-0.11, gz)
│   └── batch-NNN.xml.gz
├── pages/                     # 1 plik na stronę (wikitext + markdown)
│   └── ns0/
│       ├── LSD.wikitext       # surowy źródłowy wikitext (lossless)
│       └── LSD.md             # markdown wyrenderowany do czytania przez AI
├── data/                      # dane strukturalne z GraphQL API
│   ├── substances.json        # wszystkie substancje: dawki, ROA, czas trwania, tolerancja, klasy
│   ├── effects.json           # wszystkie efekty subiektywne
│   └── reagents.json          # testy odczynnikowe + wyniki
├── index/
│   ├── pages.json             # spis stron: tytuł, ns, url źródłowy, timestamp rewizji, sha256
│   ├── substances.json        # spis substancji -> pliki
│   └── effects.json           # spis efektów -> pliki
└── meta/
    ├── sitemap.json           # snapshot oficjalnej sitemapy (pełna lista stron)
    └── run.json                # data dumpa, liczby, checksumy, wersja pipeline'u
```

### Jak ma tego używać agent AI

- **Odpowiedź na pytanie o substancję**: wczytać `pages/ns0/<Substancja>.md` (opis, dawkowanie, efekty) + rekord z `data/substances.json` (strukturalne dawki/czasy).
- **Pytanie o efekt**: `pages/ns0/<Efekt>.md` + `index/effects.json`.
- **Wyliczenie/lista** (np. "wszystkie lysergamidy"): filtrować `data/substances.json` (klasy psychoaktywne i chemiczne są tam jawne).
- **Pełnotekstowe szukanie**: `index/pages.json` + grep po `pages/`.
- **Weryfikacja świeżości**: `meta/run.json`.
- Zasada źródła prawdy: jeśli markdown i wikitext się różnią, **wikitext (`*.wikitext`) i `dump/` są autorytatywne**; `.md` to warstwa wygody wygenerowana z wikitextu.

## Skąd i jak pobierane są dane (metodologia)

| Warstwa danych | Źródło | Metoda |
|---|---|---|
| Lista wszystkich stron | oficjalna sitemapa MediaWiki (`/sitemap/sitemap-index-mediawiki-pw_.xml`, wpis w `robots.txt`) | pobranie indeksu + plików namespace, wyparsowanie URL-i (NS_0 = ~4124 strony) |
| Pełna treść stron | `Special:Export` (POST, `curonly=1`) | batche po 50 tytułów → standardowy XML MediaWiki `export-0.11` z pełnym wikitextem; **bez scrapowania HTML** |
| Dane strukturalne | publiczne GraphQL API `api.psychonautwiki.org` | stronowanie po wszystkich substancjach (~373): dawki, ROA, czas trwania, tolerancja, klasy, efekty, reageny |

Pobieranie jest batchowane i wolne (cała treść NS_0 ≈ 85 żądań z przerwami) — pełen szacunek dla infrastruktury wiki. Endpoint `list=allpages` API jest zablokowany anti-botem (HTTP 418), dlatego sitemapa jest jedynym wiarygodnym wyliczeniem stron.

## Zależności i powiązane repozytoria

| Narzędzie | Repo | Do czego służy |
|---|---|---|
| `pypsychonaut` | [bmateuszideas/pypsychonaut](https://github.com/bmateuszideas/pypsychonaut) | transport z obejściem Cloudflare (curl_cffi/FlareSolverr), klient GraphQL, dumper markdown — z niego pochodzi główna logika pipeline'u |
| `wiki-scripts` | [bmateuszideas/wiki-scripts](https://github.com/bmateuszideas/wiki-scripts) | punkt odniesienia: `wiki_dump_downloader` robi ten sam dump przez `Special:Export` |
| `wikitextparser` | PyPI | parsowanie i konwersja wikitext → markdown |
| `python-frontmatter`, `pypandoc` | PyPI | render markdown i frontmatter z metadanymi strony |

Dokładny przebieg pracy: [`TODO.md`](TODO.md).

## Licencja i atrybucja

Cała treść wiki pochodzi z **PsychonautWiki** i jest objęta licencją **Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)**, jak podano w stopce każdej strony wiki.

Warunki redystrybucji tej treści:
1. atrybucja PsychonautWiki + link do `https://psychonautwiki.org` (każdy rekord w `index/pages.json` zawiera `source_url` strony),
2. wskazanie licencji CC BY-SA 4.0 z linkiem,
3. dalsze udostępnianie na tej samej (lub zgodnej) licencji.

Kod narzędzi w tym repo jest licencjonowany odrębnie przez właściciela repo (`LICENSE`).

## Odświeżanie dumpa

Dump można odświeżyć w dowolnym momencie ponownym uruchomieniem pipeline'u (`TODO.md`, kroki 1–7). Każdy run zapisuje timestamp i `lastmod` sitemapy w `meta/run.json`, więc wiek snapshotu jest zawsze jawny. Sitemapa zawiera `lastmod` per namespace — przyszłe runy mogą pobierać tylko zmienione strony.
