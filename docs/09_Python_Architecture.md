# 09. Python Architecture

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform \
**Dokument:** Architektura kodu Python i stack technologiczny \
**Odbiorcy:** Zespół Deweloperski, Architektura IT

---

## 1. Stack technologiczny — pełna lista i uzasadnienie

| Warstwa | Technologia | Uzasadnienie |
|---|---|---|
| Język | **Python 3.13+** | Dojrzały ekosystem bibliotek do automatyzacji przeglądarki, przetwarzania danych i integracji AI; natywne wsparcie dla `asyncio` kluczowe dla równoległości Browser Automation na skalę 1000 raportów; nowoczesne typowanie (generics, `TypedDict`, `Protocol`) wspierające zasady SOLID i statyczną analizę. |
| Framework API | **FastAPI** | Natywna integracja z Pydantic (walidacja danych, automatyczna dokumentacja OpenAPI), natywny `async/await`, wbudowany mechanizm Dependency Injection (`Depends`) spójny z zasadą DI z `03_Architecture.md`, bardzo dobra wydajność względem alternatyw synchronicznych (Flask/Django REST). |
| Automatyzacja przeglądarki | **Playwright (Python, async API)** | Uzasadnienie wyboru względem Selenium — patrz `00_Technical_Feasibility_Study.md`, sekcja 3.1–3.2. Natywne wsparcie async, auto-waiting, przechwytywanie sieci, stabilna obsługa wielu kontekstów przeglądarki na proces. |
| ORM / dostęp do danych | **SQLAlchemy 2.x (styl `async`, `Mapped`/`mapped_column`)** | Standard de facto w ekosystemie Python enterprise, pełne wsparcie typowania statycznego (MyPy), wzorzec Unit of Work/Repository łatwy do zaimplementowania nad SQLAlchemy Session, wsparcie dla trybu asynchronicznego zgodnego z resztą stacku. |
| Migracje schematu | **Alembic** | Standardowy, w pełni zintegrowany z SQLAlchemy mechanizm wersjonowania schematu bazy — kluczowy dla audytowalnych, powtarzalnych wdrożeń w środowisku bankowym (Dev→Test→UAT→Prod). |
| Baza danych | **PostgreSQL** | Dojrzałość, natywne partycjonowanie deklaratywne (istotne przy skali historii wyników — patrz `07_Data_Model.md`), wsparcie JSONB (elastyczne przechowywanie `extraction_config`, `evidence` AI), silne wsparcie transakcyjne, brak kosztów licencyjnych, powszechna akceptacja w środowiskach bankowych jako alternatywa dla komercyjnych silników. |
| Cache / broker kolejki | **Redis** | Niskie opóźnienia, wsparcie jako broker dla Celery, przechowywanie stanu sesji/rate-limiting, cache wyników zapytań agregujących do Dashboardu (NFR-P-05). |
| Orkiestracja zadań asynchronicznych | **Celery (z Redis lub RabbitMQ jako broker)** | Rekomendacja wyjściowa: **Celery + Redis** dla MVP (prostota operacyjna), z możliwością migracji na **RabbitMQ** jako broker w fazie skalowania produkcyjnego (Faza 4, patrz `11_Roadmap.md`), jeśli wymagana będzie wyższa gwarancja dostarczenia i bardziej zaawansowane routing zadań. Celery wybrany zamiast samego APScheduler, ponieważ APScheduler dobrze sprawdza się jako **harmonogram** (triggering), ale nie jako **rozproszony worker pool** wykonawczy — stąd rekomendacja hybrydowa: **APScheduler wewnątrz komponentu `Scheduler`** (odpowiedzialność: kiedy uruchomić) + **Celery jako warstwa wykonawcza** (`Test Orchestrator` → zadania Celery na worker pool, odpowiedzialność: jak i gdzie wykonać, z retry/backoff/circuit breaker). |
| Walidacja danych / konfiguracji | **Pydantic v2** | Wysoka wydajność (rdzeń w Rust), natywna integracja z FastAPI, `pydantic-settings` do zarządzania konfiguracją zgodnie z 12-Factor App (patrz `03_Architecture.md`, sekcja 5.2). |
| Framework AI | **LangChain** (moduł AI **wyłącznie**) | Zgodnie z wymogiem projektowym — używany punktowo do orkiestracji łańcucha: budowa promptu → wywołanie modelu → walidacja odpowiedzi strukturalnej (schema JSON) → zapis. Alternatywa rozważana: lekki własny wrapper nad klientem Azure OpenAI, jeśli złożoność łańcuchów w Fazie 3 okaże się niewystarczająco duża, by uzasadnić zależność od LangChain — decyzja do potwierdzenia w Fazie 3 na bazie rzeczywistych potrzeb (unikanie nadmiarowej złożoności — zasada YAGNI). |
| Dostawca modelu AI | **Azure OpenAI** (rekomendacja) | Patrz uzasadnienie w `06_AI_Module.md`, sekcja 5.3 oraz `05_NonFunctional_Requirements.md`, NFR-SEC-07. |
| Automatyzacja przeglądarki — CV/OCR (fallback) | **OpenCV** (Computer Vision) + **Tesseract / pytesseract** (OCR) | Standardowe, sprawdzone biblioteki open-source do warstwy fallbacku Value Extraction Engine (patrz `00_Technical_Feasibility_Study.md`, sekcja 6.6–6.7); brak kosztów licencyjnych, możliwość pracy w pełni on-prem. |
| Integracja XMLA (opcjonalna) | **pythonnet + ADOMD.NET** lub dedykowany mikroserwis .NET | Patrz `00_Technical_Feasibility_Study.md`, sekcja 3.4 — most technologiczny do XMLA Endpoint, izolowany jako opcjonalny connector, nieobowiązkowy dla MVP. |
| Frontend / GUI | **React + TypeScript** | Dojrzały ekosystem komponentowy, dobre wsparcie dla wykresów/trendów (biblioteki typu Recharts/Chart.js), łatwość integracji z zespołami frontendowymi banku posiadającymi doświadczenie w React; TypeScript zapewnia bezpieczeństwo typów spójne z podejściem statycznego typowania po stronie backendu. |
| Konteneryzacja | **Docker / Docker Compose (Dev/Test), Kubernetes (docelowo Prod)** | Docker Compose wystarczający i szybki do uruchomienia dla środowisk deweloperskich/testowych; Kubernetes rekomendowany dla produkcji ze względu na potrzebę horyzontalnego skalowania worker poola Playwright (patrz `10_Deployment.md`). |
| Testy | **pytest** (+ `pytest-asyncio`, `pytest-cov`, `playwright` fixtures) | Standard w ekosystemie Python, bogaty ekosystem pluginów, czytelna składnia parametryzacji testów kontraktowych connectorów (NFR-M-03). |
| Statyczna analiza | **Ruff** (linting + formatting) **+ MyPy (strict mode)** | Ruff zastępuje jednocześnie flake8/isort/black przy znacząco wyższej wydajności (implementacja w Rust) — szybki feedback w CI; MyPy strict wymusza pełne typowanie zgodne z zasadami SOLID/Clean Architecture (interfejsy jako `Protocol`/ABC). |
| CI/CD | **GitHub Actions** | Zgodnie z rekomendacją projektową; pipeline: lint → type-check → testy jednostkowe/kontraktowe → build obrazu Docker → skanowanie podatności (SCA/SAST) → publikacja do wewnętrznego registry. |
| Obserwowalność | **Prometheus + Grafana + OpenTelemetry** | Zgodnie z `05_NonFunctional_Requirements.md`, sekcja Observability — standard branżowy, dobra integracja z FastAPI (`prometheus-fastapi-instrumentator`) i Celery (`celery-prometheus-exporter` lub instrumentacja własna). |
| Zarządzanie zależnościami | **uv** (rekomendacja) lub **Poetry** | `uv` rekomendowany ze względu na istotnie wyższą wydajność instalacji/resolvingu zależności (implementacja w Rust) oraz rosnącą dojrzałość w ekosystemie 2025/2026; Poetry jako sprawdzona alternatywa, jeśli standardy wewnętrzne organizacji tego wymagają — decyzja do ostatecznego zatwierdzenia z zespołem platformowym w Fazie 0. |
| Kontener DI | **`dependency-injector`** | Jawny, czytelny Composition Root (patrz `03_Architecture.md`, sekcja 3.4), wsparcie dla konfiguracji kontenera per środowisko (dev/test/prod) oraz łatwe podmienianie implementacji w testach (Fake/Mock connectory). |

