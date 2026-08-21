# Domowy mózg — analiza pomysłu i specyfikacja MVP

Data: 20.08.2026

---

## 1. Werdykt: architektura dobra, kolejność zła

Połączenie, które proponujesz (sejf + ogarniacz rodziny + interpretator + „kiedy ostatnio") jest **słuszne co do istoty i niebezpieczne co do wykonania**.

**Dlaczego słuszne.** Wszystkie cztery funkcje sprowadzają się do jednego modelu danych. Nie są to cztery aplikacje — to jedna tabela:

**OBIEKT → ZDARZENIA (z datą) + DOKUMENTY**

| Funkcja z listy | Obiekt | Zdarzenie | Dokument |
|---|---|---|---|
| Domowy sejf | pralka | gwarancja do 14.08.2028 | paragon |
| OC | auto | odnowienie 12.10 | polisa |
| Pismo urzędowe | sprawa | odpowiedź do 28.08 | skan pisma |
| Rachunek | mieszkanie | 438 zł, zapłacone 05.08 | faktura |
| Kiedy ostatnio | ekspres | odkamienione 12.06, co 90 dni | — |
| Maskotka do szkoły | dziecko | piątek 22.08 | — |

To jest realne odkrycie, nie łączenie na siłę. ChatGPT wypisał to jako sześć osobnych pomysłów, bo nie zauważył wspólnego prymitywu.

**Dlaczego niebezpieczne.** Każda z czterech funkcji ma inny moment pozyskania użytkownika i inną wartość w dniu pierwszym:

| Funkcja | Wartość w dniu 1 | Rola |
|---|---|---|
| Interpretator pism/rachunków | **natychmiastowa** (30 sekund) | silnik pozyskania |
| Domowy sejf | za 2 lata | silnik retencji, zero siły pozyskania |
| Kiedy ostatnio | za 3 miesiące | silnik nawyku |
| Ogarniacz / głos | wymaga, by apka już znała obiekty | zależny od pozostałych |

Jeśli wypuścisz wszystko naraz, opis w Google Play musi tłumaczyć cztery rzeczy, a trzy z nich są w dniu pierwszym puste. To klasyczna śmierć aplikacji typu „drugi mózg": 500 instalacji, retencja D7 na poziomie 8%.

## 2. Mechanizm, który to ratuje

**Archiwum nie może być funkcją. Musi być efektem ubocznym.**

Interpretacja dokumentu, po którą użytkownik przyszedł, *sama z siebie* tworzy wpis w archiwum i termin w kalendarzu. Użytkownik nigdy nie „wprowadza danych" — dane wpadają, bo zrobił rzecz, którą i tak musiał zrobić.

To jest jedyne wyjście z problemu cold startu. Wspomniałaś, że „apka sama by sugerowała co wprowadzić" — dobry instynkt, ale sugerowanie nie jest wąskim gardłem. Wąskim gardłem jest wpisywanie. Nikt w niedzielę wieczorem nie fotografuje 40 paragonów.

**Test, który musi przejść każda przyszła funkcja:**
> Czy dane wchodzą do aplikacji bez tego, żeby użytkownik *postanowił* je wprowadzić?

- Przechodzi: zdjęcie papieru, którym i tak musiał się zająć; import z maila; notatka głosowa w 3 sekundy.
- Nie przechodzi: formularze, „dodaj swoje urządzenia", listy kontrolne, onboarding z 12 pytaniami.

**Drugi mechanizm, ważniejszy niż myślisz: strona pytań.** Nie samo wrzucanie, ale pytanie:

> „Kiedy ostatnio wymieniałam olej?" → odpowiedź z archiwum, w jednym zdaniu.

To zamienia notatnik w mózg. Technicznie banalne (zapytanie po małej, uporządkowanej tabeli — nie potrzeba wektorów ani RAG-a na start), a to jest cała różnica w odczuciu produktu.

## 3. Kolejność wydawania

| Wersja | Zakres | Cel |
|---|---|---|
| **v1** | Wrzuć papier → co to znaczy, co zrobić, do kiedy → termin do kalendarza → papier zostaje w archiwum | Pozyskanie + zalążek archiwum |
| **v2** | Szybkie wrzucanie głosem („maskotka w piątek") + pytania do archiwum | Codzienny nawyk |
| **v3** | Cykliczne / „kiedy ostatnio" + gwarancje z paragonów | Retencja długoterminowa |
| **v4** | Wspólne archiwum rodziny | Monetyzacja planu Rodzina + brak możliwości wyjścia |

**Warunek bezwzględny: v1 musi być użyteczny dla osoby samotnej.** Jeśli sens pojawia się dopiero gdy dołączy rodzina, produkt umrze na starcie.

**Co bym wyciął z Twojego zestawu:** instrukcje obsługi. Przechowywanie PDF-ów instrukcji brzmi użytecznie i nie jest używane nigdy — ludzie wpisują numer modelu w Google. Zamiast tego zapisuj model i numer seryjny, a instrukcję podlinkuj na żądanie. Nie buduj biblioteki PDF-ów.

---

## 4. Specyfikacja v1

**Jedno zdanie produktu:** *Zrób zdjęcie każdego papieru, który do Ciebie przyszedł, a powiem Ci, co masz z nim zrobić i do kiedy.*

### Ekrany (cztery, nie więcej)
1. **Dodaj** — kamera / udostępnij / PDF z dysku
2. **Wynik** — co to za pismo, kto nadawca, co zrobić, do kiedy, jaka kwota, **cytat ze źródła z podświetleniem miejsca na zdjęciu**
3. **Terminy** — najbliższe, chronologicznie
4. **Archiwum** — szukajka po treści

**Nie ma w v1:** rodziny, głosu, cyklicznych, kategorii, budżetu, wykresów, punktów, grywalizacji.

### Stos technologiczny
- **Android natywnie (Kotlin)** albo Flutter. Krytyczne dwie rzeczy: `ACTION_SEND` share target i kamera.
- **Backend minimalny:** Supabase albo Firebase (auth, baza, storage) + jedna funkcja brzegowa wołająca model.
- **Bez osobnego OCR.** Model vision czyta polskie dokumenty bezpośrednio — taniej, dokładniej i mniej kodu niż Tesseract plus LLM.
- **Wyjście strukturalne (JSON schema):** `typ_dokumentu`, `nadawca`, `kwoty[]`, `daty[]`, `termin_dzialania`, `wymagane_dokumenty[]`, `konsekwencje_braku_reakcji`, `cytaty_zrodlowe[]`
- **Kalendarz:** zapis do systemowego przez `CalendarContract`. Nie buduj własnego kalendarza.

### Zasada nienaruszalna
Każda data i każda kwota pokazana użytkownikowi musi mieć cytat z oryginału i podświetlenie na zdjęciu. **Jeśli model nie potrafi wskazać, gdzie to widzi — nie pokazuj tego.** Halucynowany termin to jedyny błąd, który realnie kosztuje użytkownika pieniądze i zabija produkt recenzjami.

---

## 5. Koszty jednostkowe

Wyliczenie dla modelu klasy Gemini Flash-Lite ($0,10/mln wejście, $0,40/mln wyjście), kurs NBP 3,7306 z 19.08.2026.

Jedna analiza dokumentu: obraz A4 ≈ 1500 tokenów + prompt i schema ≈ 800 + wyjście ≈ 700.

| Pozycja | Koszt |
|---|---|
| **Jedna analiza dokumentu** | **0,19 grosza** |
| 1 000 analiz | 1,90 zł |
| Notatka głosowa 20 s (STT) | 0,75 grosza |

Skala: 10 000 aktywnych użytkowników po 5 analiz miesięcznie = 50 000 analiz.

| Pozycja | Koszt / mies. |
|---|---|
| API | **95 zł** |
| API gdyby model był 10× droższy | 951 zł |
| Storage (800 GB, Cloudflare R2, egress darmowy) | 45 zł |

### Wniosek, który zmienia strategię cenową

**Koszt API nie jest ograniczeniem tego biznesu.** ChatGPT zaproponował „5 darmowych analiz → pakiet 20 za 9,99 zł", tak jakby trzeba było limitować z powodu kosztów. Nie trzeba. Przy 0,19 grosza za analizę możesz być radykalnie hojna i użyć darmowości jako broni marketingowej.

Prawdziwy koszt tego biznesu to pozyskanie użytkownika. Cała reszta to grosze.

---

## 6. Model cenowy (rekomendacja odwrotna do ChatGPT)

| Plan | Cena | Co daje |
|---|---|---|
| **Darmowy** | 0 zł | **Nielimitowane analizy**, ale wynik znika po 7 dniach. Bez archiwum, bez kalendarza, bez szukajki. |
| **PRO** | 14,99 zł/mies. lub 89 zł/rok | Archiwum bez limitu, terminy, kalendarz, szukajka, eksport PDF |
| **Rodzina** | 24,99 zł/mies. | Wspólne archiwum, 5 osób |

**Dlaczego tak.** Analiza jest tania i jest najlepszym marketingiem, jaki masz — im więcej darmowych analiz, tym więcej ludzi zobaczy wartość. Płatna jest **pamięć**: rzecz droga do odtworzenia i bolesna do utracenia. Użytkownik rozumie, za co płaci abonament, gdy płaci za to, że apka pamięta. Nie rozumie, gdy płaci za „20 analiz".

Przychód przy 10 000 aktywnych (po 15% prowizji Google):

| Konwersja | PRO 14,99 | Rodzina 24,99 |
|---|---|---|
| 2% (200 os.) | 2 548 zł/mies. | 4 248 zł/mies. |
| 4% (400 os.) | 5 097 zł/mies. | 8 497 zł/mies. |

To nie są duże pieniądze — i to jest uczciwa informacja. Przy 10 tys. użytkowników masz projekt poboczny, nie firmę. Progi zaczynają mieć sens od ~100 tys. aktywnych.

---

## 7. Konkurencja na polskim rynku

**Paragony i gwarancje — zajęte, ale słabo.** W Google Play są co najmniej trzy polskie aplikacje: *Paragon – Twoje Gwarancje*, *ParagON: Paragony i Gwarancje*, *Gwarancje App*. Wszystkie są „skanerami z przypomnieniem", żadna nie interpretuje ani nie łączy z innymi obszarami domu. Nie wchodź tam z osobnym produktem — wejdź jako funkcja v3.

**Organizery rodzinne — zatłoczone.** *Domownik* (nazwa zajęta), *FamCal*, *Planado*, *Cozi*, *Nipto*. Wszystkie to kalendarze i listy zadań z grywalizacją. Żadna nie ma archiwum dokumentów ani interpretacji. Nie konkuruj z nimi na kalendarzu.

**Interpretacja pism urzędowych — tu jest luka.** Istniejące polskie narzędzia to albo B2G (*Proste Pismo* z PCSS i UAM, *Jasnopis* — upraszczają język **dla urzędów**, nie dla obywateli), albo czatowi asystenci prawni w przeglądarce (*Prawnicy.ai*, *LEX AI*, *Prawniczka.ai*, *LexTool*). **Nikt nie zajmuje pola „zrób zdjęcie pisma → termin + działanie + kalendarz + archiwum" na Androidzie.** To jest Twoja szczelina.

**Precedens prawny jest ustalony:** działające polskie produkty stosują formułę „charakter informacyjno-edukacyjny, nie stanowi porady prawnej i nie zastępuje adwokata". Ta droga jest przejezdna.

---

## 8. Ryzyka, w kolejności powagi

1. **Halucynowany termin.** Rozwiązanie: zasada cytatu z §4. Bez wyjątków.
2. **RODO.** Przetwarzasz dokumenty urzędowe, czyli potencjalnie dane zdrowotne, finansowe i sądowe. Potrzebujesz: hosting w UE, umowa powierzenia z dostawcą modelu, wyłączone trenowanie na danych (Vertex AI w regionie `europe-central2` to daje; publiczne API sprawdź osobno), polityka prywatności. To realna praca, nie formalność — i to jest największy ukryty koszt tego projektu.
3. **Odpowiedzialność.** Nigdy „podpisz / nie podpisuj", nigdy „nie musisz płacić". Zawsze „oto co dokument mówi".
4. **Platforma.** Gemini jest wbudowany w Androida i robi analizę zdjęcia darmo. Twoja obrona: share target (jedno kliknięcie), pamięć, kalendarz, polski kontekst.
5. **Dystrybucja.** Globalnie liczba instalacji spada, a wydatki w aplikacjach użytkowych rosną (+33,9% r/r w 2025, do 82,6 mld USD). Trudniej pozyskać, łatwiej zmonetyzować. Płatne pozyskanie odpada — musisz mieć kanał organiczny.

---

## 9. Dystrybucja — konkretnie

- **Share target jako wbudowany kanał.** Pismo przychodzi mailem → udostępnij → apka odpowiada. Jedyna trwała przewaga nad wklejeniem do czatu.
- **ASO na frazach problemowych, nie produktowych:** „co znaczy pismo z urzędu", „ile mam czasu na odpowiedź do urzędu", „jak wypowiedzieć umowę", „co to opłata dystrybucyjna".
- **Rolki:** format „wrzuciłam pismo z ZUS i okazało się, że mam 7 dni". Konkretny papier, konkretny termin, konkretna konsekwencja. Bardzo mocny hook, zero kosztu produkcji.
- **Grupy na Facebooku:** wspólnoty mieszkaniowe, rodzice, opieka nad rodzicami.
- **Kanał przez dzieci do seniorów** — ten sam mechanizm co przy „czy to scam".

---

## 10. Co zrobić w tym tygodniu

1. Zbierz **30 prawdziwych dokumentów** (pisma z urzędu, rachunki, umowy, wezwania). Bez tego zbioru nie masz jak zmierzyć jakości.
2. Przetestuj sam prompt w przeglądarce, bez pisania aplikacji. Sprawdź na tych 30 dokumentach: ile razy termin jest poprawny, ile razy model zmyśla.
3. Jeśli trafność terminów wychodzi poniżej ~95% — najpierw popraw prompt i schemę, nie zaczynaj kodu.
4. Sprawdź, czy nazwa jest wolna w Google Play (*Domownik* zajęty).

Punkt 2 to najtańszy sposób, żeby dowiedzieć się, czy ten produkt w ogóle da się zrobić. Dzień pracy, zero kodu.

---

## Uwaga o danych w tym dokumencie

Koszty API policzone dla cennika Gemini Flash-Lite ($0,10 / $0,40 za mln tokenów) — **model ma być wycofany 16.10.2026**, więc przelicz dla następcy przed startem. Tokenizacja obrazu oszacowana na 1500 tokenów za zdjęcie A4; zweryfikuj na własnych plikach, bo zależy od rozdzielczości. Kurs USD/PLN 3,7306 (NBP, 19.08.2026). Liczby o konwersji na płatnych (2–4%) to typowe widełki dla aplikacji użytkowych, nie dane dla tego produktu.
