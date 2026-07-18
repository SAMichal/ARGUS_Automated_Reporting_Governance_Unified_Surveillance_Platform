# 05. Non-Functional Requirements

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform \
**Dokument:** Wymagania niefunkcjonalne \
**Odbiorcy:** Architektura IT, Security Office, Zespół Deweloperski, Zespół Utrzymania

---

## 1. Wydajność (Performance)

| ID | Wymaganie | Kryterium akceptacji |
|---|---|---|
| NFR-P-01 | Pełny cykl testowy dla całego katalogu raportów (docelowo 1000) musi zakończyć się w oknie czasowym akceptowalnym operacyjnie. | ≤ 4 godziny dla 1000 raportów przy standardowej konfiguracji worker pool (do doprecyzowania liczby workerów w PoC — patrz `11_Roadmap.md`, Faza 0). |
| NFR-P-02 | Pojedynczy przebieg testowy (otwarcie raportu + ekstrakcja + walidacja SQL) nie powinien przekraczać rozsądnego czasu jednostkowego. | Mediana ≤ 15 s dla raportu o przeciętnej złożoności (Power BI), ≤ 10 s (SSRS); wartości referencyjne do potwierdzenia w PoC. |
| NFR-P-03 | System musi wykonywać testy równolegle, ze skalowalną liczbą workerów. | Konfigurowalny worker pool, skalowalny horyzontalnie (patrz `10_Deployment.md`). |
| NFR-P-04 | Zapytania do hurtowni danych (SQL Validation Engine) muszą być zoptymalizowane i nie obciążać nadmiernie hurtowni. | Zapytania wykonywane z jasno zdefiniowanym timeoutem, monitorowane pod kątem czasu wykonania; rekomendacja: dedykowana pula połączeń o ograniczonym limicie równoległości do hurtowni. |
| NFR-P-05 | Dashboard musi ładować podsumowanie stanu katalogu (1000 raportów) w czasie akceptowalnym dla użytkownika. | Czas ładowania widoku głównego ≤ 2 s przy typowym obciążeniu (dzięki zdenormalizowanym/zagregowanym widokom, patrz `07_Data_Model.md`). |

## 2. Skalowalność (Scalability)

| ID | Wymaganie |
|---|---|
| NFR-S-01 | Architektura musi wspierać wzrost katalogu z ok. 300 do ≥ 1000 raportów bez zmian architektonicznych — wyłącznie przez skalowanie horyzontalne warstwy wykonawczej (workery Browser Automation). |
| NFR-S-02 | Warstwa API i baza danych muszą wspierać skalowanie horyzontalne (stateless API, connection pooling do PostgreSQL, możliwość read-replik dla obciążeń analitycznych/dashboardowych w przyszłości). |
| NFR-S-03 | Kolejka zadań testowych (Scheduler → Test Orchestrator → workery) musi być odporna na skoki obciążenia (np. cały katalog uruchomiony ręcznie jednocześnie) poprzez mechanizm kolejkowania z limitem współbieżności (Celery/Redis lub RabbitMQ — patrz `09_Python_Architecture.md`). |
| NFR-S-04 | Model danych musi wspierać rosnący wolumen historii wyników (1000 raportów × wiele przebiegów dziennie × wieloletnia retencja) bez degradacji wydajności zapytań — strategia partycjonowania opisana w `07_Data_Model.md`. |

## 3. Bezpieczeństwo (Security)

System jest projektowany zgodnie z podejściem **Security by Design**, adekwatnym do środowiska instytucji finansowej.

