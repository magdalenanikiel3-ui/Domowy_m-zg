# Parser wejścia głosowego — testy na trudnych zdaniach

Data testu: 20.08.2026 · kontekst: czwartek, 15:30

> **Uwaga: prompt i schema zostały uproszczone do wersji 2.0.** Testy poniżej przeprowadzono na wersji 1.0. Wnioski zostają w mocy, ale część przypadków z grupy B ("po majówce", "w drugim tygodniu ferii", "przed świętami") nie jest już obsługiwana celowo — trafiają do kotwicy `brak`. Powód w sekcji "Cięcie v2" na końcu pliku.

**Kontekst podstawiony do testów:** obiekty znane aplikacji — auto (Skoda, 102 400 km), Burek (pies), Fela (pies), ekspres, pralka, Kuba (dziecko), mieszkanie. Województwo: nieznane.

---

## Najważniejszy wniosek, zanim przejdziemy do tabel

**Groźna klasa błędu to nie zła data. To wymyślona data tam, gdzie żadnej nie było.**

Spodziewałam się, że najtrudniejsze będą wyrażenia względne („po majówce", „w drugim tygodniu ferii"). Okazało się, że te są bezpieczne, bo model albo je rozumie, albo widać, że nie rozumie. Prawdziwe niebezpieczeństwo jest w zdaniach, które **wyglądają jak zadanie z datą, a jej nie mają**: „olej 105300 km", „przegląd auta", „jak wrócę z urlopu", „kiedy ostatnio wymieniałam olej?". Model pod presją wypełnienia schemy wstawia tam „dzisiaj" albo „za tydzień" i użytkownik dostaje śmieć w kalendarzu, którego nie zamawiał.

To dlatego w schemie `kotwica_czasu.typ` ma wartość `"brak"`, a w prompcie jest osobna zasada „nie zmyślasz". Bez tego cały system jest niewiarygodny.

**Drugi wniosek: zasada „model nie liczy dat" jest warta więcej, niż myślałam** — i mam na to dowód na sobie. W makiecie karty, którą Ci pokazałam wcześniej, napisałam „13 wizyt". Policzyłam potem kodem: **wizyt jest 14** (21.08.2026 – 19.02.2027, co 14 dni). Popełniłam dokładnie ten błąd, przed którym chroni ta zasada. Gdyby liczbę liczył kod, a nie model, błąd byłby niemożliwy.

**Trzeci wniosek, którego nie przewidziałam.** Wśród tych 14 wizyt jedna wypada **25.12 — w Boże Narodzenie**. Żaden weterynarz nie przyjmuje. Resolver musi sprawdzać kolizje ze świętami i dniami wolnymi i proponować przesunięcie. To jest dokładnie ta „polskość", o której mówiłaś — i nie da się jej dodać później, bo musi siedzieć w warstwie liczącej daty.

---

## Grupa A — przechodzą czysto, bez pytania (9 z 20)

| Wypowiedź | Kotwica | Uwaga |
|---|---|---|
| „w piątek do weterynarza na 15, co dwa tygodnie przez pół roku" | `dzien_tygodnia` PT, najbliższy + powtarzanie 14 dni, koniec po 6 mies. | obiekt niejasny → patrz grupa B |
| „we wtorek do dentysty" *(mówione w czwartek)* | `dzien_tygodnia` WT, najbliższy | bezpieczne, bo wtorek nie jest dziś |
| „za dwa tygodnie kontrola u lekarza" | `przesuniecie` 2 tygodnie | |
| „w pierwszy poniedziałek września zebranie wspólnoty" | `n_ty_dzien_miesiaca` n=1, PON, IX | kod wyliczy 07.09.2026 |
| „na koniec miesiąca odczyt licznika" | `koniec_okresu` miesiac, `tylko_roboczy: true` | odczyt = sprawa urzędowa, więc dzień roboczy |
| „PIT do 30 kwietnia" | `data_wprost` 30.04, rok null | **rodzaj: termin**, nie wydarzenie |
| „co 90 dni filtr do wody" | powtarzanie co=90, jednostka=`dzien` | nie normalizuj na „3 miesiące" — to inna data |
| „odkamieniłam dziś ekspres" | `data_wprost` dziś | **rodzaj: wykonane** — wpis do historii, nie do kalendarza |
| „wymienić olej za 3 tysiące km" | `przebieg` km_delta 3000 | powtarzanie w jednostce `km`, nie w czasie |

---

## Grupa B — poprawnie pytają (6 z 20)

Te przypadki **muszą** zadać pytanie. Gdyby model je rozstrzygnął sam, byłby to błąd nawet przy trafnym zgadnięciu — bo raz na kilka razy zgadnie źle i straci zaufanie.

