# 08. API

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform \
**Dokument:** Projekt API REST \
**Odbiorcy:** Zespół Deweloperski, Zespoły integrujące się z ARGUS

---

## 1. Zasady projektowe API

1. **REST nad HTTP/JSON**, zgodny ze specyfikacją **OpenAPI 3.1** — dokumentacja API generowana automatycznie z kodu (FastAPI generuje schemat OpenAPI natywnie na bazie modeli Pydantic, co eliminuje ryzyko rozjazdu między dokumentacją a implementacją).
2. **Wersjonowanie w ścieżce** (`/api/v1/...`) — pozwala na wprowadzanie zmian niekompatybilnych wstecz bez łamania istniejących integracji (np. przyszłych integracji z innymi systemami banku).
3. **Zasoby, nie akcje** — endpointy modelowane wokół rzeczowników (`/reports`, `/validation-profiles`, `/test-executions`), zgodnie z konwencją REST; wyjątki (akcje typu "uruchom test") modelowane jako pod-zasoby/operacje (`POST /reports/{id}/test-executions`).
4. **Spójna struktura odpowiedzi błędu** — zgodna z RFC 7807 (`application/problem+json`): `type`, `title`, `status`, `detail`, `instance`, ułatwiająca jednolitą obsługę błędów po stronie klienta (GUI).
5. **Paginacja kursorowa** dla list mogących urosnąć do dużych rozmiarów (`test-executions`, `audit-log`) — stabilniejsza niż offsetowa przy dużym, stale rosnącym wolumenie danych.
6. **Idempotencja** dla operacji tworzących zasoby wywoływanych z potencjalnym retry po stronie klienta (np. ręczne wyzwolenie testu) — poprzez nagłówek `Idempotency-Key`.
7. **Autoryzacja na poziomie zasobu** — każdy endpoint modyfikujący dane weryfikuje rolę RBAC (patrz `04_Functional_Requirements.md`, FR-060) poprzez zależność FastAPI (`Depends(require_role(...))`).

---

## 2. Uwierzytelnianie i autoryzacja

- Uwierzytelnianie oparte o token wydany po SSO organizacji (np. OIDC/OAuth2 Authorization Code Flow z Azure AD/AD FS — do potwierdzenia z zespołem IAM w Fazie 0), przekazywany jako `Bearer` token w nagłówku `Authorization`.
- API waliduje token (podpis, ważność, `aud`/`iss`) i mapuje tożsamość na encję `User` z przypisaną rolą RBAC.
- Komunikacja API ↔ workery (wewnętrzna, service-to-service) zabezpieczona osobnym mechanizmem (mTLS lub token serwisowy w sieci wewnętrznej) — nieeksponowana na zewnątrz.

---

## 3. Zasoby główne i endpointy

### 3.1 `Reports` — katalog raportów

| Metoda | Endpoint | Opis | Rola min. |
|---|---|---|---|
| `GET` | `/api/v1/reports` | Lista raportów (filtrowanie: `business_area_id`, `platform_type`, `status`, wyszukiwanie tekstowe `q`) | Viewer |
| `GET` | `/api/v1/reports/{report_id}` | Szczegóły raportu, w tym aktualny Validation Profile i ostatni wynik | Viewer |
| `POST` | `/api/v1/reports` | Rejestracja nowego raportu w katalogu | Administrator |
| `PATCH` | `/api/v1/reports/{report_id}` | Aktualizacja metadanych raportu (właściciel, obszar biznesowy, aktywność) | Administrator |
| `GET` | `/api/v1/business-areas` | Lista obszarów biznesowych (zakładek) | Viewer |

### 3.2 `Validation Profiles`

