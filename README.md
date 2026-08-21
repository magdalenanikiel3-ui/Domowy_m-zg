# Domowy mózg

Aplikacja na Androida, która pamięta domowe sprawy za Ciebie: papiery, terminy, gwarancje, przeglądy i to, kiedy ostatnio coś zrobiłaś.

**Stan projektu:** faza projektowania. Kodu jeszcze nie ma. W repo są analiza rynku, specyfikacja MVP i działające artefakty parsera gotowe do przetestowania.

---

## O co chodzi

Wszystko, co dzieje się w domu, sprowadza się do jednej struktury:

> **obiekt → zdarzenia z datą + dokumenty**

| Sprawa | Obiekt | Zdarzenie | Dokument |
|---|---|---|---|
| Gwarancja na pralkę | pralka | do 14.08.2028 | paragon |
| OC | auto | odnowienie 12.10 | polisa |
| Pismo z urzędu | sprawa | odpowiedź do 28.08 | skan |
| Odkamienianie ekspresu | ekspres | zrobione 12.06, co 90 dni | — |
| Maskotka do szkoły | dziecko | piątek 22.08 | — |

To nie jest pięć aplikacji. To jedna tabela. Cały produkt wynika z tej obserwacji.

## Dwie zasady, które trzymają całość

**1. Archiwum jest efektem ubocznym, nie funkcją.**
Użytkownik nigdy nie „wprowadza danych". Robi zdjęcie pisma, bo musi się nim zająć — a wpis w archiwum i termin w kalendarzu powstają przy okazji. Każda przyszła funkcja musi przejść test: *czy dane wchodzą bez tego, żeby użytkownik postanowił je wprowadzić?*

**2. Model nie liczy dat. Kod liczy.**
Model językowy zwraca symbol („piątek, najbliższy"), a konkretną datę wylicza deterministyczny resolver. Modele mylą się w arytmetyce kalendarzowej, a błędna data to jedyny błąd tego systemu, który realnie kosztuje użytkownika przegapiony termin.

## Kolejność wydawania

| Wersja | Zakres |
|---|---|
| **v1** | Zdjęcie papieru → co to znaczy, co zrobić, do kiedy → termin do kalendarza → papier w archiwum |
| **v2** | Szybkie wrzucanie głosem + pytania do archiwum („kiedy ostatnio wymieniałam olej?") |
| **v3** | Cykliczne czynności, gwarancje z paragonów |
| **v4** | Wspólne archiwum rodziny |

Warunek bezwzględny: **v1 musi być użyteczny dla osoby samotnej.** Jeśli sens pojawia się dopiero, gdy dołączy rodzina — produkt umrze na starcie.

---

## Co jest w repo

```
docs/
  01-analiza-produktu.md   analiza rynku, konkurencja, koszty, model cenowy
  02-roadmapa.md           zakres v1–v4, co świadomie wycięte
  03-decyzje.md            kluczowe decyzje projektowe z uzasadnieniem

parser/
  prompt.txt               prompt systemowy, gotowy do wklejenia
  schema.json              wymuszona struktura odpowiedzi
  testy.md                 raport z testów na 20 trudnych zdaniach
  przypadki-testowe.json   te same zdania, maszynowo

resolver/
  README.md                co musi policzyć kod, a czego nie wolno modelowi
  swieta-pl.json           polskie dni wolne 2026–2030
```

## Następny krok

Przetestować parser na modelu, którego realnie użyjesz — instrukcja w [`resolver/README.md`](resolver/README.md) i [`parser/testy.md`](parser/testy.md).

Próg zielonego światła: **zero wymyślonych dat** i maksymalnie **1 na 20** nieuzasadnionych rozstrzygnięć zamiast pytania. Poniżej tego progu nie ma sensu pisać kodu.