| ID | Wymaganie |
|---|---|
| NFR-SEC-01 | Konto techniczne używane przez ARGUS do dostępu do Platformy BI i hurtowni danych musi działać zgodnie z zasadą **najmniejszych uprawnień (least privilege)** — wyłącznie odczyt raportów/danych niezbędnych do walidacji, bez uprawnień administracyjnych na platformie BI. |
| NFR-SEC-02 | Sekrety (dane uwierzytelniające konta technicznego, connection stringi, klucze API) muszą być przechowywane w dedykowanym, zarządzanym magazynie sekretów (np. HashiCorp Vault / Azure Key Vault / rozwiązanie wewnętrzne banku), nigdy w kodzie, plikach konfiguracyjnych repozytorium ani zmiennych środowiskowych w postaci jawnej w obrazach kontenerów. |
| NFR-SEC-03 | Dostęp użytkowników do GUI/API musi być uwierzytelniany przez istniejący mechanizm SSO organizacji, z autoryzacją opartą o RBAC (role: Administrator, Analityk/Właściciel, Przeglądający). |
| NFR-SEC-04 | Cała komunikacja sieciowa (GUI ↔ API, API ↔ workery, workery ↔ Platforma BI/hurtownia) musi być szyfrowana (TLS) zgodnie z polityką organizacji. |
| NFR-SEC-05 | Dane wrażliwe w logach technicznych muszą być maskowane (np. nie logujemy pełnych wyników zapytań SQL zawierających dane biznesowe, o ile nie jest to jawnie wymagane dla celów audytowych w kontrolowanym, ograniczonym dostępowo repozytorium). |
| NFR-SEC-06 | System musi podlegać standardowym procesom bezpieczeństwa organizacji przed wdrożeniem produkcyjnym: testy podatności / testy penetracyjne, przegląd kodu pod kątem bezpieczeństwa (SAST), skanowanie zależności (SCA) w pipeline CI/CD. |
| NFR-SEC-07 | Wszelkie treści generowane przez moduł AI wysyłane do zewnętrznego dostawcy modelu (jeśli wybrany wariant chmurowy — patrz `06_AI_Module.md`) muszą być pozbawione danych osobowych i danych sklasyfikowanych jako wrażliwe zgodnie z polityką klasyfikacji danych organizacji; rekomendowany wariant **Azure OpenAI w ramach dzierżawy (tenant) organizacji** z kontraktowym brakiem wykorzystania danych do trenowania modeli, alternatywnie hostowany model open-weight w infrastrukturze wewnętrznej — decyzja do podjęcia z Security Office w Fazie 0. |
| NFR-SEC-08 | System musi wspierać segmentację sieciową — komponenty wykonawcze (workery Browser Automation) uruchamiane w segmencie sieci z kontrolowanym dostępem wyłącznie do niezbędnych zasobów (Platforma BI, hurtownia danych), zgodnie z zasadą defense-in-depth. |

## 4. Dostępność i niezawodność (Availability & Reliability)

| ID | Wymaganie |
|---|---|
| NFR-A-01 | Docelowa dostępność systemu (API + GUI) w godzinach pracy organizacji: **≥ 99.5%**. |
| NFR-A-02 | Awaria pojedynczego workera (Browser Automation) nie może wpływać na dostępność pozostałych przebiegów testowych — izolacja procesów wykonawczych. |
| NFR-A-03 | System musi implementować mechanizmy odpornościowe (Retry z backoff, Circuit Breaker, Timeout) dla wszystkich integracji zewnętrznych (Platforma BI, hurtownia danych) — patrz `03_Architecture.md`, sekcja 3.6. |
| NFR-A-04 | Chwilowa niedostępność Platformy BI lub hurtowni danych musi skutkować jednoznacznym, odróżnialnym statusem technicznym (np. `PLATFORM_UNAVAILABLE`), a nie fałszywym wynikiem FAIL sugerującym błąd raportu. |
| NFR-A-05 | System musi posiadać zdefiniowaną strategię Disaster Recovery (kopie zapasowe bazy danych, RPO/RTO — do sprecyzowania z zespołem infrastruktury, patrz `10_Deployment.md`). |

## 5. Utrzymywalność (Maintainability)

| ID | Wymaganie |
|---|---|
| NFR-M-01 | Kod musi być zgodny z zasadami SOLID/Clean Architecture opisanymi w `03_Architecture.md`, zweryfikowanymi w code review. |
| NFR-M-02 | Pokrycie testami jednostkowymi warstwy domenowej (Validation Engine, Business Rule Engine) — cel: **≥ 85%**. |
| NFR-M-03 | Każdy connector platformy raportowej musi posiadać dedykowany zestaw testów kontraktowych (contract tests) weryfikujących zgodność z interfejsem `IReportConnector`. |
| NFR-M-04 | Statyczna analiza kodu (Ruff, MyPy w trybie strict) musi być częścią pipeline CI, blokującą merge w przypadku naruszeń. |
| NFR-M-05 | Dokumentacja architektoniczna (niniejsze repozytorium) musi być aktualizowana równolegle ze zmianami architektonicznymi jako część definition of done dla zadań o charakterze architektonicznym. |

## 6. Obserwowalność (Observability)

