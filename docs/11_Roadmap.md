# 11. Roadmap

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform \
**Dokument:** Plan wdrożenia w fazach \
**Odbiorcy:** Project Manager, Sponsor, Architektura IT, Zespół Deweloperski

---

## 1. Podejście do wdrożenia

Wdrożenie ARGUS jest planowane **fazowo, w modelu przyrostowym (incremental delivery)**, minimalizując ryzyko i skracając czas do pierwszej wartości biznesowej. Każda faza dostarcza działający, użyteczny przyrost systemu, zamiast jednorazowego, długiego cyklu "big bang". Kolejność faz odzwierciedla zależności techniczne zidentyfikowane w `00_Technical_Feasibility_Study.md` (najpierw potwierdzenie fundamentów integracyjnych, dopiero potem skalowanie i funkcje zaawansowane).

---

## 2. Faza 0 — Discovery & Proof of Concept

**Cel:** Zweryfikować w praktyce kluczowe założenia techniczne z `00_Technical_Feasibility_Study.md` na realnym środowisku organizacji, zanim podjęte zostaną nieodwracalne decyzje architektoniczne.

| Zadanie | Rezultat |
|---|---|
| Potwierdzenie modelu wdrożenia Power BI (on-prem / hybrydowy) i dostępnych licencji (Premium/PPU) | Odpowiedź na otwarte pytania z `00_Technical_Feasibility_Study.md`, sekcja 8 |
| Potwierdzenie mechanizmu SSO (Kerberos/NTLM) i modelu konta technicznego | Zatwierdzony model autoryzacji dla connectorów |
| PoC: Browser Automation (Playwright) — otwarcie po jednym reprezentatywnym raporcie PBI i SSRS, odczyt przykładowego KPI | Potwierdzenie wykonalności technicznej warstwy integracyjnej |
| PoC: pomiar rzeczywistego czasu otwarcia/renderowania raportów | Kalibracja założeń wydajnościowych z `05_NonFunctional_Requirements.md` |
| Ustalenie polityk sieciowych i dostępu do magazynu sekretów | Odblokowanie prac Fazy 1 |
| Decyzja Security Office ws. dostawcy modelu AI (Azure OpenAI vs. on-prem) | Wejście do `06_AI_Module.md`, sekcja 5.3 |
| Akceptacja niniejszej dokumentacji architektonicznej przez Architekturę IT | Formalne rozpoczęcie developmentu |

**Kryterium wyjścia z Fazy 0:** Wszystkie otwarte pytania z `00_Technical_Feasibility_Study.md`, sekcja 8, mają udzieloną odpowiedź; PoC potwierdza wykonalność Browser Automation w środowisku organizacji.

---

## 3. Faza 1 — MVP (Minimum Viable Product)

**Cel:** Dostarczyć działający rdzeń systemu dla ograniczonego, reprezentatywnego podzbioru raportów (jeden obszar biznesowy), dowodzący pełnego cyklu: definicja testu → wykonanie → wynik → prezentacja.

| Zakres | Szczegóły |
|---|---|
| Moduły rdzeniowe | Test Orchestrator, Report Platform Connector Layer (PowerBIConnector, SSRSConnector — warstwa Browser Automation), Value Extraction Engine (Warstwa 0–1: API/model danych + DOM, bez CV/OCR), Validation Engine, SQL Validation Engine, Results Repository, Configuration/Validation Profile Repository |
| GUI | Katalog raportów (jeden obszar biznesowy), konfiguracja Validation Profile (podstawowa), przegląd wyników, historia |
| Harmonogram | Scheduler + wykonanie sekwencyjne/ograniczenie równoległości (bez pełnego autoscalingu) |
| Bezpieczeństwo | RBAC podstawowy, SSO, konto techniczne, magazyn sekretów |
| AI | Poza zakresem MVP (patrz Faza 3) |
| Audyt/Monitoring | Podstawowy log audytowy, podstawowe metryki (bez pełnego Grafana/OpenTelemetry) |

**Kryterium wyjścia:** System automatycznie testuje wszystkie raporty jednego wybranego obszaru biznesowego, zgodnie z harmonogramem, z poprawnym porównaniem wartości raport↔SQL i prezentacją wyniku w GUI.

---

## 4. Faza 2 — Scale-out i pełna funkcjonalność testowa

**Cel:** Rozszerzenie na cały bieżący katalog (~300 raportów), pełna funkcjonalność definiowania testów i reguł biznesowych, wprowadzenie równoległości.

| Zakres | Szczegóły |
|---|---|
| Rozszerzenie katalogu | Objęcie wszystkich obszarów biznesowych, pełna migracja/inwentaryzacja katalogu raportów |
| Test Definition Engine | Pełna walidacja definicji testów, klonowanie profili (FR-018), edytor testów SQL z podglądem |
| Business Rule Engine | Pełny zestaw typów reguł (sumy kontrolne, zakresy, trend) |
| Value Extraction Engine | Dodanie Warstwy 1b (eksport strukturalny) i Warstwy 2 (Computer Vision) fallbacku |
| Wykonanie równoległe | Worker pool Celery, konfigurowalna współbieżność, mechanizmy odpornościowe (retry, circuit breaker) |
| Notification Service | Pełna funkcjonalność (deduplikacja, reguły eskalacji) |
| Health Score / Confidence Score | Pełne wyliczanie i prezentacja trendów |
| Monitoring | Pełna integracja Prometheus/Grafana/OpenTelemetry |