| Metoda | Endpoint | Opis | Rola min. |
|---|---|---|---|
| `GET` | `/api/v1/reports/{report_id}/validation-profiles` | Historia wersji profilu dla raportu | Viewer |
| `GET` | `/api/v1/validation-profiles/{profile_id}` | Szczegóły konkretnej wersji profilu | Viewer |
| `POST` | `/api/v1/reports/{report_id}/validation-profiles` | Utworzenie nowej wersji Validation Profile (testy, harmonogram, tolerancje) | Analyst/Owner |
| `POST` | `/api/v1/validation-profiles/{profile_id}/activate` | Aktywacja danej wersji jako obowiązującej | Analyst/Owner |
| `POST` | `/api/v1/validation-profiles/{profile_id}/clone` | Klonowanie profilu jako punkt startowy dla innego raportu (FR-018) | Analyst/Owner |

### 3.3 `Test Definitions` (w ramach profilu)

| Metoda | Endpoint | Opis | Rola min. |
|---|---|---|---|
| `GET` | `/api/v1/validation-profiles/{profile_id}/test-definitions` | Lista testów w profilu | Viewer |
| `POST` | `/api/v1/validation-profiles/{profile_id}/test-definitions` | Dodanie testu (SQL / wartość pola / reguła biznesowa) | Analyst/Owner |
| `POST` | `/api/v1/test-definitions/validate` | Walidacja poprawności definicji testu **przed zapisem** (Test Definition Engine) — sprawdzenie składni SQL, poprawności konfiguracji ekstrakcji | Analyst/Owner |
| `DELETE` | `/api/v1/test-definitions/{test_id}` | Dezaktywacja testu (soft-delete, zachowanie w historii) | Analyst/Owner |

### 3.4 `Test Executions` — wykonania testów

| Metoda | Endpoint | Opis | Rola min. |
|---|---|---|---|
| `POST` | `/api/v1/reports/{report_id}/test-executions` | Ręczne uruchomienie testu dla raportu (wymaga `Idempotency-Key`) | Analyst/Owner |
| `POST` | `/api/v1/business-areas/{area_id}/test-executions` | Ręczne uruchomienie testów dla całego obszaru biznesowego | Analyst/Owner |
| `GET` | `/api/v1/test-executions` | Lista przebiegów testowych (filtrowanie: `report_id`, `status`, zakres dat; paginacja kursorowa) | Viewer |
| `GET` | `/api/v1/test-executions/{execution_id}` | Szczegóły przebiegu wraz z wynikami poszczególnych testów (`TestResult`) | Viewer |
| `GET` | `/api/v1/test-executions/{execution_id}/status` | Lekki endpoint statusu (do odpytywania "w toku" z GUI) | Viewer |

### 3.5 `Health & Confidence Scores`

| Metoda | Endpoint | Opis | Rola min. |
|---|---|---|---|
| `GET` | `/api/v1/reports/{report_id}/health-score?from=&to=` | Trend Health Score raportu w zadanym okresie | Viewer |
| `GET` | `/api/v1/business-areas/{area_id}/health-score` | Zagregowany Health Score obszaru biznesowego | Viewer |
| `GET` | `/api/v1/dashboard/summary` | Zagregowane podsumowanie dla ekranu głównego (liczniki statusów, trend globalny) — endpoint zoptymalizowany pod NFR-P-05 | Viewer |

### 3.6 `AI Insights`

| Metoda | Endpoint | Opis | Rola min. |
|---|---|---|---|
| `GET` | `/api/v1/test-executions/{execution_id}/ai-insight` | Interpretacja AI powiązana z przebiegiem (jeśli wygenerowana) | Viewer |
| `GET` | `/api/v1/ai-insights/clusters` | Lista klastrów podobnych błędów (FR-041) | Viewer |
| `POST` | `/api/v1/ai-insights/{insight_id}/feedback` | Ocena trafności sugestii AI przez użytkownika (feedback loop) | Analyst/Owner |

### 3.7 `Notifications`

| Metoda | Endpoint | Opis | Rola min. |
|---|---|---|---|
| `GET` | `/api/v1/notifications` | Historia wysłanych powiadomień (filtrowanie) | Analyst/Owner |
| `POST` | `/api/v1/validation-profiles/{profile_id}/notification-rules` | Konfiguracja reguł powiadomień/eskalacji dla profilu | Analyst/Owner |

### 3.8 Administracja