| Wypowiedź | Dlaczego pytanie | Pytanie |
|---|---|---|
| „psa do weterynarza…" | dwa psy w kontekście | Burek / Fela |
| „we wtorek do dentysty" *(mówione we wtorek)* | dziś czy za tydzień — realna dwuznaczność | Dziś / Za tydzień |
| „w przyszły czwartek szczepienie" | **„przyszły" jest w polszczyźnie dwuznaczne** i sami Polacy rozumieją je różnie: 27.08 albo 03.09 | 27.08 / 03.09 |
| „w drugim tygodniu ferii wycieczka Kuby" | ferie zimowe różnią się województwem i zmieniają co rok | wybór województwa |
| „przed świętami kupić prezenty" | w sierpniu „święta" to prawie pewnie Boże Narodzenie, ale odległość do Wielkanocy nie jest absurdalna | Boże Narodzenie / Wielkanoc |
| „za tydzień w środę" | **sprzeczność w samym zdaniu** — za tydzień to czwartek 27.08, a środa to 26.08 | 26.08 / 02.09 |

Ostatni przypadek jest moim ulubionym, bo pokazuje, że parser musi wyłapywać **wewnętrzne sprzeczności**, nie tylko braki. Słaby model wybierze jedną z liczb i nic nie powie.

---

## Grupa C — pułapki. Tu model musi powiedzieć „nie wiem" (5 z 20)

Każdy z tych przypadków ma poprawną odpowiedź „brak kotwicy czasu". Każdy jest zaproszeniem do halucynacji.

| Wypowiedź | Poprawnie | Czego się boję |
|---|---|---|
| „olej 105300 km" | `przebieg` km=105300, brak daty. Uwaga: 105 300 > obecne 102 400, więc to **plan**, nie zapis wykonania | model wstawi dzisiejszą datę i uzna olej za wymieniony |
| „kiedy ostatnio wymieniałam olej?" | **rodzaj: pytanie** — nic nie zapisuj, odpowiedz z archiwum | model utworzy zadanie „wymienić olej" |
| „przegląd auta" | `brak`. Apka sama wie, że przegląd jest roczny — sprawdź archiwum i **zaproponuj** datę, ale jako propozycję | model zgadnie datę z powietrza |
| „jak wrócę z urlopu oddzwonić do Kowalskiego" | `warunek`, bez daty. Wpis pływający na liście „bez terminu" | model wstawi „za tydzień" |
| „opony zmienić jak spadnie poniżej 7 stopni" | `warunek` + `propozycja_zastepcza`: połowa października (polska konwencja) | model wstawi datę bez oznaczenia jej jako propozycji |

Rozróżnienie **data ustalona vs. propozycja apki** musi istnieć w modelu danych i być widoczne w interfejsie (np. datą pisaną kursywą albo szarą). Inaczej po pół roku użytkownik nie wie, które terminy sam podał, a które apka mu dorzuciła — i przestaje wierzyć wszystkim.

---

## Uczciwe ograniczenie tego testu

Te wyniki to **sufit, nie podłoga**. Przeszłam te zdania własnym rozumowaniem, a jestem dużym modelem. Model, którego realnie użyjesz w produkcji — tani, szybki, klasy Flash-Lite za 0,19 grosza za wywołanie — wypadnie zauważalnie gorzej, szczególnie w:

- grupie B: tani model **rozstrzyga zamiast pytać**, bo pytanie wymaga przyznania się do niepewności
- grupie C: tani model **wypełnia schemę do końca**, bo puste pola wyglądają na niedokończoną pracę

Dlatego test na prawdziwym modelu jest obowiązkowy i to jest Twój następny krok. Jak go zrobić:

1. Wejdź w Google AI Studio (darmowe), wklej `parser-prompt.txt` jako instrukcję systemową, ustaw `parser-schema.json` jako wymuszone wyjście
2. Przepuść te 20 zdań, po jednym
3. Licz tylko dwie rzeczy: **ile razy wymyślił datę, której nie było** (grupa C) i **ile razy rozstrzygnął zamiast zapytać** (grupa B)
4. Jeśli którakolwiek liczba jest większa od zera na 20 próbach — dokręcaj prompt, nie zaczynaj kodu

Progi, które przyjęłabym za zielone światło: **zero halucynacji daty** i **maksymalnie 1 na 20 nieuzasadnionych rozstrzygnięć**. Halucynacja daty to warunek zerowy, bo to jedyny błąd tego systemu, który realnie kosztuje użytkownika pieniądze albo przegapiony termin.

---

## Co musi zrobić kod, a nie model

Podział pracy, który wynika z tych testów:

