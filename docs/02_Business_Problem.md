# 02. Business Problem

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform \
**Dokument:** Analiza problemu biznesowego i uzasadnienie inwestycji \
**Odbiorcy:** Sponsor, Project Manager, Interesariusze biznesowi

---

## 1. Kontekst biznesowy

Organizacja utrzymuje wewnętrzną **Platformę BI** stanowiącą centralny punkt dostępu do raportów zarządczych i operacyjnych. Platforma obejmuje obecnie **ok. 300 raportów**, pogrupowanych tematycznie według obszarów biznesowych (zakładek — np. Sprzedaż, Kredyty, Ryzyko), zbudowanych w dwóch technologiach:

- **Microsoft Power BI** (`https://pbi.bank.com.pl/reports/powerbi/...`),
- **SQL Server Reporting Services / Report Server** (`https://bgkssrs/Reports/report/...`).

Trajektoria wzrostu liczby raportów wskazuje na docelowy katalog **co najmniej 1000 raportów**, co oznacza ponad trzykrotny wzrost względem stanu obecnego — bez odpowiadającego wzrostu zasobów ludzkich dedykowanych do ich kontroli jakości.

## 2. Obecny proces (AS-IS)

Weryfikacja poprawności raportu odbywa się dziś **w pełni manualnie**, według powtarzalnego, lecz nieskalowalnego schematu:

1. Analityk otwiera raport w przeglądarce.
2. Czeka na jego załadowanie (czas zależny od złożoności raportu i obciążenia platformy).
3. Wizualnie ocenia, czy raport w ogóle się otworzył i wyrenderował bez błędów.
4. Odczytuje wzrokiem jedną lub kilka wartości KPI z raportu.
5. Otwiera osobne narzędzie (np. SSMS) i uruchamia odpowiadające zapytanie SQL na hurtowni danych.
6. Porównuje wynik zapytania z wartością odczytaną z raportu — ręcznie, "na oko" lub w arkuszu kalkulacyjnym.
7. Przechodzi do kolejnego raportu i powtarza cały cykl.

### 2.1 Charakterystyka procesu

| Cecha | Opis |
|---|---|
| Czasochłonność | Pojedynczy cykl (otwarcie + odczyt + zapytanie SQL + porównanie) trwa realistycznie kilka–kilkanaście minut na raport, w zależności od złożoności. |
| Powtarzalność | Proces identyczny dla każdego raportu, a mimo to wykonywany ręcznie za każdym razem — klasyczny kandydat do automatyzacji. |
| Podatność na błędy | Porównanie "na oko" wartości liczbowych (formatowanie, zaokrąglenia, jednostki) jest podatne na przeoczenia, zwłaszcza przy wielogodzinnej, monotonnej pracy. |
| Brak śladu audytowego | Wynik ręcznej weryfikacji zwykle nie jest nigdzie systematycznie rejestrowany — nie istnieje historyczna, przeszukiwalna baza "czy i kiedy raport X był ostatnio poprawny". |
| Brak powtarzalnej częstotliwości | Ze względu na koszt czasowy, pełny przegląd katalogu raportów odbywa się rzadko lub wybiórczo, a nie w sposób ciągły. |
| Skalowalność | Czas procesu rośnie **liniowo** wraz z liczbą raportów, podczas gdy zasoby ludzkie dedykowane do tego zadania nie rosną w tym samym tempie — przy 1000 raportach proces w obecnej formie staje się praktycznie niewykonalny w rozsądnym cyklu. |

## 3. Konsekwencje biznesowe obecnego stanu

### 3.1 Ryzyko operacyjne i decyzyjne

Raporty BI w środowisku bankowym są podstawą podejmowania decyzji biznesowych (sprzedażowych, kredytowych, ryzykowych). Błędny raport, który nie zostanie wykryty na czas, może prowadzić do:

- podjęcia decyzji biznesowej na podstawie nieaktualnych lub błędnych danych,
- utraty zaufania odbiorców biznesowych do Platformy BI jako całości (efekt "raz oszukany, zawsze podejrzliwy" — spadek adopcji nawet poprawnych raportów),
- wydłużonego czasu wykrycia i naprawy błędu, jeśli zostanie on zauważony dopiero przez użytkownika biznesowego, a nie przez IT/analityków w toku rutynowej kontroli.

### 3.2 Koszt operacyjny

Manualna weryfikacja skaluje się liniowo z liczbą raportów, co oznacza, że wzrost katalogu z 300 do 1000 raportów (założenie docelowe) **potroiłby** nakład pracy potrzebny do utrzymania obecnego poziomu kontroli — albo, co bardziej prawdopodobne przy niezmienionych zasobach, **proporcjonalnie obniżyłby rzeczywiste pokrycie kontrolą** (mniejszy odsetek raportów faktycznie regularnie sprawdzanych).

