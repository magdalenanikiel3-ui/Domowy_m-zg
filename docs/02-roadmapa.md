# Roadmapa

## Dlaczego akurat taka kolejność

Cztery funkcje, które składają się na produkt, mają **różny moment pozyskania użytkownika i różną wartość w dniu pierwszym**. To jest jedyny powód, dla którego nie wypuszczamy ich razem.

| Funkcja | Wartość w dniu 1 | Rola |
|---|---|---|
| Interpretator pism i rachunków | natychmiastowa, 30 sekund | silnik pozyskania |
| Archiwum (paragony, gwarancje, dokumenty) | za 2 lata | silnik retencji, zero siły pozyskania |
| Cykliczne czynności / „kiedy ostatnio" | za 3 miesiące | silnik nawyku |
| Szybkie wrzucanie głosem | wymaga, by apka znała już obiekty | zależny od pozostałych |

Wypuszczone naraz: opis w sklepie musi tłumaczyć cztery rzeczy, a trzy z nich są w dniu pierwszym puste. To klasyczna śmierć aplikacji typu „drugi mózg".

---

## v1 — Skrzynka na papiery

**Jedno zdanie:** zrób zdjęcie każdego papieru, który do Ciebie przyszedł, a powiem Ci, co masz z nim zrobić i do kiedy.

**Cztery ekrany, ani jednego więcej:**

1. Dodaj — kamera, udostępnij, PDF z dysku
2. Wynik — co to za pismo, kto nadawca, co zrobić, do kiedy, jaka kwota, **cytat ze źródła z podświetleniem na zdjęciu**
3. Terminy — najbliższe, chronologicznie
4. Archiwum — szukajka po treści

**Nie ma w v1:** rodziny, głosu, cyklicznych, kategorii, budżetu, wykresów, punktów, grywalizacji.

**Techniczne minimum:**

- Android natywnie (Kotlin) albo Flutter. Krytyczne: `ACTION_SEND` jako share target oraz kamera
- Backend: Supabase albo Firebase (auth, baza, storage) plus jedna funkcja brzegowa
- Bez osobnego OCR — model vision czyta polskie dokumenty bezpośrednio, taniej i dokładniej niż Tesseract plus LLM
- Kalendarz systemowy przez `CalendarContract`. Nie budujemy własnego kalendarza

**Zasada nienaruszalna:** każda data i kwota pokazana użytkownikowi ma cytat z oryginału i podświetlenie na zdjęciu. Jeśli model nie potrafi wskazać, gdzie to widzi — nie pokazujemy tego.

---

## v2 — Szybkie wrzucanie i pytania

Notatka głosowa albo jedna linijka tekstu zamieniana na wpis. Szczegóły w [`../parser/`](../parser/).

**Trzy drogi wejścia, wszystkie do tego samego miejsca:**

- widget na ekranie głównym — jedno dotknięcie zaczyna nagrywanie (otwieranie apki to przegrana: z trzech sekund robi się piętnaście)
- pole tekstowe z tym samym parserem — głos jest świetny w domu, bezużyteczny w autobusie
- share target — z v1

**Strona pytań jest ważniejsza niż strona wpisów.** „Kiedy ostatnio wymieniałam olej?" z odpowiedzią w jednym zdaniu zamienia notatnik w mózg. Technicznie banalne — zapytanie po małej, uporządkowanej tabeli, bez wektorów i bez RAG-a.

**Przepływ potwierdzania:**

1. Zapis natychmiastowy i bezwarunkowy — nawet gdy model nie zrozumie ani słowa, nagranie nie ginie
2. Interpretacja w tle, wynik jako **jedna karta**, jeden przycisk na całą serię
3. „Popraw" to znowu głos, **nigdy formularz** — inaczej głos był stratą czasu
4. Materializacja leniwa: zapisz **regułę** plus dwa najbliższe wystąpienia, nie całą serię

---

## v3 — Cykliczne i gwarancje

Czynności powtarzalne z dużym przyciskiem „zrobione dzisiaj". Gwarancje wyciągane z paragonów. Przypomnienia przed końcem ochrony.

Tu dopiero wchodzimy na teren zajęty przez istniejące polskie apki (*Paragon – Twoje Gwarancje*, *ParagON*, *Gwarancje App*) — ale wchodzimy jako funkcja w większej całości, nie jako osobny produkt.

**Zasada powiadomień:** maksymalnie **jedno powiadomienie dziennie, zbiorcze, o stałej godzinie**. Apka przypominająca o trzydziestu rzeczach zostaje wyciszona w drugim tygodniu, a wyciszona apka jest martwa.

---

## v4 — Rodzina

Wspólne archiwum, kilka osób, plan Rodzina. Moment monetyzacji i moment, w którym dane stają się na tyle wspólne, że nikt z tego nie wychodzi.

---

## Co świadomie wycięte

### Z listy pierwotnych pomysłów

| Pomysł | Powód odrzucenia |
|---|---|
| AI lodówka ze zdjęcia | Rozpoznawanie wnętrza lodówki działa w demie, nie w życiu: produkty zasłonięte, opakowane, w połowie zjedzone. Świetny TikTok, zerowa retencja na 30. dzień |
| Lista rzeczy do spakowania | Commodity — dowolny czat robi to za darmo i lepiej |
| Analiza wyciągów bankowych | Aplikacje banków już to pokazują, a my braliśmy na siebie najwrażliwsze dane jakie istnieją plus osobną politykę Google Play |
| „Gdzie to schowałem" | Wymaga logowania w momencie chowania rzeczy, czyli dokładnie wtedy, kiedy nikt nie ma cierpliwości |
| Biblioteka instrukcji obsługi (PDF) | Brzmi użytecznie, nie jest używane nigdy — ludzie wpisują numer modelu w wyszukiwarkę. Zamiast tego: zapisz model i numer seryjny, instrukcję podlinkuj na żądanie |

### Z parsera (cięcie v2)

Dziewięć typów kotwicy czasu zeszło do pięciu. Wypadły: `po_wydarzeniu` („po majówce", „w drugim tygodniu ferii"), `n_ty_dzien_miesiaca`, `koniec_okresu`, `warunek`.

Kryterium nie brzmiało „usuń wyrażenia względne", tylko **„usuń wyrażenia rzadkie"**. „W piątek", „jutro", „za dwa tygodnie" i „co roku" zostają — to one są powodem, dla którego głos jest szybszy od kalendarza. Uzasadnienie w [`../parser/testy.md`](../parser/testy.md), sekcja „Cięcie v2".

---

## Ryzyka, w kolejności powagi

1. **Halucynowana data** — zasada cytatu, bez wyjątków
2. **RODO** — dokumenty urzędowe to potencjalnie dane zdrowotne, finansowe i sądowe. Hosting w UE, umowa powierzenia z dostawcą modelu, wyłączone trenowanie na danych. Największy ukryty koszt projektu
3. **Odpowiedzialność** — nigdy „podpisz / nie podpisuj", nigdy „nie musisz płacić". Zawsze „oto co dokument mówi"
4. **Platforma** — asystent AI jest wbudowany w Androida i analizuje zdjęcia za darmo. Obrona: share target, pamięć, kalendarz, polski kontekst
5. **Dystrybucja** — instalacje globalnie spadają, wydatki w aplikacjach użytkowych rosną. Trudniej pozyskać, łatwiej zmonetyzować. Płatne pozyskanie odpada, potrzebny kanał organiczny