**Kryterium wyjścia:** Pełny katalog ~300 raportów objęty automatycznym, cyklicznym testowaniem, z równoległym wykonaniem i pełnym monitoringiem operacyjnym.

---

## 5. Faza 3 — Moduł AI

**Cel:** Wprowadzenie warstwy interpretacyjnej AI zgodnie z granicami opisanymi w `06_AI_Module.md`.

| Zakres | Szczegóły |
|---|---|
| AI Analysis Engine | Context Builder, LLM Orchestration Layer, Response Validator |
| Integracja z dostawcą modelu | Zgodnie z decyzją Fazy 0 (Azure OpenAI / on-prem) |
| Wyjaśnialność | Wymuszenie strukturalnej odpowiedzi z polem `evidence`, logowanie kontekstu wywołań (NFR-C-04) |
| Anomaly Detector | Detekcja odchyleń od wzorca historycznego |
| Clustering Engine | Grupowanie podobnych błędów (FR-041) |
| GUI | Ekran AI Insights, wizualne oznaczenie treści AI (NFR-U-02), mechanizm feedbacku użytkownika |
| Guardraile bezpieczeństwa | Testy akceptacyjne potwierdzające niemożność zmiany statusu PASS/FAIL przez AI (FR-043) |

**Kryterium wyjścia:** Moduł AI generuje wyjaśnialne interpretacje dla przebiegów FAIL/WARNING, zweryfikowane pod kątem zgodności z zasadą nadrzędną (AI nie decyduje), zaakceptowane przez Security Office.

---

## 6. Faza 4 — Pełne wdrożenie produkcyjne (skala 1000 raportów)

**Cel:** Osiągnięcie docelowej skali i pełnej gotowości operacyjnej klasy Enterprise.

| Zakres | Szczegóły |
|---|---|
| Skalowanie infrastruktury | Pełny autoscaling worker pool, weryfikacja NFR-P-01 (≤4h dla 1000 raportów) w warunkach zbliżonych do produkcyjnych |
| Wysoka dostępność | Pełna redundancja komponentów, weryfikacja SLA (NFR-A-01) |
| Disaster Recovery | Pełne wdrożenie i testy procedur DR (patrz `10_Deployment.md`, sekcja 6) |
| Bezpieczeństwo | Pełny audyt bezpieczeństwa, testy penetracyjne (NFR-SEC-06) |
| Dokumentacja operacyjna | Runbooki dla zespołu utrzymania, procedury eskalacji |
| Rollout | Stopniowe włączanie kolejnych raportów/obszarów aż do pełnego pokrycia katalogu (docelowo ≥1000) |

**Kryterium wyjścia:** System w pełni produkcyjny, obsługujący docelową skalę raportów, zaakceptowany przez wszystkich interesariuszy z `01_Project_Overview.md`, sekcja 5.

---

## 7. Faza 5 — Ciągłe doskonalenie (Continuous Improvement)

Faza nieograniczona w czasie, obejmująca:

- Rozszerzanie katalogu raportów wraz z jego naturalnym wzrostem organizacyjnym (powyżej 1000).
- Iteracyjne doskonalenie promptów/modelu AI na bazie zebranego feedbacku użytkowników (FR-041, mechanizm oceny trafności sugestii).
- Ewentualne dodanie kolejnych connectorów platform raportowych zgodnie z potrzebami biznesowymi (architektonicznie przewidziane — patrz `03_Architecture.md`, sekcja 6).
- Optymalizacja kosztowa (infrastruktura, wywołania AI) na bazie rzeczywistych danych operacyjnych.
- Regularny przegląd i aktualizacja rejestru ryzyk (`12_Risks.md`).

---

## 8. Zależności między fazami (skrót)

```
Faza 0 (Discovery/PoC)
   │  [wymagane: potwierdzenie wykonalności technicznej]
   ▼
Faza 1 (MVP — 1 obszar biznesowy)
   │  [wymagane: działający rdzeń end-to-end]
   ▼
Faza 2 (Scale-out — cały bieżący katalog ~300)
   │  [wymagane: równoległość, pełne testy biznesowe]
   ├──────────────┐
   ▼              ▼
Faza 3 (AI)   Faza 4 (Pełna skala 1000, HA, DR)
   │              │
   └──────┬───────┘
          ▼
   Faza 5 (Continuous Improvement)
```

Fazy 3 i 4 mogą być realizowane częściowo równolegle przez odrębne strumienie zespołu, ponieważ nie mają twardej zależności technicznej między sobą (moduł AI jest architektonicznie non-blocking względem rdzenia systemu — patrz `06_AI_Module.md`, sekcja 5.1) — decyzja o równoległej realizacji należy do Project Managera na bazie dostępności zasobów zespołu.

---

## 9. Powiązanie z dalszą dokumentacją

- Uzasadnienie biznesowe wdrożenia fazowego — `02_Business_Problem.md`.
- Kryteria sukcesu projektu jako całości — `01_Project_Overview.md`, sekcja 9.
- Ryzyka mogące wpłynąć na harmonogram powyższych faz — `12_Risks.md`.
