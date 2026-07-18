# 07. Data Model

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform
**Dokument:** Model danych i słownik encji
**Odbiorcy:** Zespół Deweloperski, DBA, Architektura Danych

---

## 1. Cel dokumentu

Dokument opisuje logiczny model danych systemu ARGUS na poziomie encji biznesowych i ich relacji. Pełny diagram graficzny (encje, atrybuty, kardynalności) znajduje się w `uml/ERD.puml`. Fizyczny model bazy danych (typy kolumn, indeksy, migracje) jest przedmiotem implementacji w Alembic, na bazie niniejszego modelu logicznego — patrz `09_Python_Architecture.md`.

Silnik bazy danych: **PostgreSQL** (uzasadnienie wyboru — patrz `09_Python_Architecture.md`, sekcja Stack Technologiczny).

---

## 2. Słownik encji

### 2.1 `BusinessArea` (Obszar biznesowy / zakładka)

Odwzorowuje grupowanie raportów zgodnie ze strukturą istniejącej Platformy BI (zakładki: Sprzedaż, Kredyty, Ryzyko itd.).

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `name` | Nazwa obszaru biznesowego |
| `description` | Opis |
| `owner` | Właściciel biznesowy obszaru (opcjonalnie) |

### 2.2 `Report` (Raport)

Reprezentuje pojedynczy raport z katalogu Platformy BI.

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `name` | Nazwa raportu |
| `platform_type` | `POWER_BI` \| `SSRS` (rozszerzalne — patrz `03_Architecture.md`, sekcja 6) |
| `url` | Adres raportu na Platformie BI |
| `business_area_id` | FK → `BusinessArea` |
| `owner` | Właściciel biznesowy raportu (analityk odpowiedzialny) |
| `is_active` | Czy raport jest aktywnie monitorowany |
| `created_at`, `updated_at` | Znaczniki czasu |

### 2.3 `ValidationProfile`

Konfiguracja testowa przypisana do raportu — **wersjonowana** (każda zmiana tworzy nową wersję, poprzednie są zachowywane dla audytu).

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `report_id` | FK → `Report` |
| `version` | Numer wersji (rosnący) |
| `is_active` | Czy jest to aktualnie obowiązująca wersja |
| `schedule_cron` | Wyrażenie harmonogramu (częstotliwość wykonania) |
| `created_by` | FK → `User` |
| `created_at` | Znacznik czasu utworzenia wersji |
| `change_reason` | Opcjonalny opis powodu zmiany (dla audytu) |

### 2.4 `TestDefinition`

Pojedynczy test wchodzący w skład Validation Profile.

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `validation_profile_id` | FK → `ValidationProfile` |
| `name` | Nazwa testu (np. "KPI_SPRZEDAZ_TOTAL") |
| `type` | `SQL_COMPARISON` \| `FIELD_VALUE` \| `BUSINESS_RULE` \| `AVAILABILITY` |
| `extraction_config` | Konfiguracja odnajdywania wartości w raporcie (selektor semantyczny, strategia fallback — patrz `00_Technical_Feasibility_Study.md`) |
| `tolerance_abs` | Dopuszczalne odchylenie bezwzględne |
| `tolerance_pct` | Dopuszczalne odchylenie procentowe |
| `is_active` | Czy test jest aktywny |

### 2.5 `SqlValidationQuery`

Zapytanie SQL powiązane z testem typu `SQL_COMPARISON`.

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `test_definition_id` | FK → `TestDefinition` |
| `sql_text` | Treść zapytania (parametryzowana) |
| `data_source_ref` | Referencja do zdefiniowanego, autoryzowanego źródła danych (hurtowni) — nigdy dowolny connection string wpisywany ad-hoc |
| `expected_result_mapping` | Mapowanie wyniku zapytania na wartość porównywaną |

### 2.6 `BusinessRule`