---

## 2. Struktura katalogów (Clean Architecture / Ports & Adapters)

```
argus/
├── src/
│   └── argus/
│       ├── domain/                      # DOMAIN LAYER — zero zależności od frameworków
│       │   ├── entities/                # Report, ValidationProfile, TestDefinition, TestResult, ...
│       │   ├── value_objects/           # ExtractedValue, HealthScore, ConfidenceScore, Tolerance
│       │   ├── services/                # ValidationEngine, BusinessRuleEngine (logika czysta)
│       │   └── ports/                   # Interfejsy (Protocol): IReportRepository,
│       │                                 # IValidationProfileRepository, IResultsRepository,
│       │                                 # IReportConnector, IExtractionStrategy, ISqlValidator,
│       │                                 # INotificationChannel, IAIProvider
│       │
│       ├── application/                 # APPLICATION LAYER — przypadki użycia / orkiestracja
│       │   ├── use_cases/               # np. RunTestExecutionUseCase, CreateValidationProfileUseCase
│       │   ├── orchestration/           # TestOrchestrator, Scheduler (logika harmonogramowania)
│       │   └── ai/                      # AIAnalysisEngine (orkiestracja wywołań, non-blocking)
│       │
│       ├── infrastructure/              # INFRASTRUCTURE LAYER — adaptery
│       │   ├── connectors/
│       │   │   ├── power_bi/            # PowerBIConnector + strategie ekstrakcji specyficzne
│       │   │   ├── ssrs/                # SSRSConnector + strategie ekstrakcji specyficzne
│       │   │   └── factory.py           # ConnectorFactory (Registry Pattern)
│       │   ├── browser_automation/      # Cienka warstwa nad Playwright (pule kontekstów, SSO)
│       │   ├── extraction/              # Implementacje IExtractionStrategy (DOM, CV, OCR, Export, API)
│       │   ├── sql_validation/          # SQLValidationEngine (bezpieczne wykonanie zapytań)
│       │   ├── persistence/
│       │   │   ├── models/              # Modele SQLAlchemy (mapowanie 1:1 z domain/entities)
│       │   │   ├── repositories/        # Implementacje portów repozytoriów
│       │   │   └── migrations/          # Alembic
│       │   ├── notifications/           # Implementacje INotificationChannel (Email, ...)
│       │   ├── ai_providers/            # Implementacja IAIProvider (Azure OpenAI + LangChain)
│       │   ├── audit/                   # Audit & Logging (append-only writer)
│       │   └── monitoring/              # Instrumentacja Prometheus/OpenTelemetry
│       │
│       ├── presentation/                # PRESENTATION LAYER
│       │   ├── api/
│       │   │   ├── routers/             # reports.py, validation_profiles.py, test_executions.py, ...
│       │   │   ├── schemas/             # Pydantic request/response models (DTO, odseparowane od domain)
│       │   │   └── dependencies.py      # Depends() — auth, RBAC, DI wiring per request
│       │   └── workers/                 # Celery task definitions (cienkie, delegujące do use_cases)
│       │
│       ├── config/                      # Pydantic Settings, konfiguracja per środowisko
│       └── composition_root.py          # Wiring kontenera DI — jedyne miejsce łączące porty z adapterami
│
├── frontend/                            # Aplikacja React + TypeScript (GUI)
│
├── tests/
│   ├── unit/                            # Testy domain/ i application/ (z Fake/Mock adapterami)
│   ├── contract/                        # Testy kontraktowe każdego IReportConnector (NFR-M-03)
│   ├── integration/                     # Testy z realną bazą (testcontainers) i mockowaną platformą BI
│   └── e2e/                             # Testy end-to-end na środowisku Test/UAT (ograniczony zakres)
│
├── docker/
│   ├── Dockerfile.api
│   ├── Dockerfile.worker
│   └── docker-compose.yml
│
├── .github/workflows/                   # Pipeline CI/CD (GitHub Actions)
├── pyproject.toml
└── alembic.ini
```