### 3.3 Brak wiedzy organizacyjnej o jakości danych w czasie

Brak systematycznego rejestrowania wyników weryfikacji oznacza, że organizacja nie dysponuje danymi do odpowiedzi na pytania typu: *"Który raport najczęściej ulega awarii?"*, *"Czy jakość raportów w obszarze Kredyty pogarsza się w czasie?"*, *"Jak długo trwało wykrycie ostatniej rozbieżności?"* — a to są pytania kluczowe dla governance danych i priorytetyzacji działań naprawczych.

## 4. Analiza przyczyn źródłowych (Root Cause)

Proces manualny nie jest wynikiem złej woli czy braku kompetencji zespołu — jest **strukturalnie nieskalowalny**, ponieważ:

1. Nie istnieje jeden, ujednolicony mechanizm dostępu programistycznego do "tego, co widzi użytkownik" w obu technologiach raportowych jednocześnie (stąd konieczność dogłębnej analizy technicznej — patrz `00_Technical_Feasibility_Study.md`).
2. Testy porównawcze (raport vs. hurtownia) wymagają wiedzy specjalistycznej (SQL, znajomość modelu danych) połączonej z dostępem do samego raportu — dziś skoncentrowanej w głowach pojedynczych analityków, a nie w reużywalnej, wersjonowanej konfiguracji.
3. Brak platformy, która pozwoliłaby **raz zdefiniować** regułę testową i **wielokrotnie, automatycznie** ją wykonywać w czasie.

## 5. Stan docelowy (TO-BE)

Po wdrożeniu ARGUS proces wygląda następująco:

1. Administrator/analityk definiuje **Validation Profile** dla raportu — jednorazowo, z możliwością aktualizacji w miarę zmian.
2. **Scheduler** automatycznie uruchamia testy zgodnie ze zdefiniowaną częstotliwością (lub na żądanie).
3. System **automatycznie** otwiera raport, sprawdza dostępność i renderowanie, odczytuje wartości KPI, wykonuje odpowiadające zapytania SQL do hurtowni i porównuje wyniki — bez udziału człowieka w pętli wykonawczej.
4. Wyniki są zapisywane w historii, agregowane w **Health Score**, a w razie wykrycia rozbieżności — moduł AI generuje czytelne wyjaśnienie i wskazuje prawdopodobną przyczynę, a **Notification Service** informuje odpowiednie osoby.
5. Analityk/administrator poświęca czas **wyłącznie** na przypadki faktycznie wymagające uwagi (odsiane i priorytetyzowane przez system), zamiast na rutynowe sprawdzanie raportów działających poprawnie.

## 6. Uzasadnienie inwestycji (Business Case — jakościowe)

| Wymiar | Stan obecny (AS-IS) | Stan docelowy (TO-BE z ARGUS) |
|---|---|---|
| Zakres kontroli | Wybiórczy, ograniczony przepustowością ludzką | Pełne pokrycie katalogu raportów (docelowo 1000) |
| Częstotliwość | Nieregularna, zależna od dostępności analityka | Regularna, harmonogramowana, konfigurowalna per raport |
| Czas wykrycia błędu | Godziny–dni (zależnie od priorytetu ręcznego przeglądu) | Minuty–godziny od wystąpienia (zgodnie z harmonogramem testów) |
| Ślad audytowy | Brak lub fragmentaryczny | Pełna, przeszukiwalna historia + audyt zmian konfiguracji |
| Koszt krańcowy dodania kolejnego raportu do kontroli | Rosnący liniowo wraz z liczbą raportów (koszt osobowy) | Marginalny (konfiguracja Validation Profile), niezależny od skali infrastruktury wykonawczej |
| Wiedza organizacyjna o jakości danych w czasie | Anegdotyczna | Ustrukturyzowana, mierzalna (Health Score, trendy) |

Niniejszy dokument stanowi jakościowe uzasadnienie biznesowe. Ilościowa estymacja kosztów wdrożenia i utrzymania (infrastruktura, zespół, licencje) powinna zostać wykonana przez Project Managera we współpracy z Architekturą IT na bazie stacku technologicznego zdefiniowanego w `09_Python_Architecture.md` oraz planu faz w `11_Roadmap.md`, jako odrębne ćwiczenie budżetowe — poza zakresem niniejszego dokumentu architektonicznego.

## 7. Powiązanie z dalszą dokumentacją

- Szczegółowa analiza techniczna sposobu realizacji — `00_Technical_Feasibility_Study.md`.
- Wymagania funkcjonalne wynikające z opisanego procesu — `04_Functional_Requirements.md`.
- Plan wdrożenia fazowego, minimalizujący ryzyko i czas do pierwszej wartości biznesowej — `11_Roadmap.md`.
- Rejestr ryzyk projektu, w tym ryzyk związanych z akceptacją zmiany procesu przez organizację — `12_Risks.md`.