| Zadanie | Kto |
|---|---|
| Rozpoznanie rodzaju wypowiedzi, obiektu, wyrażeń czasowych | model |
| Wykrycie niejednoznaczności i sprzeczności | model |
| **Wyliczenie konkretnych dat z symboli** | kod |
| **Policzenie liczby wystąpień w serii** | kod |
| **Sprawdzenie kolizji ze świętami i dniami wolnymi** | kod |
| Terminy ferii zimowych per województwo | kod, z aktualizowanej tabeli |
| Materializacja: zapisz regułę + 2 najbliższe wystąpienia | kod |
| Godziny domyślne, przypomnienia, czas trwania | kod |

Wszystko w prawej kolumnie jest deterministyczne, testowalne testem jednostkowym i darmowe. Wszystko, co przeniesiesz z prawej do lewej, staje się źródłem błędów, za które płacisz zaufaniem użytkownika.

**Jedyna tabela danych, którą musisz utrzymywać ręcznie:** święta i dni wolne w Polsce. Aktualizacja raz w roku, kilkanaście pozycji. (W wersji 1.0 była tu jeszcze tabela ferii zimowych per województwo — wypadła razem z cięciem v2.)

---

## Cięcie v2 — co usunęliśmy i dlaczego

Wersja 1.0 miała **dziewięć typów kotwicy czasu**. Wersja 2.0 ma **pięć**. Usunięte:

| Usunięty typ | Przykład | Dlaczego wypadł |
|---|---|---|
| `po_wydarzeniu` | „po majówce", „przed świętami", „w drugim tygodniu ferii" | Rzadkie w praktyce, a kosztowały najwięcej: tabela ferii per województwo aktualizowana co roku, rozstrzyganie Wielkanoc/Boże Narodzenie, pytanie o region |
| `n_ty_dzien_miesiaca` | „w pierwszy poniedziałek września" | Rzadkie. Osobny resolver i osobny sposób pokazania w interfejsie |
| `koniec_okresu` | „na koniec miesiąca" | Rzadkie, a dodatkowo dwuznaczne (ostatni dzień czy ostatni roboczy) |
| `warunek` | „jak wrócę z urlopu", „jak spadnie poniżej 7 stopni" | Nie da się zamienić na datę. Funkcja zachowana, ale przez `brak` — wpis ląduje na liście bez terminu |

**Co zostało i dlaczego akurat to:** `data_wprost`, `dzien_tygodnia`, `przesuniecie`, `przebieg`, `brak`.

Kryterium cięcia nie brzmiało „usuń wyrażenia względne", tylko „usuń wyrażenia rzadkie". To ważne rozróżnienie. „W piątek", „jutro", „za dwa tygodnie" i „co roku" **zostają**, bo są częste i bo są jedynym powodem, dla którego głos jest szybszy od kalendarza. Gdyby użytkownik musiał mówić „dwudziestego pierwszego sierpnia dwa tysiące dwudziestego szóstego roku", nagrywanie trwałoby dłużej niż dotknięcie daty w kalendarzu i cała funkcja traciłaby sens.

**Co czyni to cięcie bezpiecznym:** kotwica `brak` z kalendarzem jako wyjściem awaryjnym. Model, który nie rozumie „po majówce", nie zgaduje i nie blokuje się — zapisuje wpis bez daty i pokazuje kalendarz. Użytkownik dotyka daty raz i idzie dalej. Koszt nieobsłużonego wyrażenia to jedno dodatkowe dotknięcie, a nie utracony wpis.

To jest właściwy kierunek dla v1: **wąski parser plus dobry wyjątek bije szeroki parser bez wyjątku.** Egzotyczne wyrażenia można dołożyć później, gdy zobaczysz w danych, że ludzie faktycznie ich używają — a nie dlatego, że dało się je przewidzieć.

**Czego cięcie nie ruszyło.** Cztery główne wnioski z testów dotyczyły warstwy, która została nietknięta:

1. Model nie liczy dat — kod liczy
2. Halucynacja daty tam, gdzie jej nie ma, to najgroźniejszy błąd (i po cięciu jest **ważniejsza**, bo więcej wypowiedzi trafia do `brak`)
3. Rozróżnienie rodzaju wypowiedzi: wydarzenie / termin / wykonane / pytanie
4. Kolizje z polskimi świętami sprawdza resolver

Punkt 3 jest najcenniejszą częścią całego parsera i nie ma nic wspólnego z datami. To on decyduje, czy „odkamieniłam dziś ekspres" trafi do historii, a nie do kalendarza na przyszłość, i czy „kiedy ostatnio wymieniałam olej?" zostanie potraktowane jako pytanie, a nie jako nowe zadanie.