**Uzasadnienie:** struktura fizycznych katalogów odwzorowuje 1:1 warstwy logiczne z `03_Architecture.md`, sekcja 2 — deweloper poruszający się po repozytorium natychmiast widzi, do której warstwy Clean Architecture należy dany plik, co ułatwia utrzymanie reguły zależności (import-linter / narzędzie do wymuszania granic warstw rekomendowane jako część CI — patrz sekcja 5).

---

## 3. Dependency Injection — Composition Root

```python
# composition_root.py — ilustracja koncepcyjna, nie kod produkcyjny do wdrożenia wprost
from dependency_injector import containers, providers

class ApplicationContainer(containers.DeclarativeContainer):
    config = providers.Configuration()

    # Infrastructure — repozytoria
    report_repository = providers.Singleton(SqlAlchemyReportRepository, session_factory=...)
    validation_profile_repository = providers.Singleton(SqlAlchemyValidationProfileRepository, ...)

    # Infrastructure — connectory (Registry przez Factory)
    connector_factory = providers.Singleton(
        ConnectorFactory,
        connectors={
            "POWER_BI": providers.Factory(PowerBIConnector, browser_engine=...),
            "SSRS": providers.Factory(SSRSConnector, browser_engine=...),
        },
    )

    # Application — use case'y komponowane z portów
    run_test_execution_use_case = providers.Factory(
        RunTestExecutionUseCase,
        connector_factory=connector_factory,
        extraction_chain=...,
        validation_engine=...,
        results_repository=...,
        ai_engine=...,
        notification_service=...,
    )
```

