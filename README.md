# ARGUS
### Automated Reporting Governance & Unified Surveillance Platform

**Klasa systemu:** Enterprise / Mission-critical \
**Domena:** Bankowość — Platforma BI (Power BI, SQL Server Reporting Services) \
**Typ dokumentu:** Repozytorium dokumentacji architektonicznej i projektowej \
**Status:** Do akceptacji przez Menadżera oraz Dział IT \
**Wersja dokumentacji:** 1.0 \
**Data:** 2026-07-18 

---

## 1. O projekcie

W organizacji funkcjonuje wewnętrzna Platforma BI udostępniająca ok. 300 raportów (docelowo ≥ 1000), zbudowanych w technologii **Microsoft Power BI** oraz **SQL Server Reporting Services (SSRS)**. Obecnie poprawność raportów jest weryfikowana ręcznie przez analityków, co jest procesem czasochłonnym, podatnym na błędy i niemożliwym do skalowania wraz ze wzrostem liczby raportów.

**ARGUS** to platforma klasy enterprise, której zadaniem jest **autonomiczne, ciągłe i audytowalne monitorowanie jakości raportów BI** — dostępności, poprawności renderowania, zgodności wartości KPI ze źródłem danych (hurtownią) oraz reguł biznesowych — wspierana (ale nie zastępowana) przez moduł AI, który interpretuje wyniki, wykrywa anomalie i tłumaczy je zrozumiałym językiem dla użytkownika biznesowego.

---

## 2. Struktura repozytorium

```
BI-Quality-Assurance-Platform/
│
├── README.md                              ← ten dokument
│
├── docs/                                   ← dokumentacja architektoniczna (Markdown)
│   ├── 00_Technical_Feasibility_Study.md   ← analiza techniczna metod integracji i ekstrakcji danych
│   ├── 01_Project_Overview.md              ← wizja, cele, zakres, interesariusze
│   ├── 02_Business_Problem.md              ← problem biznesowy i uzasadnienie inwestycji
│   ├── 03_Architecture.md                  ← architektura systemu (Clean Architecture, moduły)
│   ├── 04_Functional_Requirements.md       ← wymagania funkcjonalne, przypadki użycia, GUI
│   ├── 05_NonFunctional_Requirements.md    ← wydajność, bezpieczeństwo, skalowalność, SLA
│   ├── 06_AI_Module.md                     ← rola i granice modułu AI
│   ├── 07_Data_Model.md                    ← model danych i słownik encji
│   ├── 08_API.md                           ← projekt API REST
│   ├── 09_Python_Architecture.md           ← architektura kodu Python, stack technologiczny
│   ├── 10_Deployment.md                    ← środowiska, CI/CD, infrastruktura, DR
│   ├── 11_Roadmap.md                       ← plan wdrożenia w fazach
│   └── 12_Risks.md                         ← rejestr ryzyk
│
├── uml/                                    ← diagramy PlantUML (źródła .puml)
│   ├── UseCase.puml
│   ├── ClassDiagram.puml
│   ├── ActivityDiagram.puml
│   ├── SequenceDiagram.puml
│   ├── ComponentDiagram.puml
│   ├── DeploymentDiagram.puml
│   └── ERD.puml
│
└── ARGUS-Documentation-Complete.pdf         ← skompilowana dokumentacja (wszystkie rozdziały + diagramy)
```

---

## 3. Jak korzystać z tego repozytorium

| Rola | Od czego zacząć |
|---|---|
| **Sponsor / Project Manager** | `01_Project_Overview.md` → `02_Business_Problem.md` → `11_Roadmap.md` → `12_Risks.md` |
| **Architekt IT / Security** | `00_Technical_Feasibility_Study.md` → `03_Architecture.md` → `05_NonFunctional_Requirements.md` → `10_Deployment.md` |
| **Zespół deweloperski** | `03_Architecture.md` → `07_Data_Model.md` → `08_API.md` → `09_Python_Architecture.md` → `uml/` |
| **Analityk biznesowy / właściciel raportów** | `04_Functional_Requirements.md` → `06_AI_Module.md` |

Diagramy w katalogu `uml/` są renderowalne dowolnym silnikiem zgodnym z PlantUML (rozszerzenie VS Code, `plantuml.jar`, CI). W dokumencie PDF diagramy są już wyrenderowane i opatrzone opisem oraz uzasadnieniem projektowym.

---

## 4. Kluczowe decyzje architektoniczne (skrót)

1. **Warstwa integracji jako zestaw wymiennych Connectorów** (`IReportConnector`) — dodanie nowej platformy raportowej (np. Tableau, Qlik) w przyszłości wymaga wyłącznie implementacji nowego connectora, bez zmian w rdzeniu systemu (Open/Closed Principle).
2. **Browser Automation (Playwright) jako uniwersalna warstwa integracyjna "najniższego wspólnego mianownika"**, uzupełniona o natywne API (Power BI REST API / XMLA, SSRS Web Services) tam, gdzie są dostępne i opłacalne — patrz uzasadnienie w `00_Technical_Feasibility_Study.md`.
3. **Value Extraction Engine oparty na strategii wielowarstwowego fallbacku** (DOM/HTML → API/Export → Computer Vision/OCR), aby system był odporny na drobne zmiany wizualne raportów.
4. **AI jako warstwa interpretacyjna, nigdy decyzyjna** — o poprawności raportu decyduje wyłącznie deterministyczny silnik walidacji.
5. **Clean Architecture + Dependency Injection** — separacja domeny od infrastruktury, testowalność, łatwość utrzymania w horyzoncie wieloletnim.

---

## 5. Stack technologiczny (skrót)

Pełne uzasadnienie każdego wyboru — patrz `09_Python_Architecture.md`. 
* Python 3.13+,
* FastAPI,
* Playwright,
* SQLAlchemy 2.x + Alembic,
* PostgreSQL,
* Redis,
* Celery + Redis/RabbitMQ (orkiestracja harmonogramu i workerów),
* Pydantic v2,
* LangChain/Semantic Kernel (wyłącznie moduł AI) + Azure OpenAI,
* React + TypeScript (GUI),
* Docker/Docker Compose + Kubernetes (docelowo),
* pytest,
* Ruff,
* MyPy,
* GitHub Actions,
* Prometheus,
* Grafana,
* OpenTelemetry. 

---

## 6. Status dokumentu

Niniejsza dokumentacja stanowi **Software Design Document** w rozumieniu procesu SDLC organizacji i jest podstawą do:
- oceny przez Architekturę IT i Bezpieczeństwo,
- estymacji przez zespół deweloperski,
- rozpoczęcia fazy Proof of Concept (patrz `11_Roadmap.md`, Faza 0).

Dokument nie zawiera kodu implementacyjnego — jest to dokumentacja projektowa poprzedzająca development.
