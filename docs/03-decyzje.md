# Decyzje projektowe

Zapis kluczowych rozstrzygnięć wraz z powodem. Sens tego pliku: za pół roku będziesz chciała wiedzieć, **dlaczego** coś jest tak, a nie inaczej — i czy powód nadal obowiązuje.

---

## D1. Model językowy nie liczy dat

**Decyzja.** Model zwraca symbol (`dzien_tygodnia: PT, ktory: najblizszy`). Konkretną datę wylicza deterministyczny kod.

**Powód.** Modele językowe mylą się w arytmetyce kalendarzowej, a błędna data to jedyny błąd tego systemu, który kosztuje użytkownika przegapiony termin.

**Dowód z praktyki.** Przy projektowaniu makiety karty potwierdzenia policzyłam, że wizyt u weterynarza co dwa tygodnie przez pół roku będzie **13**. Kod policzył **14** (21.08.2026 – 19.02.2027). Błąd popełniony dokładnie w miejscu, przed którym chroni ta zasada.

**Konsekwencja.** Cała arytmetyka kalendarzowa jest testowalna testem jednostkowym i darmowa. Wszystko, co przeniesiesz z kodu do modelu, staje się źródłem błędów, za które płacisz zaufaniem użytkownika.

---

## D2. Brak daty jest poprawną odpowiedzią

**Decyzja.** `kotwica_czasu.typ = "brak"` to pełnoprawny wynik. Aplikacja zapisuje wtedy wpis bez terminu i pokazuje kalendarz do wybrania ręcznie.

**Powód.** Najgroźniejsza klasa błędu to **nie zła data, tylko wymyślona data tam, gdzie żadnej nie było**. Zdania typu „olej 105300 km", „przegląd auta", „jak wrócę z urlopu", „kiedy ostatnio wymieniałam olej?" wyglądają jak zadanie z datą, a jej nie mają. Model pod presją wypełnienia schemy wstawia „dzisiaj" i użytkownik dostaje śmieć w kalendarzu, którego nie zamawiał.

**Konsekwencja.** Ta zasada stała się **ważniejsza** po cięciu v2, bo więcej wypowiedzi trafia teraz do `brak`. Koszt nieobsłużonego wyrażenia to jedno dodatkowe dotknięcie w kalendarzu, a nie utracony wpis.

---

## D3. Rodzaj wypowiedzi przed wszystkim innym

**Decyzja.** Parser najpierw klasyfikuje: `wydarzenie` / `termin` / `wykonane` / `pytanie` / `niejasne`.

**Powód.** To samo pole tekstowe obsługuje cztery semantycznie różne rzeczy:

- „w piątek do weterynarza" → wydarzenie w przyszłości
- „PIT do 30 kwietnia" → **termin**, czyli inna logika powiadomień (wcześniej i wielokrotnie)
- „odkamieniłam dziś ekspres" → **zapis przeszłości**, nie wpis do kalendarza
- „kiedy ostatnio wymieniałam olej?" → **pytanie**, nie twórz zadania

Pomylenie tych czterech daje najbardziej irytujące błędy: zadanie „wymienić olej" utworzone w odpowiedzi na pytanie o olej.

**Uwaga.** To najcenniejsza część parsera i **nie ma nic wspólnego z datami**. Przetrwała cięcie v2 nietknięta.

---

## D4. Archiwum jest efektem ubocznym

**Decyzja.** Nie ma ekranu „dodaj swoje urządzenia". Dane wpadają przy okazji czynności, którą użytkownik i tak wykonuje.

**Powód.** Produkty typu „drugi mózg" mają cmentarny wskaźnik przeżycia, bo wymagają użytkownika, który prowadzi system — takich jest około 5%. Przeżywają wyłącznie te, w których utrzymanie danych jest produktem ubocznym czegoś innego. Aplikacja banku pamięta wszystkie Twoje transakcje, a Ty nie robisz nic.

**Test dla każdej przyszłej funkcji.** Czy dane wchodzą bez tego, żeby użytkownik *postanowił* je wprowadzić?

- przechodzi: zdjęcie papieru, którym i tak musiał się zająć; import z maila; notatka głosowa w trzy sekundy
- nie przechodzi: formularze, listy kontrolne, onboarding z dwunastoma pytaniami

---

