# 03. Architecture

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform \
**Dokument:** Architektura systemu \
**Odbiorcy:** Architektura IT, Zespół Deweloperski, Security Office

---

## 1. Zasady architektoniczne (Architectural Principles)

Architektura ARGUS jest projektowana w oparciu o pięć nadrzędnych zasad wskazanych w wymaganiach projektowych, konsekwentnie stosowanych na wszystkich poziomach systemu:

### 1.1 SOLID

- **S — Single Responsibility.** Każdy moduł (patrz sekcja 4) ma dokładnie jedną, jasno zdefiniowaną odpowiedzialność (np. `Scheduler` wyłącznie harmonogramuje, nie wykonuje testów; `Value Extraction Engine` wyłącznie odczytuje wartości, nie waliduje ich poprawności biznesowej).
- **O — Open/Closed.** System jest **otwarty na rozszerzenie, zamknięty na modyfikację** — kluczowy wymóg projektowy: dodanie nowej platformy raportowej (np. Tableau w przyszłości) wymaga wyłącznie dodania nowej klasy implementującej `IReportConnector`, bez zmian w silniku orkiestracji, walidacji czy GUI.
- **L — Liskov Substitution.** Każdy `IReportConnector` (Power BI, SSRS, przyszłe) jest w pełni wymienny z punktu widzenia `Test Orchestrator` — orkiestrator nie zawiera logiki `if platform == "PowerBI"`.
- **I — Interface Segregation.** Interfejsy są wąskie i zorientowane na konkretny kontrakt (`IReportConnector`, `IExtractionStrategy`, `ISqlValidator`, `INotificationChannel`) zamiast jednego wszechogarniającego interfejsu.
- **D — Dependency Inversion.** Warstwy wyższe (domena, logika aplikacyjna) zależą od **abstrakcji**, nie od konkretnych implementacji infrastrukturalnych (np. domena zależy od `IReportRepository`, a nie bezpośrednio od SQLAlchemy) — realizowane przez Dependency Injection (sekcja 3.4).

### 1.2 Clean Architecture

System jest podzielony na koncentryczne warstwy, gdzie zależności wskazują **zawsze do wewnątrz** (Dependency Rule): warstwa Infrastruktury zależy od Domeny, nigdy odwrotnie.

### 1.3 Separation of Concerns

Rozdzielenie odpowiedzialności realizowane jest na trzech poziomach jednocześnie:
- **Poziom modułów** — 17 modułów o rozłącznych odpowiedzialnościach (sekcja 4).
- **Poziom warstw** — Presentation / Application / Domain / Infrastructure (sekcja 2).
- **Poziom przekrojowy (cross-cutting)** — bezpieczeństwo, logowanie, audyt, konfiguracja jako aspekty niezależne od logiki biznesowej (sekcja 5).

### 1.4 Dependency Injection

Wszystkie zależności między komponentami są **wstrzykiwane**, a nie tworzone wewnątrz klas (`new`/bezpośrednia instancjacja). W warstwie API realizowane natywnym mechanizmem `Depends()` FastAPI; w warstwie domenowej i serwisowej — poprzez kontener DI (rekomendacja: `dependency-injector`, patrz `09_Python_Architecture.md`) konfigurowany w warstwie kompozycji (Composition Root), oddzielnie dla środowisk (dev/test/prod) i dla testów jednostkowych (mock/stub connectorów).

### 1.5 Open/Closed jako zasada nadrzędna dla rozszerzalności platformowej

To wymaganie było explicite podniesione w briefie projektowym i jest traktowane jako **twardy wymóg architektoniczny, a nie sugestia**: żaden nowy typ platformy raportowej nie powinien wymagać zmiany kodu poza katalogiem `connectors/`. Mechanizm rejestracji connectorów opiera się na wzorcu **Registry + Factory**, konfigurowanym deklaratywnie (typ platformy → implementacja connectora), tak by dodanie wpisu konfiguracyjnego i nowej klasy było wystarczające.

---