Reguła biznesowa powiązana z testem typu `BUSINESS_RULE` (patrz `03_Architecture.md`, sekcja 3.3).

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `test_definition_id` | FK → `TestDefinition` |
| `rule_type` | Typ reguły (np. `SUM_CHECK`, `RANGE_CHECK`, `TREND_DEVIATION`, `NON_NEGATIVE`) |
| `configuration` | Parametry reguły (strukturalne, zależne od typu) |

### 2.7 `TestExecution` (Przebieg testowy)

Pojedyncze wykonanie zestawu testów dla raportu (całego Validation Profile) w danym momencie.

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `report_id` | FK → `Report` |
| `validation_profile_id` | FK → `ValidationProfile` (konkretna wersja użyta w tym przebiegu — kluczowe dla audytu) |
| `triggered_by` | `SCHEDULER` \| `MANUAL:{user_id}` |
| `started_at`, `finished_at` | Znaczniki czasu |
| `overall_status` | `PASS` \| `FAIL` \| `WARNING` \| `ERROR` (zagregowany status przebiegu) |
| `platform_availability_status` | Status dostępności/renderowania raportu (odróżnia awarię platformy od błędu danych — patrz NFR-A-04) |

### 2.8 `TestResult` (Wynik pojedynczego testu)

Wynik jednego `TestDefinition` w ramach danego `TestExecution`.

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `test_execution_id` | FK → `TestExecution` |
| `test_definition_id` | FK → `TestDefinition` |
| `status` | `PASS` \| `FAIL` \| `WARNING` \| `ERROR` |
| `report_value` | Wartość odczytana z raportu |
| `sql_value` | Wartość z hurtowni danych (jeśli dotyczy) |
| `deviation_abs`, `deviation_pct` | Wyliczone odchylenie |
| `extraction_method` | Metoda ekstrakcji faktycznie użyta (patrz `00_Technical_Feasibility_Study.md`, sekcja 7) |
| `confidence_score` | Wskaźnik pewności odczytu (0.0–1.0) |
| `raw_evidence_ref` | Referencja do zapisanego dowodu (zrzut ekranu / fragment DOM / plik eksportu) |

### 2.9 `HealthScoreSnapshot`

Zagregowany, okresowy wskaźnik jakości raportu (do szybkiego renderowania trendów bez przeliczania całej historii na żądanie — denormalizacja celowa dla wydajności, patrz `05_NonFunctional_Requirements.md`, NFR-P-05).

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `report_id` | FK → `Report` |
| `snapshot_date` | Data agregacji |
| `health_score` | Wartość wskaźnika (0–100) |
| `pass_count`, `fail_count`, `warning_count` | Liczniki składowe |

### 2.10 `AIInsight`

Interpretacja wygenerowana przez moduł AI, powiązana z przebiegiem/wynikiem testu — **nigdy nie modyfikuje** encji `TestResult`/`TestExecution` (patrz `06_AI_Module.md`).

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `test_execution_id` | FK → `TestExecution` |
| `explanation` | Wyjaśnienie w języku naturalnym |
| `evidence` | Dane źródłowe, na których oparto wyjaśnienie (strukturalne, JSON) |
| `probable_causes` | Lista prawdopodobnych przyczyn |
| `suggested_actions` | Sugerowane działania naprawcze |
| `cluster_id` | Opcjonalne powiązanie z grupą podobnych błędów |
| `model_version`, `prompt_version` | Wersjonowanie dla audytu i odtwarzalności |
| `user_feedback` | Opcjonalna ocena trafności przez użytkownika (feedback loop) |
| `created_at` | Znacznik czasu |

### 2.11 `Notification`

Rekord wysłanego powiadomienia.

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `test_execution_id` | FK → `TestExecution` |
| `channel` | `EMAIL` \| inne kanały organizacyjne |
| `recipient` | Odbiorca |
| `sent_at` | Znacznik czasu |
| `status` | Status dostarczenia |

