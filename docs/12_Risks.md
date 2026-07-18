# 12. Risks

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform
**Dokument:** Rejestr ryzyk
**Odbiorcy:** Sponsor, Project Manager, Architektura IT, Security Office

---

## 1. Metodologia

Każde ryzyko oceniane jest w skali **Prawdopodobieństwo × Wpływ** (Niskie / Średnie / Wysokie), z przypisanym właścicielem i strategią mitygacji. Rejestr podlega cyklicznemu przeglądowi (rekomendacja: na początku każdej fazy z `11_Roadmap.md` oraz ad-hoc przy istotnych zmianach).

---

## 2. Rejestr ryzyk

| ID | Ryzyko | Kategoria | Prawdop. | Wpływ | Mitygacja | Właściciel |
|---|---|---|---|---|---|---|
| R-01 | Zmiana layoutu/struktury DOM raportu przez Microsoft (aktualizacja produktu Power BI/SSRS) psuje selektory ekstrakcji na dużą skalę | Techniczne | Średnie | Wysoki | Strategia wielowarstwowego fallbacku (`00_Technical_Feasibility_Study.md`, sekcja 7), selektory semantyczne, automatyczna detekcja podejrzanej zmiany strukturalnej, wersjonowanie Validation Profile | Zespół Deweloperski |
| R-02 | Niedostępność/degradacja wydajności Platformy BI pod obciążeniem generowanym przez ARGUS (zbyt agresywny harmonogram testów) | Techniczne / Operacyjne | Średnie | Wysoki | Konfigurowalny limit współbieżności, Circuit Breaker, testy obciążeniowe w Fazie 0/2, harmonogramy poza godzinami szczytu tam, gdzie to możliwe | Zespół Deweloperski, Administrator Platformy BI |
| R-03 | Brak licencji Premium/PPU uniemożliwiający wykorzystanie connectora XMLA dla części raportów | Techniczne / Licencyjne | Wysokie | Niski | Architektura nie zakłada zależności od XMLA jako warstwy obowiązkowej — Browser Automation pozostaje warstwą bazową uniwersalną niezależnie od licencji (`00_Technical_Feasibility_Study.md`, sekcja 4) | Architektura IT |
| R-04 | Problemy z integracją SSO/Kerberos w kontekście headless przeglądarki (typowe wyzwanie w środowiskach on-prem z NTLM/Kerberos) | Techniczne | Średnie | Wysoki | Wczesna weryfikacja w Fazie 0 (PoC), ścisła współpraca z zespołem IAM, rezerwowy plan (dedykowane konto serwisowe z bezpośrednim dostępem, jeśli delegacja/impersonacja niedostępna) | Zespół Deweloperski, IAM |
| R-05 | Niewystarczające zaangażowanie administratorów/analityków w definiowanie Validation Profiles — niskie pokrycie katalogu testami | Organizacyjne | Średnie | Wysoki | Uproszczony UX konfiguracji, klonowanie profili (FR-018), wsparcie AI w propozycji wstępnej konfiguracji (FR-019), jasna komunikacja wartości biznesowej, wsparcie sponsora w promowaniu adopcji | Project Manager, Właściciele obszarów biznesowych |
| R-06 | Nadmierna liczba fałszywych alarmów (zbyt wąskie tolerancje / niska jakość ekstrakcji) prowadząca do utraty zaufania użytkowników biznesowych do systemu | Biznesowe / Techniczne | Średnie | Wysoki | Confidence Score jako wskaźnik jawnie prezentowany, iteracyjne strojenie tolerancji w Fazie 2, mechanizm feedbacku (ocena trafności) już od Fazy 3 | Analitycy biznesowi, Zespół Deweloperski |
| R-07 | Halucynacja lub niepożądana treść generowana przez moduł AI | Techniczne / Reputacyjne | Niskie–Średnie | Średni | Wymóg strukturalnej odpowiedzi z `evidence`, Response Validator, jednoznaczne oznaczenie treści AI, brak wpływu AI na status PASS/FAIL (`06_AI_Module.md`) | Zespół Deweloperski, Security Office |
| R-08 | Wyciek danych wrażliwych do zewnętrznego dostawcy modelu AI | Bezpieczeństwo / Compliance | Niskie | Wysoki | Minimalizacja danych przekazywanych do modelu, kontraktowe gwarancje dostawcy, opcja wariantu w pełni on-prem, decyzja Security Office w Fazie 0 | Security Office, Data Governance |
| R-09 | Niedoszacowanie infrastruktury wykonawczej (worker pool) względem rzeczywistego czasu renderowania raportów przy skali 1000 | Techniczne / Wydajnościowe | Średnie | Wysoki | Kalibracja empiryczna w PoC (Faza 0), autoscaling oparty o głębokość kolejki, monitoring trendu czasu wykonania od Fazy 1 | Architektura IT, Zespół Infrastruktury |
| R-10 | Opóźnienia w uzyskaniu dostępów sieciowych/kontowych w regulowanym środowisku bankowym | Organizacyjne / Proceduralne | Wysokie | Średni | Wczesne zgłoszenie wymagań dostępowych w Fazie 0, jasno zdefiniowana lista otwartych pytań (`00_Technical_Feasibility_Study.md`, sekcja 8) przekazana odpowiednim zespołom na starcie projektu | Project Manager |
| R-11 | Konto techniczne o nadmiernych uprawnieniach staje się celem/wektorem ataku | Bezpieczeństwo | Niskie | Wysoki | Least privilege (NFR-SEC-01), segmentacja sieciowa (NFR-SEC-08), zarządzany magazyn sekretów, regularny przegląd uprawnień | Security Office |
| R-12 | Rozrost wolumenu danych historycznych degraduje wydajność zapytań/Dashboardu szybciej niż zakładano | Techniczne | Średnie | Średni | Partycjonowanie od Fazy 2 (`07_Data_Model.md`, sekcja 4), monitoring wzrostu wolumenu, zdefiniowana polityka retencji/archiwizacji | Zespół Deweloperski, DBA |
| R-13 | Zmiana zakresu projektu w trakcie realizacji (np. objęcie dodatkowych platform raportowych wcześniej niż planowano) | Zarządcze | Średnie | Średni | Architektura Open/Closed minimalizuje koszt techniczny takiej zmiany (`03_Architecture.md`, sekcja 6); formalny proces zarządzania zmianą zakresu po stronie PM | Project Manager |
| R-14 | Niewystarczające testy kontraktowe connectorów prowadzą do regresji przy aktualizacjach zależności (np. nowa wersja Playwright) | Techniczne / Jakościowe | Średnie | Średni | NFR-M-03 (obowiązkowe testy kontraktowe), pipeline CI blokujący merge przy regresji, kontrolowany proces aktualizacji zależności | Zespół Deweloperski |
| R-15 | Rotacja/odejście kluczowych członków zespołu deweloperskiego w trakcie wieloletniego projektu | Organizacyjne | Średnie | Średni | Kompletna, aktualizowana dokumentacja architektoniczna (niniejsze repozytorium) jako źródło wiedzy niezależne od osób, wysokie pokrycie testami ułatwiające onboarding, code review jako mechanizm transferu wiedzy | Project Manager, Tech Lead |