| Metoda | Endpoint | Opis | Rola min. |
|---|---|---|---|
| `GET`/`POST` | `/api/v1/admin/users` | Zarządzanie użytkownikami i rolami (RBAC) | Administrator |
| `GET`/`POST` | `/api/v1/admin/connectors` | Zarządzanie katalogiem platform/connectorów (bez ekspozycji sekretów) | Administrator |
| `GET` | `/api/v1/admin/audit-log` | Przegląd logu audytowego (filtrowanie, paginacja kursorowa) | Administrator |
| `GET` | `/api/v1/admin/system-metrics` | Skrócony widok metryk operacyjnych wewnątrz aplikacji (uzupełniający Grafanę) | Administrator |

---

## 4. Przykładowy kontrakt — wynik przebiegu testowego

Poniżej ilustracyjny (skrócony) kształt odpowiedzi `GET /api/v1/test-executions/{execution_id}`, prezentujący kluczowe pola istotne dla warstwy Presentation:

```json
{
  "execution_id": "a1b2c3d4",
  "report_id": "rep-00231",
  "report_name": "Sprzedaż / Konsorcja Kredytowe",
  "platform_type": "SSRS",
  "triggered_by": "SCHEDULER",
  "started_at": "2026-07-18T02:00:04Z",
  "finished_at": "2026-07-18T02:00:19Z",
  "overall_status": "WARNING",
  "platform_availability_status": "AVAILABLE",
  "test_results": [
    {
      "test_definition_id": "td-0091",
      "test_name": "KPI_SPRZEDAZ_TOTAL",
      "status": "WARNING",
      "report_value": "1204500.00",
      "sql_value": "1189300.00",
      "deviation_pct": 1.28,
      "tolerance_pct": 1.00,
      "extraction_method": "DOM_ANALYSIS",
      "confidence_score": 0.94
    }
  ],
  "ai_insight_available": true
}
```

Uwaga projektowa: pole `extraction_method` i `confidence_score` są celowo eksponowane na poziomie API (a nie tylko wewnętrznie), zgodnie z wymogiem transparentności (FR-033, UC-13) — użytkownik biznesowy musi mieć możliwość oceny, jak bardzo ufać danemu wynikowi.

---

## 5. Obsługa błędów

Zgodnie z RFC 7807, przykład:

```json
{
  "type": "https://argus.bank.com.pl/errors/platform-unavailable",
  "title": "Platforma raportowa niedostępna",
  "status": 503,
  "detail": "SSRS Report Server nie odpowiedział w oczekiwanym czasie (timeout 30s).",
  "instance": "/api/v1/reports/rep-00231/test-executions/a1b2c3d4"
}
```

Kategorie błędów standaryzowane w całym API: `validation-error` (400), `unauthorized` (401), `forbidden` (403), `not-found` (404), `conflict` (409, np. równoległa edycja tego samego profilu), `platform-unavailable` (503, zgodnie z NFR-A-04), `internal-error` (500).

---

## 6. Webhooki / integracje wychodzące (rozszerzenie przyszłościowe)

Poza zakresem MVP, ale architektonicznie przewidziane (zgodnie z zasadą Open/Closed): możliwość rejestracji webhooków (`POST /api/v1/admin/webhooks`) powiadamiających zewnętrzne systemy organizacji (np. centralny system ticketowy) o zdarzeniach `test_execution.failed` — zaprojektowane jako rozszerzenie `Notification Service` o kolejny `INotificationChannel`, bez zmian w rdzeniu.

---

## 7. Powiązanie z dalszą dokumentacją

- Encje leżące pod powyższymi zasobami — `07_Data_Model.md`.
- Implementacja API w FastAPI, struktura routerów — `09_Python_Architecture.md`.
- Wymagania bezpieczeństwa dot. uwierzytelniania i szyfrowania transportu — `05_NonFunctional_Requirements.md`.
- Diagram sekwencji ilustrujący przepływ wywołań dla scenariusza wykonania testu — `uml/SequenceDiagram.puml`.