## 2. Warstwy architektury (Clean Architecture)

```
┌───────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER                                            │
│  Dashboard (Web GUI), REST API (FastAPI), Notification Output  │
├───────────────────────────────────────────────────────────────┤
│  APPLICATION LAYER (Use Cases / Orchestration)                 │
│  Test Orchestrator, Scheduler, AI Analysis Engine (orkiestracja)│
├───────────────────────────────────────────────────────────────┤
│  DOMAIN LAYER (rdzeń biznesowy, niezależny od frameworków)      │
│  Validation Engine, Business Rule Engine, Health/Confidence     │
│  Score, encje domenowe: Report, ValidationProfile, TestResult   │
├───────────────────────────────────────────────────────────────┤
│  INFRASTRUCTURE LAYER (adaptery do świata zewnętrznego)         │
│  Report Platform Connectors (PBI/SSRS), Browser Automation      │
│  Engine, Value Extraction Engine, SQL Validation Engine,        │
│  Results/Configuration/Validation Profile Repository (Postgres),│
│  Notification Service, Audit & Logging, Monitoring              │
└───────────────────────────────────────────────────────────────┘
```

**Reguła zależności:** warstwa Domain nie importuje niczego z Infrastructure ani Presentation. Warstwa Application zależy od abstrakcji zdefiniowanych w Domain (porty), a konkretne implementacje (adaptery) dostarczane są przez Infrastructure i wstrzykiwane w warstwie kompozycji (`main.py` / bootstrap aplikacji). Jest to wariant architektury **Ports & Adapters (Hexagonal)** zagnieżdżony w Clean Architecture — Domain i Application definiują porty (interfejsy), Infrastructure implementuje adaptery.

---

## 3. Kluczowe wzorce projektowe

### 3.1 Connector Pattern (Report Platform Connector Layer)

```python
# Ilustracja koncepcyjna interfejsu — nie jest to kod produkcyjny do implementacji wprost
class IReportConnector(Protocol):
    async def check_availability(self, report_ref: ReportReference) -> AvailabilityResult: ...
    async def open_report(self, report_ref: ReportReference) -> RenderedReportHandle: ...
    async def list_catalog(self) -> list[ReportMetadata]: ...
```

Konkretne implementacje: `PowerBIConnector`, `SSRSConnector` — obie implementują ten sam kontrakt, ale wewnętrznie korzystają z różnych mechanizmów niskopoziomowych opisanych w `00_Technical_Feasibility_Study.md` (Browser Automation jako wspólna baza + odpowiednie API natywne jako wzbogacenie). Wybór connectora następuje na podstawie pola `platform_type` w `Validation Profile`, poprzez `ConnectorFactory`, bez logiki warunkowej rozproszonej po systemie.

### 3.2 Strategy + Chain of Responsibility (Value Extraction Engine)

Opisany szczegółowo w `00_Technical_Feasibility_Study.md`, sekcja 7. Architektonicznie: `IExtractionStrategy` jako wspólny interfejs, `ExtractionChain` jako obiekt porządkujący sekwencję strategii per punkt ekstrakcji zdefiniowany w Validation Profile, z jednolitym obiektem wyniku `ExtractedValue(value, confidence_score, method_used, raw_evidence)`.

### 3.3 Rule Engine Pattern (Business Rule Engine)

Reguły biznesowe (np. "suma kategorii = wartość całkowita ± tolerancja", "wartość nie może być ujemna", "wartość bieżąca odchyla się od średniej historycznej o więcej niż X%") są modelowane jako **deklaratywne, konfigurowalne obiekty reguł** (`IBusinessRule.evaluate(context) -> RuleResult`), a nie zaszyty w kodzie warunek — pozwala to administratorowi definiować nowe reguły bez zmiany kodu aplikacji, poprzez konfigurację lub bibliotekę predefiniowanych typów reguł rozszerzaną w cyklu deweloperskim.

### 3.4 Dependency Injection / Composition Root