---

## 3. Macierz Prawdopodobieństwo × Wpływ (skrót wizualny)

| | Wpływ: Niski | Wpływ: Średni | Wpływ: Wysoki |
|---|---|---|---|
| **Prawdop.: Wysokie** | — | R-10 | R-03* |
| **Prawdop.: Średnie** | — | R-12, R-13, R-14, R-15 | R-01, R-02, R-04, R-05, R-06, R-09 |
| **Prawdop.: Niskie** | — | R-07 | R-08, R-11 |

*\* R-03 ma wysokie prawdopodobieństwo wystąpienia ograniczenia licencyjnego, ale niski wpływ, ponieważ architektura nie zakłada zależności od XMLA jako warstwy obowiązkowej.*

Ryzyka w prawym górnym i środkowym górnym obszarze macierzy (wysoki wpływ przy średnim/wysokim prawdopodobieństwie: **R-01, R-02, R-04, R-05, R-06, R-09**) wymagają najwyższej uwagi zespołu projektowego i powinny być monitorowane aktywnie od Fazy 0.

---

## 4. Powiązanie z dalszą dokumentacją

- Mitygacje techniczne szczegółowo opisane w `00_Technical_Feasibility_Study.md` (R-01, R-03) oraz `03_Architecture.md` (R-13, R-14).
- Otwarte pytania Fazy 0 bezpośrednio adresujące ryzyka R-03, R-04, R-08, R-10 — `00_Technical_Feasibility_Study.md`, sekcja 8.
- Wymagania niefunkcjonalne mitygujące ryzyka bezpieczeństwa i wydajności — `05_NonFunctional_Requirements.md`.
- Harmonogram faz, w którym poszczególne ryzyka są aktywnie adresowane — `11_Roadmap.md`.