| ID | Wymaganie |
|---|---|
| NFR-O-01 | System musi eksponować metryki operacyjne w formacie kompatybilnym z Prometheus (czas wykonania testów, liczba sukcesów/błędów per connector, stan worker pool, opóźnienia kolejki). |
| NFR-O-02 | System musi udostępniać dashboardy operacyjne w Grafanie dla zespołu utrzymania (odrębne od Dashboardu biznesowego dla użytkowników — patrz `04_Functional_Requirements.md`). |
| NFR-O-03 | System musi implementować rozproszone śledzenie (distributed tracing) zgodne z OpenTelemetry, umożliwiające prześledzenie pojedynczego przebiegu testowego przez wszystkie moduły. |
| NFR-O-04 | Logi muszą być ustrukturyzowane (JSON) i zawierać `trace_id`/`correlation_id` umożliwiający korelację zdarzeń między modułami/workerami. |

## 7. Zgodność i audytowalność (Compliance)

| ID | Wymaganie |
|---|---|
| NFR-C-01 | Każda zmiana Validation Profile musi być rejestrowana w logu audytowym z identyfikacją autora, znacznikiem czasu i pełną treścią zmiany (before/after). |
| NFR-C-02 | Wyniki testów muszą być przechowywane w sposób niemodyfikowalny (append-only) przez okres retencji zgodny z polityką organizacji (do potwierdzenia z Data Governance — rekomendacja wyjściowa: minimum 24 miesiące, patrz `07_Data_Model.md`). |
| NFR-C-03 | System musi umożliwiać wygenerowanie raportu audytowego (kto, kiedy, co zmienił/uruchomił) na żądanie audytora wewnętrznego. |
| NFR-C-04 | Decyzje modułu AI muszą być logowane wraz z uzasadnieniem i danymi wejściowymi, na których się opierały, dla celów wyjaśnialności (explainability) i audytu — patrz `06_AI_Module.md`. |

## 8. Użyteczność (Usability)

| ID | Wymaganie |
|---|---|
| NFR-U-01 | GUI musi być dostępne bez konieczności instalacji dodatkowego oprogramowania po stronie użytkownika (standardowa przeglądarka wspierana w organizacji). |
| NFR-U-02 | Interfejs musi jednoznacznie odróżniać wizualnie treści generowane deterministycznie (wynik Validation Engine) od treści generowanych przez AI (interpretacja wspierająca) — wymaganie wynikające wprost z zasady nadrzędnej projektu. |
| NFR-U-03 | GUI powinno być responsywne (wsparcie dla typowych rozdzielczości stacji roboczych organizacji); wsparcie mobilne nie jest wymagane w MVP. |

## 9. Przenośność (Portability)

| ID | Wymaganie |
|---|---|
| NFR-PO-01 | Komponenty systemu muszą być konteneryzowane (Docker), umożliwiając uruchomienie w różnych środowiskach (Dev/Test/UAT/Prod) bez zmian w kodzie — wyłącznie przez konfigurację. |
| NFR-PO-02 | System nie może zakładać twardej zależności od konkretnego dostawcy chmury — komponenty rdzeniowe (API, baza danych, workery) muszą działać on-prem zgodnie z założeniem projektowym; wyjątkiem jest opcjonalny, kontraktowo kontrolowany dostawca modelu AI (patrz NFR-SEC-07). |

## 10. Podsumowanie — macierz wymagań krytycznych

| Wymiar | Wymóg krytyczny | Ryzyko przy niespełnieniu |
|---|---|---|
| Wydajność | Pełny cykl 1000 raportów w akceptowalnym oknie czasowym | Brak realnej wartości biznesowej — system "nie nadąża" za katalogiem |
| Bezpieczeństwo | Least privilege konta technicznego + zarządzany magazyn sekretów | Brak akceptacji Security Office, ryzyko incydentu bezpieczeństwa |
| Niezawodność | Odróżnienie awarii platformy od błędu raportu | Fałszywe alarmy, utrata zaufania biznesu do systemu |
| Audytowalność | Pełny, niemodyfikowalny ślad audytowy | Brak zgodności z wymogami compliance instytucji finansowej |
| Wyjaśnialność AI | Logowanie uzasadnień decyzji AI | Brak akceptacji dla wykorzystania AI w procesie decyzyjnym |

Powiązanie: szczegóły infrastruktury realizującej powyższe wymagania — `10_Deployment.md`; rejestr ryzyk związanych z niespełnieniem powyższych wymogów — `12_Risks.md`.