Cała aplikacja (worker wykonawczy, API, scheduler) posiada jeden punkt kompozycji zależności (Composition Root), w którym konkretne implementacje (repozytoria SQLAlchemy, connectory, kanały powiadomień) są wiązane z interfejsami domenowymi. Ułatwia to:
- podmianę implementacji w testach (np. `FakeReportConnector` zamiast prawdziwego Playwright w testach jednostkowych Test Orchestratora),
- niezależne skalowanie i wdrażanie komponentów (worker od API),
- czytelność zależności dla nowego dewelopera dołączającego do zespołu.

### 3.5 Repository Pattern

Dostęp do danych trwałych (raporty, profile, wyniki testów) odbywa się wyłącznie przez interfejsy repozytoriów zdefiniowane w warstwie Domain (`IReportRepository`, `IValidationProfileRepository`, `IResultsRepository`), implementowane w Infrastructure przy użyciu SQLAlchemy. Domena nigdy nie zna szczegółów ORM.

### 3.6 Odporność i resilience (Circuit Breaker, Retry, Timeout)

Ze względu na zależność od zewnętrznych, czasem niestabilnych platform (Power BI, SSRS), warstwa Infrastructure stosuje wzorce odpornościowe: **Retry z backoff wykładniczym** dla przejściowych błędów sieciowych, **Circuit Breaker** wyłączający tymczasowo testy dla platformy zgłaszającej masowe awarie (aby nie "zalewać" niedziałającego serwera i nie generować fałszywych alarmów o pojedynczych raportach, gdy przyczyna leży w całej platformie), oraz jawne **Timeouty** na każdym poziomie (nawigacja, ekstrakcja, zapytanie SQL).

---

## 4. Główne moduły systemu i ich odpowiedzialności

