# 04. Functional Requirements

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform \
**Dokument:** Wymagania funkcjonalne, przypadki użycia, wymagania GUI \
**Odbiorcy:** Zespół Deweloperski, Analitycy biznesowi, QA 

---

## 1. Aktorzy systemu

| Aktor | Opis |
|---|---|
| **Administrator systemu** | Zarządza konfiguracją globalną, kontami technicznymi, uprawnieniami (RBAC), harmonogramami, katalogiem platform. |
| **Analityk / Właściciel raportu** | Definiuje i utrzymuje Validation Profiles dla przypisanych raportów, przegląda wyniki i analizy AI dla swoich obszarów. |
| **Przeglądający (Viewer)** | Przegląda dashboard, historię, Health Score — bez prawa edycji konfiguracji. |
| **System (Scheduler)** | Aktor niebędący człowiekiem — automatycznie inicjuje wykonania testów zgodnie z harmonogramem. |
| **AI Analysis Engine** | Aktor wspierający — dostarcza interpretacje i sugestie w ramach zdefiniowanych granic (patrz `06_AI_Module.md`). |

---

## 2. Lista przypadków użycia

Pełny diagram graficzny — patrz `uml/UseCase.puml`. Poniżej lista referencyjna z ID używanym w wymaganiach funkcjonalnych.

| ID | Nazwa przypadku użycia | Aktor główny |
|---|---|---|
| UC-01 | Przegląd katalogu raportów wg obszarów biznesowych | Przeglądający, Analityk, Administrator |
| UC-02 | Wyszukiwanie raportu | Przeglądający, Analityk, Administrator |
| UC-03 | Definiowanie Validation Profile dla raportu | Analityk, Administrator |
| UC-04 | Definiowanie testu SQL | Analityk, Administrator |
| UC-05 | Definiowanie testu wartości pola (KPI) | Analityk, Administrator |
| UC-06 | Definiowanie reguły biznesowej | Analityk, Administrator |
| UC-07 | Ustawienie harmonogramu wykonania testów | Analityk, Administrator |
| UC-08 | Ręczne uruchomienie testu / grupy testów | Analityk, Administrator |
| UC-09 | Automatyczne wykonanie testów wg harmonogramu | System (Scheduler) |
| UC-10 | Przegląd wyniku pojedynczego przebiegu testowego | Przeglądający, Analityk, Administrator |
| UC-11 | Przegląd historii wyników raportu | Przeglądający, Analityk, Administrator |
| UC-12 | Przegląd Health Score raportu/obszaru | Przeglądający, Analityk, Administrator |
| UC-13 | Przegląd Confidence Score odczytu | Analityk, Administrator |
| UC-14 | Filtrowanie i przegląd błędów | Przeglądający, Analityk, Administrator |
| UC-15 | Przegląd analizy i wyjaśnienia wygenerowanego przez AI | Przeglądający, Analityk, Administrator |
| UC-16 | Otrzymanie powiadomienia o wykrytym problemie | Analityk, Administrator |
| UC-17 | Zarządzanie użytkownikami i rolami (RBAC) | Administrator |
| UC-18 | Zarządzanie katalogiem platform / connectorów | Administrator |
| UC-19 | Przegląd logu audytowego | Administrator |
| UC-20 | Przegląd metryk operacyjnych systemu (Monitoring) | Administrator |

---

## 3. Wymagania funkcjonalne

Priorytety wg MoSCoW: **M**ust have / **S**hould have / **C**ould have / **W**on't have (na tym etapie).

### 3.1 Katalog raportów i nawigacja (powiązane UC-01, UC-02)

| ID | Wymaganie | Priorytet |
|---|---|---|
| FR-001 | System musi prezentować raporty pogrupowane według obszarów biznesowych (zakładek), odzwierciedlając strukturę istniejącej Platformy BI. | M |
| FR-002 | System musi umożliwiać wyszukiwanie raportu po nazwie, adresie, obszarze biznesowym i typie platformy. | M |
| FR-003 | System musi rozróżniać i jednoznacznie oznaczać typ platformy raportu (Power BI / SSRS) w każdym miejscu prezentacji. | M |
| FR-004 | System powinien umożliwiać filtrowanie katalogu wg statusu ostatniego testu (OK / Błąd / Ostrzeżenie / Brak testów). | S |

### 3.2 Definiowanie testów i Validation Profile (powiązane UC-03–UC-07)