### 2.12 `User` / `Role`

| Atrybut | Opis |
|---|---|
| `User.id`, `User.username`, `User.email`, `User.role` | Konto użytkownika zsynchronizowane z SSO organizacji |
| `Role` | `ADMINISTRATOR` \| `ANALYST_OWNER` \| `VIEWER` (patrz `04_Functional_Requirements.md`, RBAC) |

### 2.13 `AuditLogEntry`

Append-only log audytowy (patrz `05_NonFunctional_Requirements.md`, NFR-C-01).

| Atrybut | Opis |
|---|---|
| `id` | Identyfikator |
| `actor_id` | FK → `User` (lub `SYSTEM`) |
| `action` | Typ zdarzenia (np. `PROFILE_UPDATED`, `TEST_TRIGGERED_MANUALLY`) |
| `entity_type`, `entity_id` | Encja, której dotyczy zdarzenie |
| `before_state`, `after_state` | Stan przed/po (JSON) |
| `timestamp` | Znacznik czasu |

---

## 3. Diagram relacji (skrót)

Pełny diagram — `uml/ERD.puml`. Kluczowe relacje:

```
BusinessArea (1) ──── (N) Report
Report (1) ──── (N) ValidationProfile           [wersje w czasie]
ValidationProfile (1) ──── (N) TestDefinition
TestDefinition (1) ──── (0..1) SqlValidationQuery
TestDefinition (1) ──── (0..1) BusinessRule
Report (1) ──── (N) TestExecution
ValidationProfile (1) ──── (N) TestExecution     [wersja użyta w danym przebiegu]
TestExecution (1) ──── (N) TestResult
TestDefinition (1) ──── (N) TestResult
TestExecution (1) ──── (0..N) AIInsight
TestExecution (1) ──── (0..N) Notification
Report (1) ──── (N) HealthScoreSnapshot
User (1) ──── (N) ValidationProfile              [created_by]
User/SYSTEM (1) ──── (N) AuditLogEntry
```

---

## 4. Strategia retencji i partycjonowania

Ze względu na skalę docelową (≥ 1000 raportów, wiele przebiegów testowych dziennie, wieloletnia historia), tabele `TestExecution`, `TestResult` i `AuditLogEntry` są kandydatami do **partycjonowania po czasie** (np. partycje miesięczne w PostgreSQL, `pg_partman` lub natywne partycjonowanie deklaratywne) w celu:

- utrzymania wydajności zapytań na "świeżych" danych (najczęściej odpytywanych przez Dashboard),
- umożliwienia efektywnego archiwizowania/usuwania starych partycji zgodnie z polityką retencji (Data Governance),
- ograniczenia kosztu operacji VACUUM/REINDEX na dużych tabelach.

`HealthScoreSnapshot` jest tabelą celowo zdenormalizowaną (agregaty prekalkulowane cyklicznie, a nie liczone on-the-fly z `TestResult`), aby zapewnić szybkie ładowanie Dashboardu (NFR-P-05) niezależnie od rozmiaru historii szczegółowej.

Rekomendowana wyjściowa polityka retencji (do potwierdzenia z Data Governance):
- `TestResult`/`TestExecution` (dane szczegółowe): min. 24 miesiące w bazie operacyjnej, starsze — archiwizacja do zimnego storage.
- `AuditLogEntry`: zgodnie z polityką audytową organizacji (typowo dłużej niż dane operacyjne).
- `AIInsight`: powiązane cyklem życia z `TestExecution`.

---

## 5. Powiązanie z dalszą dokumentacją

- Pełny diagram ERD — `uml/ERD.puml`.
- Diagram klas domenowych (zachowania, nie tylko dane) — `uml/ClassDiagram.puml`.
- Kontrakty API operujące na powyższych encjach — `08_API.md`.
- Implementacja (SQLAlchemy models, Alembic migrations) — `09_Python_Architecture.md`.
