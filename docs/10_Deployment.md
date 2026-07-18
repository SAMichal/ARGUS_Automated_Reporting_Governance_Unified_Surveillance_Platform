# 10. Deployment

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform
**Dokument:** Środowiska, CI/CD, infrastruktura, Disaster Recovery
**Odbiorcy:** Architektura IT, Zespół Infrastruktury/Platform, Security Office

---

## 1. Środowiska

| Środowisko | Cel | Charakterystyka |
|---|---|---|
| **Dev** | Praca deweloperska, integracja bieżąca | Docker Compose lokalnie/na współdzielonym hoście deweloperskim; dane testowe/syntetyczne; mockowane connectory platform BI dopuszczalne dla szybkiej iteracji. |
| **Test** | Testy automatyczne (integracyjne, kontraktowe), testy zespołu QA | Zbliżone do Prod pod względem topologii, ograniczony worker pool; połączenie z testowym/kopią środowiska Platformy BI, jeśli dostępne, w innym wypadku z reprezentatywnym podzbiorem raportów produkcyjnych (read-only). |
| **UAT** | Akceptacja biznesowa, przegląd bezpieczeństwa | Konfiguracja zbliżona do Prod, ograniczony podzbiór raportów produkcyjnych, pełna ścieżka SSO/autoryzacji jak w Prod. |
| **Prod** | Produkcja | Pełna skala (docelowo 1000 raportów), pełny worker pool, pełny monitoring i alerting, wysoka dostępność (NFR-A-01). |

Wszystkie środowiska konteneryzowane (NFR-PO-01), różnice wyłącznie konfiguracyjne (zmienne środowiskowe / wpisy w magazynie sekretów), zgodnie z zasadą 12-Factor App.

---

## 2. Topologia infrastruktury (on-prem, zgodnie z założeniem projektowym)

```
                         ┌──────────────────────────────┐
                         │      Load Balancer / Reverse   │
                         │      Proxy (organizacyjny)     │
                         └───────────────┬────────────────┘
                                          │ TLS
                     ┌────────────────────┼─────────────────────┐
                     ▼                                          ▼
           ┌──────────────────┐                        ┌──────────────────┐
           │   ARGUS API       │  (N replik, stateless) │   ARGUS GUI       │
           │   (FastAPI)       │◀──────REST/JSON───────▶│   (React SPA,     │
           │   Kubernetes Pod  │                        │   statyczne pliki)│
           └─────────┬─────────┘                        └──────────────────┘
                     │
        ┌────────────┼─────────────────────┐
        ▼            ▼                     ▼
 ┌─────────────┐ ┌───────────┐   ┌───────────────────┐
 │ PostgreSQL   │ │  Redis     │   │  Celery Workers    │  (N replik, autoscaling
 │ (Primary +   │ │ (broker +  │   │  (Browser Automation│   wg obciążenia kolejki)
 │  replika     │ │  cache)    │   │  Engine + Playwright │
 │  do odczytu) │ │            │   │  + Connectory)       │
 └─────────────┘ └───────────┘   └──────────┬─────────┘
                                              │  (segment sieci kontrolowany —
                                              │   NFR-SEC-08)
                                  ┌───────────┴────────────┐
                                  ▼                        ▼
                         ┌─────────────────┐     ┌──────────────────┐
                         │  Platforma BI     │     │  Hurtownia danych │
                         │  (Power BI /SSRS) │     │  (SQL Server)      │
                         └─────────────────┘     └──────────────────┘

           Warstwa obserwowalności (równoległa do powyższych):
           Prometheus ← metryki ← (API, Workery, Scheduler)
           Grafana ← dashboardy operacyjne
           OpenTelemetry Collector ← trace'y rozproszone
           Centralny system logów organizacji ← logi ustrukturyzowane (JSON)

           Magazyn sekretów (Vault / Azure Key Vault / rozwiązanie wewnętrzne)
           ← konto techniczne, connection stringi, klucze API (NFR-SEC-02)
```

Pełny, formalny diagram wdrożenia — patrz `uml/DeploymentDiagram.puml`.

### 2.1 Uzasadnienie kluczowych decyzji topologicznych

- **API jako komponent bezstanowy (stateless)**, umożliwiający skalowanie horyzontalne i pracę za load balancerem bez sticky session — zgodnie z NFR-S-02.
- **Wydzielenie Celery Workers jako osobnej warstwy skalowanej niezależnie od API** — obciążenie generowane przez Browser Automation (CPU/RAM na instancję przeglądarki) ma zupełnie inną charakterystykę niż obciążenie API, więc niezależne skalowanie obu warstw optymalizuje koszt infrastruktury.
- **Segmentacja sieciowa workerów** — workery mają dostęp wyłącznie do Platformy BI i hurtowni danych (NFR-SEC-08), nie do szerszej sieci korporacyjnej, minimalizując powierzchnię ataku w razie kompromitacji kontenera wykonawczego.
- **PostgreSQL z replika do odczytu** — przygotowanie pod przyszłe obciążenia analityczne/raportowe generowane przez sam Dashboard ARGUS bez obciążania instancji operacyjnej (NFR-S-02).

---

## 3. CI/CD — pipeline (GitHub Actions)