## D5. Asymetryczne potwierdzanie

**Decyzja.** Potwierdzamy tylko to, co niepewne i kosztowne. Maksymalnie **jedno pytanie** na wypowiedź, zawsze jako dwa lub trzy przyciski, nigdy jako pole tekstowe.

**Powód.** Za dużo potwierdzania — wolniej niż wpisać, użytkownik przestaje. Zero potwierdzania — jedna zła interpretacja i tracisz zaufanie, a przy serii powtarzanej to kilkanaście śmieci w kalendarzu naraz.

**Hierarchia:**

| | Zachowanie |
|---|---|
| Milcz | przypomnienie, czas trwania, obiekt gdy pasuje tylko jeden |
| Pokaż, nie pytaj | wyliczona data (**21.08**, nie „piątek"), liczba powtórzeń |
| Zapytaj | dwa psy; „we wtorek" powiedziane we wtorek; sprzeczność w zdaniu |

Zawsze pokazuj **wyliczoną datę, nie usłyszane słowo** — użytkownik wyłapie błąd wzrokiem w pół sekundy, bez czytania.

---

## D6. Materializacja leniwa

**Decyzja.** Seria powtarzalna zapisywana jest jako **jedna reguła** plus dwa najbliższe wystąpienia. Reszta generuje się w miarę upływu czasu.

**Powód.**

- zła reguła to jedna poprawka zamiast usuwania kilkunastu wpisów z kalendarza systemowego
- wizyty i tak się przesuwają — sztywna seria jest nieaktualna po drugim tygodniu
- kalendarz systemowy nie zostaje zaśmiecony

---

## D7. Resolver sprawdza polskie dni wolne

**Decyzja.** Po wyliczeniu dat kod sprawdza kolizje z tabelą [`../resolver/swieta-pl.json`](../resolver/swieta-pl.json) i proponuje przesunięcie.

**Powód.** Wykryte podczas testów: w serii wizyt u weterynarza co dwa tygodnie od 21.08.2026 jedna wypada **25.12, w Boże Narodzenie**. Żaden weterynarz nie przyjmuje.

**Dlaczego to nie może poczekać.** To jest dokładnie ta „polskość", która ma być przewagą produktu — i musi siedzieć w warstwie liczącej daty od pierwszego dnia, bo doklejenie jej później oznacza przeliczenie wszystkich istniejących serii.

**Pułapka, o której łatwo zapomnieć.** Od 2025 roku **24 grudnia jest w Polsce dniem ustawowo wolnym** (ustawa z 6.12.2024). Większość gotowych bibliotek kalendarzowych o tym nie wie. Tabela w repo to uwzględnia.

---

## D8. Cennik odwrotny do intuicji

**Decyzja.** Analizy dokumentów **bez limitu za darmo**, ale wynik znika po 7 dniach. Płatne jest archiwum, terminy, kalendarz i szukajka.

**Powód.** Jedna analiza dokumentu kosztuje **0,19 grosza**. Przy 10 000 użytkowników po 5 analiz miesięcznie to 95 zł miesięcznie za API. Koszt modelu **nie jest ograniczeniem tego biznesu** — więc limitowanie analiz to wyrzucanie najlepszego narzędzia marketingowego, jakie masz.

Płatna jest **pamięć**: rzecz droga do odtworzenia i bolesna do utracenia. Użytkownik rozumie, za co płaci abonament, gdy płaci za to, że apka pamięta. Nie rozumie, gdy płaci za „dwadzieścia analiz".

**Do przeliczenia przed startem.** Kalkulacja oparta na cenniku modelu klasy Flash-Lite, który ma być wycofany 16.10.2026. Przelicz dla następcy.

---

## D9. Trzy drogi wejścia, jedno miejsce docelowe

**Decyzja.** Widget na ekranie głównym, pole tekstowe i share target prowadzą do tego samego parsera i tej samej tabeli.

**Powód.** „Jednym kliknięciem" znaczy **nie otwierając aplikacji**. Otwarcie apki zamienia trzy sekundy w piętnaście i funkcja przestaje być używana. Głos jest przy tym świetny w domu i bezużyteczny w autobusie — bez wejścia tekstowego tracisz połowę sytuacji.

Share target to jedyna trwała przewaga nad wklejeniem dokumentu do dowolnego czatu AI.