| ID | Wymaganie | Priorytet |
|---|---|---|
| FR-010 | System musi umożliwiać utworzenie Validation Profile dla dowolnego raportu z katalogu, zawierającego: adres, typ platformy, zestaw testów, sposób odnajdywania KPI, zapytania SQL, dopuszczalne odchylenia, reguły biznesowe, częstotliwość. | M |
| FR-011 | System musi wymuszać, że raport bez zdefiniowanego co najmniej jednego testu nie jest objęty automatycznym harmonogramem (zgodnie z założeniem projektowym — testy muszą być zdefiniowane). | M |
| FR-012 | System musi umożliwiać definiowanie testów typu SQL (zapytanie do hurtowni + oczekiwany wynik/zakres). | M |
| FR-013 | System musi umożliwiać definiowanie testów typu "wartość pola raportu" (wskazanie elementu KPI + oczekiwana wartość lub reguła porównania z SQL). | M |
| FR-014 | System musi umożliwiać zdefiniowanie dopuszczalnego odchylenia (tolerancji) dla porównań liczbowych (wartość bezwzględna i/lub procentowa). | M |
| FR-015 | System musi umożliwiać definiowanie reguł biznesowych wykraczających poza proste porównanie (np. sumy kontrolne, zakresy, trend historyczny). | M |
| FR-016 | System musi umożliwiać administratorowi aktualizację (wersjonowaną) istniejącego Validation Profile, z zachowaniem historii poprzednich wersji. | M |
| FR-017 | System musi umożliwiać ustawienie indywidualnej częstotliwości wykonania testów per raport (np. co godzinę, dziennie, tygodniowo, na żądanie). | M |
| FR-018 | System powinien umożliwiać kopiowanie/klonowanie Validation Profile jako punkt startowy dla podobnego raportu. | S |
| FR-019 | AI Analysis Engine może zaproponować wstępną konfigurację Validation Profile (np. wykryte kandydackie elementy KPI) — wymaga akceptacji człowieka przed aktywacją (patrz `06_AI_Module.md`). | C |

### 3.3 Wykonanie testów (powiązane UC-08, UC-09)

| ID | Wymaganie | Priorytet |
|---|---|---|
| FR-020 | System musi automatycznie wykonywać testy zgodnie z harmonogramem zdefiniowanym w Validation Profile, bez interakcji użytkownika. | M |
| FR-021 | System musi umożliwiać ręczne uruchomienie testu dla pojedynczego raportu na żądanie użytkownika uprawnionego. | M |
| FR-022 | System musi umożliwiać ręczne uruchomienie testów dla grupy raportów (np. cały obszar biznesowy) na żądanie. | S |
| FR-023 | System musi wykonywać testy równolegle (worker pool), z konfigurowalnym limitem współbieżności, aby zapewnić realizację pełnego cyklu dla 1000 raportów w oknie czasowym określonym w `05_NonFunctional_Requirements.md`. | M |
| FR-024 | System musi rejestrować pełny cykl życia przebiegu testowego (start, poszczególne etapy, zakończenie, czas trwania) niezależnie od jego wyniku (w tym w przypadku awarii/timeoutu). | M |

### 3.4 Wyniki, historia i metryki (powiązane UC-10–UC-14)

| ID | Wymaganie | Priorytet |
|---|---|---|
| FR-030 | System musi prezentować wynik każdego przebiegu testowego ze statusem (PASS / FAIL / WARNING / ERROR technicznym) i szczegółami (wartości porównywane, odchylenie, użyta metoda ekstrakcji). | M |
| FR-031 | System musi przechowywać pełną historię wyników dla każdego raportu, przeszukiwalną wg zakresu dat. | M |
| FR-032 | System musi wyliczać i prezentować **Health Score** — zagregowany wskaźnik jakości raportu w czasie (per raport, per obszar biznesowy, globalnie). | M |
| FR-033 | System musi wyliczać i prezentować **Confidence Score** dla każdej odczytanej wartości, informujący o pewności metody ekstrakcji. | M |
| FR-034 | System musi umożliwiać filtrowanie listy błędów wg obszaru biznesowego, platformy, typu błędu, zakresu dat, statusu. | M |
| FR-035 | System powinien umożliwiać eksport wyników (np. CSV) do celów raportowania zarządczego poza systemem. | S |

### 3.5 AI i wsparcie decyzyjne (powiązane UC-15)

| ID | Wymaganie | Priorytet |
|---|---|---|
| FR-040 | System musi prezentować wygenerowane przez AI wyjaśnienie każdego wykrytego błędu/anomalii w języku naturalnym, wraz z uzasadnieniem opartym na konkretnych danych (nigdy gołosłowne). | M |
| FR-041 | System musi grupować/klastrować podobne błędy w celu redukcji szumu informacyjnego (np. "17 raportów w obszarze Kredyty wskazuje ten sam wzorzec błędu"). | S |
| FR-042 | System musi jednoznacznie oznaczać w GUI, że dana treść pochodzi od AI (interpretacja wspierająca), odróżniając ją wizualnie od deterministycznego werdyktu Validation Engine. | M |
| FR-043 | System nie może pozwolić AI na zmianę statusu PASS/FAIL wygenerowanego przez Validation Engine — wymaganie o charakterze ograniczającym (guardrail), zweryfikowane testami akceptacyjnymi. | M |

### 3.6 Powiadomienia (powiązane UC-16)