```
Push / Pull Request
     │
     ▼
[1] Lint & Format Check         — Ruff
     │
     ▼
[2] Static Type Check           — MyPy (strict)
     │
     ▼
[3] Architectural Boundary Check — import-linter (patrz 09_Python_Architecture.md, sekcja 5)
     │
     ▼
[4] Unit Tests                  — pytest (domain/, application/) + coverage gate (≥85%, NFR-M-02)
     │
     ▼
[5] Contract Tests               — pytest (każdy IReportConnector, NFR-M-03)
     │
     ▼
[6] Dependency Scan (SCA)        — np. pip-audit / Dependabot
     │
     ▼
[7] Build obrazów Docker         — argus-api, argus-worker, argus-ui
     │
     ▼
[8] Static Application Security Testing (SAST)
     │
     ▼
[9] Publikacja do wewnętrznego registry kontenerów organizacji
     │
     ▼
[10] Integration Tests (środowisko Test)  — testcontainers + mockowane platformy BI
     │
     ▼
[11] Manualna akceptacja / Change Management  → Deploy do UAT → Deploy do Prod
```

Kroki 1–9 wykonywane automatycznie na każdy Pull Request; wdrożenie do UAT/Prod podlega standardowemu procesowi zarządzania zmianą organizacji (poza automatycznym pipeline, zgodnie z procedurami banku).

---

## 4. Zarządzanie sekretami i kontem technicznym

- Konto techniczne (dostęp do Platformy BI i hurtowni danych) zarządzane zgodnie z polityką IAM organizacji — rekomendacja: **Group Managed Service Account (gMSA)** w środowisku Active Directory, jeśli dostępne, jako mechanizm eliminujący konieczność ręcznej rotacji haseł kontenerowych; alternatywnie konto serwisowe z hasłem rotowanym automatycznie przez magazyn sekretów.
- Sekrety wstrzykiwane do kontenerów wyłącznie w runtime (np. jako zamontowane pliki/zmienne środowiskowe pobierane z Vault/Key Vault przy starcie), nigdy zapisane w obrazie Docker ani w repozytorium (NFR-SEC-02).
- Rotacja sekretów — zgodnie z polityką organizacji, wspierana architektonicznie przez brak twardego cache'owania danych uwierzytelniających dłużej niż czas życia sesji roboczej workera.

---

## 5. Skalowanie

| Komponent | Strategia skalowania |
|---|---|
| ARGUS API | Horyzontalne (liczba replik Kubernetes Pod), na podstawie obciążenia CPU/liczby requestów. |
| Celery Workers (Browser Automation) | Horyzontalne, na podstawie długości kolejki zadań (autoscaling oparty o metrykę głębokości kolejki Redis/RabbitMQ) — kluczowe dla realizacji NFR-P-01 (pełny cykl 1000 raportów w oknie ≤ 4h). |
| PostgreSQL | Wertykalne w pierwszej kolejności; read-replika dla obciążeń odczytowych Dashboardu w miarę potrzeb. |
| Redis | Wertykalne / Redis Cluster w miarę wzrostu obciążenia kolejki i cache. |

Docelowa liczba workerów i parametry autoscalingu wymagają kalibracji empirycznej w ramach Proof of Concept (Faza 0, patrz `11_Roadmap.md`) — dokument nie zakłada z góry konkretnej liczby, ponieważ zależy ona od rzeczywistej charakterystyki wydajnościowej raportów w środowisku organizacji.

---

## 6. Disaster Recovery i kopie zapasowe

| Aspekt | Rekomendacja wyjściowa |
|---|---|
| Kopie zapasowe PostgreSQL | Pełne kopie okresowe + Point-in-Time Recovery (WAL archiving), zgodnie ze standardem organizacji dla baz produkcyjnych. |
| RPO (Recovery Point Objective) | Do potwierdzenia z zespołem infrastruktury — rekomendacja wyjściowa: ≤ 1 godzina dla danych operacyjnych. |
| RTO (Recovery Time Objective) | Do potwierdzenia z zespołem infrastruktury — rekomendacja wyjściowa: ≤ 4 godziny. |
| Odtwarzalność konfiguracji | Cała konfiguracja aplikacji (poza sekretami) wersjonowana w repozytorium (Infrastructure as Code — manifesty Kubernetes/Helm chart), umożliwiając pełne odtworzenie środowiska z repozytorium + magazynu sekretów. |
| Odporność na awarię pojedynczego workera | Nie wymaga procedur DR — architektonicznie izolowana (NFR-A-02), automatyczne wznowienie zadania przez Celery. |

Ostateczne wartości RPO/RTO oraz procedury DR muszą zostać zatwierdzone przez zespół infrastruktury/Business Continuity organizacji jako część standardowego procesu dla systemów klasy Enterprise — niniejszy dokument dostarcza rekomendacji wyjściowej do tej dyskusji, nie ostatecznej decyzji.

---

## 7. Integracja z monitoringiem organizacyjnym

- Metryki Prometheus eksponowane na standardowym endpointcie `/metrics` każdego komponentu, zbierane przez istniejącą infrastrukturę Prometheus organizacji (jeśli dostępna) lub dedykowaną instancję projektu.
- Logi w formacie JSON przekazywane do centralnego systemu logowania organizacji (np. ELK/Splunk — zależnie od standardu wewnętrznego, do potwierdzenia w Fazie 0).
- Alerting (Alertmanager/Grafana Alerting) skonfigurowany dla scenariuszy krytycznych: worker pool wyczerpany, wskaźnik błędów connectora powyżej progu, opóźnienie kolejki powyżej progu, niedostępność API.

---

## 8. Powiązanie z dalszą dokumentacją

- Wymagania niefunkcjonalne realizowane przez powyższą infrastrukturę — `05_NonFunctional_Requirements.md`.
- Stack technologiczny i struktura kodu — `09_Python_Architecture.md`.
- Diagram wdrożenia (formalny UML) — `uml/DeploymentDiagram.puml`.
- Plan fazowy wdrożenia produkcyjnego — `11_Roadmap.md`.
- Ryzyka związane z infrastrukturą i operacjami — `12_Risks.md`.