W testach jednostkowych ten sam `ApplicationContainer` jest konfigurowany z implementacjami `Fake`/`Mock` (np. `FakeReportConnector` zwracający zdefiniowane w teście dane), co pozwala testować `RunTestExecutionUseCase` w pełnej izolacji od Playwright, sieci czy bazy danych — kluczowe dla realizacji NFR-M-02 (pokrycie testami warstwy domenowej ≥ 85%).

---

## 4. Model wykonawczy i współbieżność

### 4.1 Dwuwarstwowy model harmonogramu i wykonania

```
APScheduler (Scheduler)                Celery (worker pool, N replik)
   │  odpytuje ValidationProfile           │
   │  wg schedule_cron                     │
   ▼                                       ▼
"czas uruchomić raport X"  ──enqueue──▶  task: run_test_execution(report_id)
                                            │
                                            ▼
                                  RunTestExecutionUseCase
                                  (Playwright async, per-task izolowany
                                   kontekst przeglądarki, timeout, retry)
```

- **APScheduler** odpowiada wyłącznie za "kiedy" (trigger), publikując zadania do kolejki Celery — rozdzielenie odpowiedzialności zgodne z SRP.
- **Celery workery** skalowane horyzontalnie (liczba replik konfigurowalna, patrz `10_Deployment.md`), każdy worker zarządza własną pulą kontekstów przeglądarki Playwright (nie całych instancji przeglądarki — reużycie procesu przeglądarki między zadaniami dla wydajności, przy izolacji sesji przez oddzielne konteksty).
- Limit współbieżności per worker oraz globalny limit równoległych połączeń do hurtowni danych są konfigurowalne (`config/`), zgodnie z NFR-P-04 (ochrona hurtowni przed przeciążeniem).

### 4.2 Asynchroniczność w kodzie

Cały łańcuch krytyczny dla wydajności (`browser_automation`, `extraction`, wywołania HTTP do API platform, klient AI) implementowany w `asyncio` (natywne wsparcie Playwright async API i `httpx.AsyncClient`), z jawnymi timeoutami na każdym poziomie (`asyncio.wait_for`) zgodnie z wymogiem odpornościowym z `03_Architecture.md`, sekcja 3.6.

---

## 5. Wymuszanie granic architektonicznych w CI

Aby zapobiec erozji architektury w czasie (częsty problem długoletnich projektów enterprise), pipeline CI zawiera dedykowany krok **import-linter** (lub równoważne narzędzie), wymuszający reguły:

- `domain/` nie importuje niczego z `infrastructure/` ani `presentation/`,
- `application/` nie importuje bezpośrednio z `infrastructure/` (wyłącznie przez porty zdefiniowane w `domain/ports/`),
- `presentation/` nie zawiera logiki domenowej (wyłącznie mapowanie DTO ↔ wywołanie use case).

Naruszenie którejkolwiek reguły blokuje merge — architektura jest w ten sposób **wymuszana narzędziowo, nie tylko opisana w dokumentacji**.

---

## 6. Zarządzanie kosztem wywołań AI

Zgodnie z `06_AI_Module.md`, sekcja 7 (ryzyko kosztowe), warstwa `application/ai/` implementuje jawną politykę selektywnego wywoływania modelu — `AIAnalysisEngine` jest wyzwalany przez `TestOrchestrator` **wyłącznie** dla przebiegów o statusie innym niż pełny `PASS` lub oznaczonych przez `Anomaly Detector` jako odbiegające od trendu, minimalizując liczbę (a więc i koszt) wywołań LLM przy pełnej skali 1000 raportów.

---

## 7. Powiązanie z dalszą dokumentacją

- Warstwy i wzorce architektoniczne odwzorowane w powyższej strukturze — `03_Architecture.md`.
- Encje domenowe mapowane do `domain/entities/` i `infrastructure/persistence/models/` — `07_Data_Model.md`.
- Endpointy `presentation/api/routers/` — `08_API.md`.
- Środowiska uruchomieniowe, obrazy Docker, pipeline CI/CD — `10_Deployment.md`.
