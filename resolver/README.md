# Resolver — warstwa, która liczy

Resolver zamienia symbol z parsera na konkretne daty. Jest deterministyczny, testowalny i darmowy. **Wszystko, co przeniesiesz stąd do modelu, staje się źródłem błędów.**

## Podział pracy

| Zadanie | Kto |
|---|---|
| Rozpoznanie rodzaju wypowiedzi, obiektu, wyrażeń czasowych | model |
| Wykrycie niejednoznaczności i sprzeczności | model |
| Wyliczenie konkretnych dat z symboli | **kod** |
| Policzenie liczby wystąpień w serii | **kod** |
| Sprawdzenie kolizji ze świętami i dniami wolnymi | **kod** |
| Materializacja: reguła plus dwa najbliższe wystąpienia | **kod** |
| Godziny domyślne, przypomnienia, czas trwania | **kod** |

---

## Co resolver dostaje na wejściu

Pięć typów kotwicy z [`../parser/schema.json`](../parser/schema.json):

| Typ | Wejście | Co policzyć |
|---|---|---|
| `data_wprost` | dzień, miesiąc, rok (rok może być `null`) | przy `rok: null` wybierz najbliższe przyszłe wystąpienie |
| `dzien_tygodnia` | dzień + `ktory` | `najblizszy` → pierwszy taki dzień od jutra; `przyszly_tydzien` → +7 od tego |
| `przesuniecie` | ile + jednostka | uwaga na miesiące: 31.01 + 1 miesiąc = 28.02, nie 03.03 |
| `przebieg` | km lub km_delta | **nie zamieniaj na czas**; termin wypada przy stanie licznika |
| `brak` | — | wpis ląduje na liście „bez terminu", apka pokazuje kalendarz |

Jeśli `ktory: "niejasne"` — nie licz nic, pokaż pytanie z tablicy `niejednoznacznosc`.

---

## Reguły, które łatwo przeoczyć

**Jednostek nie wolno normalizować.** „Co 90 dni" to nie „co 3 miesiące". 12.06.2026 + 90 dni = **10.09.2026**, a + 3 miesiące = **12.09.2026**. Użytkownik zauważy różnicę.

**Kolizje z dniami wolnymi.** Po wygenerowaniu wystąpień sprawdź każde w [`swieta-pl.json`](swieta-pl.json). Przy kolizji zaproponuj przesunięcie, nie przesuwaj po cichu.

> Przykład z testów: wizyty co dwa tygodnie od 21.08.2026 dają 14 terminów, z których jeden wypada 25.12 — w Boże Narodzenie.

**24 grudnia jest dniem ustawowo wolnym od 2025 roku** (ustawa z 6.12.2024). Większość gotowych bibliotek kalendarzowych tego nie uwzględnia. Tabela w tym repo uwzględnia.

**Data ustalona a propozycja apki.** Jeśli termin pochodzi z domysłu aplikacji (np. „przegląd auta" bez daty → roczny cykl z archiwum), musi być oznaczony w modelu danych jako propozycja i wyglądać inaczej w interfejsie. Inaczej po pół roku użytkownik nie wie, które terminy sam podał, i przestaje wierzyć wszystkim.

---

## Dane

[`swieta-pl.json`](swieta-pl.json) — polskie dni ustawowo wolne, lata 2026–2030, 14 dni rocznie. Święta ruchome (Wielkanoc, Poniedziałek Wielkanocny, Zielone Świątki, Boże Ciało) wyliczone algorytmem Meeusa dla Wielkanocy zachodniej.

Weryfikacja: Wielkanoc 2026 — 5 kwietnia, 2027 — 28 marca, 2028 — 16 kwietnia.

Tabelę trzeba przejrzeć przy każdej zmianie ustawy o dniach wolnych od pracy. To jedyna dana w projekcie wymagająca ręcznego utrzymania.

---

## Jak przetestować parser przed pisaniem kodu

To jest następny krok w projekcie i kosztuje jeden dzień pracy.

1. Wejdź w Google AI Studio (darmowe)
2. Wklej [`../parser/prompt.txt`](../parser/prompt.txt) jako instrukcję systemową
3. Ustaw [`../parser/schema.json`](../parser/schema.json) jako wymuszone wyjście (structured output)
4. Podstaw kontekst: dzisiejszą datę, dzień tygodnia, listę obiektów
5. Przepuść zdania z [`../parser/przypadki-testowe.json`](../parser/przypadki-testowe.json), po jednym

**Licz tylko dwie rzeczy:**

- ile razy model **wymyślił datę**, której w zdaniu nie było
- ile razy **rozstrzygnął zamiast zapytać**

**Progi zielonego światła:** zero wymyślonych dat, maksymalnie 1 na 20 nieuzasadnionych rozstrzygnięć. Pierwsza liczba musi być zerem — to jedyny błąd tego systemu, który realnie kosztuje użytkownika przegapiony termin.

Jeśli progi nie wychodzą, poprawiaj prompt. Nie zaczynaj kodu.