| # | Moduł | Warstwa | Odpowiedzialność |
|---|---|---|---|
| 1 | **Scheduler** | Application | Harmonogramowanie wykonania testów zgodnie z częstotliwością zdefiniowaną per Validation Profile (cron-like), inicjowanie przebiegów wsadowych (batch) oraz uruchomień na żądanie (manual trigger z GUI). Nie wykonuje testów samodzielnie — deleguje do Test Orchestratora. |
| 2 | **Test Orchestrator** | Application | Koordynuje pełny cykl życia pojedynczego przebiegu testowego: pobranie Validation Profile → wybór connectora → otwarcie raportu → ekstrakcja wartości → walidacja → zapis wyniku → wyzwolenie analizy AI i powiadomień. Odpowiada za równoległość i kolejkowanie przebiegów (worker pool). |
| 3 | **Configuration Repository** | Infrastructure | Przechowuje konfigurację systemową i katalog raportów (metadane: adres, obszar biznesowy/zakładka, typ platformy, właściciel). Źródło prawdy dla inwentaryzacji raportów. |
| 4 | **Validation Profile Repository** | Infrastructure | Przechowuje i wersjonuje Validation Profiles — definicje testów, KPI, zapytań SQL, tolerancji, reguł biznesowych, harmonogramów, wraz z pełną historią zmian (audytowalność). |
| 5 | **Test Definition Engine** | Domain/Application | Udostępnia model i walidację poprawności definicji testu podczas jego tworzenia/edycji przez administratora (np. sprawdzenie spójności zapytania SQL, poprawności selektora KPI) — zapobiega zapisaniu niekompletnego/błędnego profilu. |
| 6 | **Browser Automation Engine** | Infrastructure | Cienka, reużywalna warstwa zarządzająca cyklem życia instancji przeglądarki (Playwright): pule kontekstów, zarządzanie sesją SSO, nawigacja, oczekiwanie na stabilizację renderowania, przechwytywanie zrzutów ekranu i logów sieciowych jako dowodów. Wykorzystywana przez connectory, nieeksponowana bezpośrednio na zewnątrz. |
| 7 | **Report Platform Connector Layer** | Infrastructure | Zestaw connectorów (`PowerBIConnector`, `SSRSConnector`, ...) implementujących `IReportConnector` — jedyne miejsce w systemie znające specyfikę danej platformy raportowej. |
| 8 | **Value Extraction Engine** | Infrastructure/Domain | Odczyt wartości KPI z otwartego raportu przy użyciu strategii wielowarstwowego fallbacku (patrz `00_Technical_Feasibility_Study.md`). Zwraca wartość wraz z Confidence Score i użytą metodą. |
| 9 | **Validation Engine** | Domain | Rdzeń deterministycznej oceny poprawności: porównuje wartości odczytane z raportu z wynikami SQL Validation Engine w granicach dopuszczalnych odchyleń zdefiniowanych w profilu; agreguje wyniki pojedynczych testów w wynik przebiegu. To jedyny moduł uprawniony do wydania werdyktu PASS/FAIL/WARNING. |
| 10 | **SQL Validation Engine** | Infrastructure | Bezpieczne (parametryzowane, z kontrolą uprawnień konta technicznego) wykonywanie zapytań SQL zdefiniowanych w Validation Profile na hurtowni danych i zwracanie wyników do Validation Engine. |
| 11 | **Business Rule Engine** | Domain | Wykonuje reguły biznesowe wykraczające poza proste porównanie dwóch liczb (sekcja 3.3). |
| 12 | **Results Repository** | Infrastructure | Trwały, zoptymalizowany pod odczyt historyczny magazyn wyników testów, Health Score i Confidence Score w czasie (patrz `07_Data_Model.md` — strategia retencji/partycjonowania). |
| 13 | **AI Analysis Engine** | Application (usługa wspierająca) | Interpretuje wyniki dostarczone przez Validation Engine: wykrywa anomalie na tle historii, grupuje podobne błędy, wskazuje najbardziej prawdopodobne przyczyny, generuje czytelne uzasadnienia i sugestie działań naprawczych. Nigdy nie zmienia ani nie nadpisuje werdyktu Validation Engine (patrz `06_AI_Module.md`). |
| 14 | **Dashboard** | Presentation | Aplikacja webowa GUI — przegląd raportów, konfiguracja, historia, wyniki AI, Health/Confidence Score (patrz `04_Functional_Requirements.md`). |
| 15 | **Notification Service** | Infrastructure | Dystrybucja powiadomień o wykrytych problemach do zdefiniowanych odbiorców/kanałów (e-mail, integracja z systemem powiadomień organizacji), z regułami deduplikacji i eskalacji. |
| 16 | **Audit & Logging** | Cross-cutting/Infrastructure | Rejestruje wszystkie zdarzenia istotne z punktu widzenia audytu: zmiany konfiguracji/profili, wykonania testów, decyzje systemu, działania użytkowników GUI/API. Niezależny, tylko-do-zapisu (append-only) log audytowy odrębny od logów technicznych. |
| 17 | **Monitoring** | Cross-cutting/Infrastructure | Metryki operacyjne systemu (czas wykonania testów, obciążenie worker pool, wskaźniki błędów connectorów, dostępność platform BI) eksponowane do Prometheus/Grafana; integralna część NFR (patrz `05_NonFunctional_Requirements.md`). |

### 4.1 Diagram zależności modułów (skrót koncepcyjny)

Pełny, formalny diagram komponentów z relacjami — patrz `uml/ComponentDiagram.puml`. Poniżej skrót tekstowy głównego przepływu zależności:

```
Scheduler ──▶ Test Orchestrator ──▶ Report Platform Connector Layer ──▶ Browser Automation Engine
                     │                         │
                     ▼                         ▼
           Value Extraction Engine ◀── (otwarty raport / dane platformy)
                     │
                     ▼
             Validation Engine ──▶ SQL Validation Engine (hurtownia danych)
                     │        └──▶ Business Rule Engine
                     ▼
             Results Repository ──▶ AI Analysis Engine ──▶ Notification Service
                     │                                            │
                     ▼                                            ▼
                Dashboard  ◀─────────────────────────────  (powiadomienia)

Wszystkie moduły ──▶ Audit & Logging, Monitoring (cross-cutting)
Configuration / Validation Profile Repository ◀──▶ Test Definition Engine, Test Orchestrator, Dashboard
```