| ID | Wymaganie | Priorytet |
|---|---|---|
| FR-050 | System musi wysyłać powiadomienia do wskazanych odbiorców po wykryciu błędu/rozbieżności zgodnie z regułami zdefiniowanymi w Validation Profile lub globalnie. | M |
| FR-051 | System musi umożliwiać deduplikację powiadomień (np. brak ponownego alertu o tym samym, wciąż trwającym problemie w każdym cyklu harmonogramu). | S |
| FR-052 | System powinien umożliwiać konfigurację progu eskalacji (np. powiadomienie do przełożonego po N nieudanych cyklach z rzędu). | C |

### 3.7 Administracja, bezpieczeństwo i audyt (powiązane UC-17–UC-20)

| ID | Wymaganie | Priorytet |
|---|---|---|
| FR-060 | System musi wspierać model RBAC z co najmniej rolami: Administrator, Analityk/Właściciel, Przeglądający. | M |
| FR-061 | System musi rejestrować w logu audytowym każdą zmianę konfiguracji/Validation Profile wraz z autorem i znacznikiem czasu. | M |
| FR-062 | System musi umożliwiać administratorowi przegląd logu audytowego z poziomu GUI. | M |
| FR-063 | System musi umożliwiać administratorowi zarządzanie listą platform/connectorów oraz podstawowymi parametrami połączeń (bez ekspozycji sekretów w GUI). | M |
| FR-064 | System musi eksponować metryki operacyjne (stan worker pool, czas wykonania, wskaźnik błędów) dostępne dla zespołu utrzymania. | M |

---

## 4. Wymagania GUI — opis ekranów

Poniższy opis stanowi wymagania funkcjonalne warstwy prezentacji na poziomie koncepcyjnym (nie jest to projekt graficzny/UI design — ten powstaje w odrębnym etapie projektowym z zespołem UX).

### 4.1 Ekran główny — Dashboard

- Podsumowanie stanu całego katalogu: liczba raportów OK / z błędem / z ostrzeżeniem / bez zdefiniowanych testów.
- Zagregowany Health Score globalny i per obszar biznesowy (wizualizacja trendu w czasie).
- Lista/kafelki najnowszych wykrytych problemów wymagających uwagi, posortowane wg istotności (z udziałem priorytetyzacji AI — patrz `06_AI_Module.md`).
- Skróty nawigacyjne do Katalogu Raportów, Historii, Konfiguracji.

### 4.2 Katalog Raportów (Report Catalog)

- Widok listy/siatki raportów, grupowanie wg obszarów biznesowych (zakładek) odzwierciedlających strukturę Platformy BI.
- Filtrowanie: platforma (PBI/SSRS), status ostatniego testu, obszar biznesowy, właściciel.
- Wyszukiwarka pełnotekstowa (nazwa, adres, tagi).
- Widok szczegółu raportu: metadane, aktualny Validation Profile, ostatnie wyniki, Health Score, link do historii.

### 4.3 Konfiguracja Validation Profile

- Formularz definicji: dane raportu (readonly, z katalogu), lista testów (SQL / wartość pola), harmonogram, tolerancje, reguły biznesowe.
- Edytor testu SQL z podglądem/walidacją składni przed zapisem (Test Definition Engine).
- Edytor testu wartości pola — wskazanie elementu KPI (wspierany podglądem raportu / listą wykrytych elementów przez Value Extraction Engine w trybie podglądu).
- Historia wersji profilu z możliwością porównania (diff) i przywrócenia poprzedniej wersji.

### 4.4 Uruchamianie testów

- Możliwość ręcznego uruchomienia testu dla pojedynczego raportu lub grupy (obszar biznesowy / wybór wielokrotny).
- Podgląd statusu w toku (live/near-live), z możliwością odświeżenia.

### 4.5 Historia i wyniki

- Chronologiczna lista przebiegów testowych per raport, z możliwością rozwinięcia szczegółów (porównywane wartości, użyta metoda ekstrakcji, Confidence Score).
- Wykres trendu Health Score w czasie.
- Eksport danych historycznych.

### 4.6 Analiza AI (AI Insights)

- Lista wygenerowanych przez AI wyjaśnień/anomalii, wyraźnie oznaczonych jako treść wspierająca (nie decyzyjna).
- Grupowanie podobnych błędów (klastry) z rekomendowanymi działaniami.
- Możliwość oznaczenia przez użytkownika trafności sugestii AI (feedback loop do dalszego doskonalenia promptów/modelu — patrz `06_AI_Module.md`).

### 4.7 Panel administracyjny

- Zarządzanie użytkownikami i rolami.
- Zarządzanie katalogiem platform/connectorów.
- Podgląd logu audytowego z filtrowaniem.
- Podgląd metryk operacyjnych (link/embed do Grafany lub widok uproszczony wewnątrz aplikacji).

---

## 5. Powiązanie z dalszą dokumentacją

- Model danych wspierający powyższe funkcje — `07_Data_Model.md`.
- Kontrakty API realizujące powyższe przypadki użycia — `08_API.md`.
- Diagramy UML (Use Case, Activity, Sequence) — katalog `uml/`.
- Wymagania niefunkcjonalne (wydajność wykonania testów przy skali 1000 raportów) — `05_NonFunctional_Requirements.md`.
