# Psychonautwiki-data

**Stan obecny repozytorium:** zawiera wyłącznie dokumentację (`README.md`, `TODO.md`, licencję). **Żadnych danych jeszcze tu nie ma.**

Celem tego repozytorium jest zebranie **całej wiedzy ze strony https://www.psychonautwiki.org** — encyklopedii substancji psychoaktywnych — i zapisanie jej w formie plików tak, żeby dało się ją sklonować, przeszukiwać i używać jako źródła wiedzy dla agenta AI, bez wchodzenia na stronę.

## Co będzie tu trafiało (stan docelowy, do zrealizowania wg `TODO.md`)

Zawartość strony `www.psychonautwiki.org`, pobrana w całości i ustrukturyzowana:

- **surowy wikitext każdej strony artykułowej** (pełna, bezstratna treść — to jest i pozostanie źródło prawdy),
- **wyrenderowana wersja czytelna** (markdown) — warstwa wygody wygenerowana z wikitextu,
- **dane strukturalne** (dawki, drogi podania, czas trwania, klasy, tolerancja, efekty), jakie strona udostępnia przez swoje publiczne API GraphQL,
- **indeksy** — spis wszystkich pobranych stron z linkami do źródła na psychonautwiki.org, żeby każda porcja danych była powiązana z oryginałem.

Dokładna kolejność prac, narzędzia i kryteria ukończenia: [`TODO.md`](TODO.md). Struktura katalogów powstanie jako wynik wykonania TODO — nie jest narzucona z góry.

## Skąd będą pochodzić dane (metodologia, zweryfikowana testami)

Celem jest strona **psychonautwiki.org**, a nie żadne repozytorium kodu:

1. **Lista wszystkich stron** — z oficjalnej sitemapy MediaWiki serwisu psychonautwiki.org (`/sitemap/...`, wpisanej w jego `robots.txt`). Zwykłe API wyliczania stron (`list=allpages`) jest u serwisu zablokowane anti-botem (HTTP 418), sitemapa działa i jest pełna (namespace artykułów: ~4124 strony).
2. **Treść stron** — przez `Special:Export` (natywny mechanizm MediaWiki eksportu stron do XML z pełnym wikitextem), batchami. To daje **100% treści stron** bez scrapowania HTML i bez zależności od wyglądu strony.
3. **Dane strukturalne** — z publicznego API GraphQL serwisu psychonautwiki.org (`api.psychonautwiki.org`), które strona sama udostępnia (substancje, dawki, efekty).

## Licencja i atrybucja

Treść strony PsychonautWiki jest udostępniona na licencji **Creative Commons Attribution-ShareAlike 4.0 (CC BY-SA 4.0)** (stopka każdej strony). Kopiowanie do tego repozytorium jest dozwolone pod warunkiem atrybucji serwisowi PsychonautWiki i dalszego udostępniania na tej samej licencji — każdy element danych będzie więc zawierał odnośnik do oryginalnej strony na psychonautwiki.org.

## Powiązane narzędzia

Do zbudowania pipeline'u wykorzystam Twoje repozytorium [pypsychonaut](https://github.com/bmateuszideas/pypsychonaut) (transport z obejściem Cloudflare, klient GraphQL) jako **narzędzie** — ale źródłem danych jest wyłącznie sama strona www.psychonautwiki.org.