---

## 5. Zagadnienia przekrojowe (Cross-Cutting Concerns)

### 5.1 Bezpieczeństwo

- Konto techniczne o **najmniejszym niezbędnym zakresie uprawnień** (least privilege) do Platformy BI i hurtowni danych, zarządzane zgodnie z polityką zarządzania sekretami organizacji (patrz `10_Deployment.md`).
- Autoryzacja i autentykacja użytkowników GUI/API oparta o istniejący mechanizm SSO organizacji (Windows Authentication / Azure AD / AD FS — do potwierdzenia z zespołem IAM), z modelem RBAC (role: Administrator, Analityk/Właściciel testu, Przeglądający).
- Pełny szczegół — patrz `05_NonFunctional_Requirements.md`, sekcja Bezpieczeństwo.

### 5.2 Konfiguracja

Konfiguracja aplikacji zarządzana zgodnie z zasadą **12-Factor App** — parametry środowiskowe (adresy platform, connection stringi, sekrety) wstrzykiwane przez zmienne środowiskowe / zarządzany magazyn sekretów, nigdy hardkodowane; walidowane przy starcie aplikacji (Pydantic Settings — patrz `09_Python_Architecture.md`).

### 5.3 Logowanie i audyt

Rozdzielone strumienie: **logi techniczne** (poziom DEBUG/INFO/WARNING/ERROR, do celów operacyjnych i debugowania, skorelowane przez `trace_id` zgodnie z OpenTelemetry) oraz **log audytowy** (append-only, dotyczący zdarzeń biznesowych: kto zmienił Validation Profile, kto uruchomił test ręcznie, jaki był werdykt) — ten drugi podlega odrębnym wymogom retencji zgodnym z polityką organizacji.

### 5.4 Obserwowalność (Observability)

Trzy filary: **metryki** (Prometheus/Grafana), **logi** (ustrukturyzowane, JSON, agregowane centralnie), **trace'y rozproszone** (OpenTelemetry) — pozwalające prześledzić pojedynczy przebieg testowy przez wszystkie moduły, co jest kluczowe przy debugowaniu problemów wydajnościowych na skalę 1000 raportów.

---

## 6. Rozszerzalność — scenariusz "nowa platforma raportowa"

Aby zilustrować zgodność z zasadą Open/Closed, poniżej przedstawiono pełną listę zmian wymaganych do dodania obsługi nowej, hipotetycznej platformy raportowej (np. Tableau) w przyszłości:

1. Implementacja nowej klasy `TableauConnector` zgodnej z `IReportConnector`.
2. Implementacja (opcjonalnie, jeśli potrzebna specyficzna strategia) nowej klasy `IExtractionStrategy` dla specyfiki renderowania Tableau.
3. Rejestracja nowego typu platformy w `ConnectorFactory` (jedna linia konfiguracji/rejestracji).
4. Dodanie wartości `TABLEAU` do enumeracji typu platformy w modelu domenowym (`platform_type`).

**Żadna** z powyższych zmian nie ingeruje w Test Orchestrator, Validation Engine, Business Rule Engine, Results Repository, Dashboard ani AI Analysis Engine — potwierdzając, że koszt krańcowy dodania nowej platformy jest liniowo mały i izolowany.

---

## 7. Powiązanie z dalszą dokumentacją

- Szczegółowe uzasadnienie wyboru technologii integracji — `00_Technical_Feasibility_Study.md`.
- Model danych wspierający powyższą architekturę — `07_Data_Model.md`.
- Kontrakty API pomiędzy warstwą Presentation a Application — `08_API.md`.
- Konkretna implementacja w Pythonie (struktura katalogów, biblioteki) — `09_Python_Architecture.md`.
- Diagram komponentów i diagram wdrożenia — `uml/ComponentDiagram.puml`, `uml/DeploymentDiagram.puml`.
