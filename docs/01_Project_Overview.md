# 01. Project Overview

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform \
**Dokument:** Przegląd projektu (Project Overview) \
**Odbiorcy:** Sponsor, Project Manager, Architektura IT, Zespół Deweloperski

---

## 1. Nazwa projektu i uzasadnienie

**ARGUS** — od Argosa Panoptesa, stuokiego strażnika z mitologii greckiej, którego oczy nigdy nie zasypiały w komplecie. System ma pełnić dokładnie taką rolę: nieustannie "obserwować" setki, a docelowo tysiąc raportów jednocześnie, wychwytując nieprawidłowości niewidoczne dla pojedynczego obserwatora działającego ręcznie.

Nazwa spełnia kryteria dobrej nazwy produktu enterprise:
- **krótka i łatwa do zapamiętania** w każdym języku i kontekście (rozmowy, dokumentacja, commit messages),
- **nośnik metafory zgodnej z funkcją systemu** (czujność, kompletność obserwacji, brak "martwych pól"),
- **neutralna technologicznie** — nie sugeruje wyłącznie Power BI ani wyłącznie SSRS, co jest zgodne z zasadą niezależności platformowej systemu,
- **dobrze skracalna** w nazewnictwie technicznym repozytoriów/usług: `argus-core`, `argus-connectors`, `argus-ai`, `argus-ui`, `argus-api`.

Pełna nazwa formalna do użytku w dokumentach zarządczych: **ARGUS — Automated Reporting Governance & Unified Surveillance Platform**.

---

## 2. Wizja produktu

> *Żaden błąd w raporcie BI nie dotrze do decydenta biznesowego, zanim nie zostanie automatycznie wykryty, zweryfikowany względem źródła danych i zgłoszony — szybciej i pewniej, niż mógłby to zrobić zespół analityków pracujący ręcznie.*

ARGUS ma zastąpić czasochłonny, ręczny proces kontroli jakości raportów Platformy BI systemem, który:

- działa **w sposób ciągły i deterministyczny**, a nie okazjonalny i uznaniowy,
- **skaluje się liniowo** wraz ze wzrostem liczby raportów (z ok. 300 do docelowo ≥ 1000, a w przyszłości potencjalnie więcej),
- **buduje historię i ślad audytowy** jakości każdego raportu w czasie — czego dziś praktycznie nie ma,
- **wspiera, a nie zastępuje** kompetencje analityczne zespołu — poprzez moduł AI, który tłumaczy wyniki i wskazuje prawdopodobne przyczyny, pozostawiając ostateczną ocenę deterministycznemu silnikowi reguł, a decyzję biznesową — człowiekowi.

---

## 3. Cele projektu

### 3.1 Cele biznesowe

| ID | Cel biznesowy | Miernik sukcesu |
|---|---|---|
| BC-01 | Redukcja czasu i kosztu ręcznej weryfikacji raportów | Skrócenie czasu potrzebnego na pełen przegląd katalogu raportów z dni do minut/godzin |
| BC-02 | Wzrost zaufania do danych prezentowanych w raportach BI | Spadek liczby incydentów "błędny raport wykryty przez biznes zamiast przez IT" |
| BC-03 | Skalowalność procesu QA wraz ze wzrostem katalogu raportów | System obsługujący ≥ 1000 raportów bez wzrostu nakładu pracy ludzkiej proporcjonalnego do liczby raportów |
| BC-04 | Budowa historii jakości danych jako aktywa organizacji | Dostępna, przeszukiwalna historia wyników testów dla każdego raportu od dnia wdrożenia |
| BC-05 | Wsparcie zgodności i audytowalności (compliance) | Pełny ślad audytowy: kto, kiedy, jaki test zdefiniował/zmienił, jaki był wynik i dlaczego |

### 3.2 Cele techniczne

| ID | Cel techniczny | Powiązany dokument |
|---|---|---|
| TC-01 | Architektura niezależna od konkretnej technologii raportowej (rozszerzalna o nowe platformy) | `03_Architecture.md` |
| TC-02 | Jednolity mechanizm testowy dla Power BI i SSRS | `00_Technical_Feasibility_Study.md` |
| TC-03 | Odporność na drobne zmiany wizualne raportów (self-healing/fallback) | `00_Technical_Feasibility_Study.md`, sekcja 7 |
| TC-04 | Pełna separacja logiki domenowej od infrastruktury (Clean Architecture, SOLID, DI) | `03_Architecture.md` |
| TC-05 | Wyjaśnialność każdej decyzji o błędzie/poprawności raportu | `06_AI_Module.md` |
| TC-06 | Bezpieczeństwo klasy bankowej (SSO, RBAC, audyt, least privilege) | `05_NonFunctional_Requirements.md` |

---

## 4. Zakres projektu

### 4.1 W zakresie (In Scope)

- Automatyczne otwieranie i sprawdzanie dostępności/renderowania raportów Power BI i SSRS.
- Odczyt wskazanych wartości KPI z raportów oraz porównanie ich z wynikami zapytań SQL do hurtowni danych.
- Definiowanie, wersjonowanie i wykonywanie testów biznesowych (Validation Profiles) przez administratora/analityka.
- Harmonogramowanie testów (cykliczne i na żądanie).
- Przechowywanie historii wyników testów wraz z metrykami (Health Score, Confidence Score).
- Moduł AI wspierający interpretację wyników, wykrywanie anomalii i generowanie czytelnych podsumowań.
- Aplikacja webowa (GUI) do przeglądu raportów, konfiguracji, historii i wyników.
- Powiadomienia o wykrytych problemach (np. e-mail, integracja z kanałem powiadomień organizacji).
- Mechanizmy bezpieczeństwa, audytu i monitoringu zgodne ze standardami środowiska bankowego.

### 4.2 Poza zakresem (Out of Scope) — w bieżącej fazie

- Automatyczne podejmowanie decyzji biznesowych na podstawie wyników testów (decyzje pozostają po stronie ludzi, wspieranych przez AI — patrz `06_AI_Module.md`).
- Integracja z platformami raportowymi innymi niż Power BI i SSRS w fazie MVP (przewidziana architektonicznie, ale poza zakresem pierwszego wdrożenia — patrz `11_Roadmap.md`).

---

## 5. Interesariusze (Stakeholders)

| Rola | Odpowiedzialność w projekcie | Kluczowe zainteresowania |
|---|---|---|
| **Sponsor biznesowy** | Zatwierdzenie budżetu i priorytetów | ROI, redukcja ryzyka operacyjnego |
| **Project Manager** | Harmonogram, zasoby, komunikacja | Terminowość, zakres, ryzyka |
| **Architekt IT / Enterprise Architecture** | Zgodność z architekturą korporacyjną | Standardy, integracje, dług technologiczny |
| **Zespół Bezpieczeństwa / Security Office** | Ocena ryzyka bezpieczeństwa | Zarządzanie kontem technicznym, RBAC, audyt, dane wrażliwe |
| **Administrator Platformy BI** | Udostępnienie dostępów, wiedza o środowisku PBI/SSRS | Stabilność platformy, obciążenie generowane przez testy |
| **Zespół Deweloperski** | Implementacja | Jakość dokumentacji wejściowej, jasność wymagań |
| **Analitycy biznesowi / Właściciele raportów** | Definiowanie testów i reguł biznesowych, konsumenci wyników | Trafność testów, minimalizacja fałszywych alarmów |
| **Data Governance / Compliance** | Nadzór nad jakością i audytowalnością danych | Ślad audytowy, retencja danych, zgodność regulacyjna |

---

## 6. Kluczowe założenia projektowe

1. System działa **wewnątrz infrastruktury organizacji** (on-prem), z dedykowanym kontem technicznym mającym dostęp do Platformy BI oraz hurtowni danych — brak zależności od usług chmurowych publicznych jako wymogu obowiązkowego (opcjonalna integracja z Azure OpenAI jako usługą regulowaną/kontraktową — patrz `06_AI_Module.md`).
2. Autoryzacja użytkowników do samych raportów pozostaje bez zmian (SSO/Windows Authentication) — ARGUS **nie zastępuje** i nie modyfikuje mechanizmu autoryzacji Platformy BI; korzysta z własnego konta technicznego o kontrolowanym, najniższym niezbędnym zakresie uprawnień.
3. Docelowa skala — **1000 raportów** — jest wymaganiem projektowym od dnia pierwszego, nawet jeśli faza MVP obejmuje mniejszy podzbiór (patrz `11_Roadmap.md`). Architektura musi być zaprojektowana pod tę skalę, a nie dostosowywana post factum.
4. Administrator/analityk biznesowy **musi** zdefiniować test dla raportu, aby ten był objęty monitoringiem — ARGUS nie zgaduje reguł biznesowych automatycznie (poza wspierającą rolą AI w sugerowaniu konfiguracji, patrz `06_AI_Module.md`).
5. AI pełni rolę **wyłącznie wspierającą** — nie podejmuje decyzji o poprawności raportu (zasada nadrzędna projektu — patrz `06_AI_Module.md`).

## 7. Ograniczenia projektu

- Zależność od stabilności i dostępności API/mechanizmów udostępnianych przez Power BI i SSRS — zmiany po stronie Microsoft (aktualizacje produktu) mogą wymagać adaptacji connectorów (mitygowane architekturą modułową — patrz `03_Architecture.md`).
- Zależność od jakości i kompletności definicji testów dostarczanych przez administratorów — system jest tak dobry, jak zdefiniowane Validation Profiles.
- Środowisko bankowe wprowadza dodatkowe wymogi bezpieczeństwa i proceduralne (zarządzanie zmianą, testy penetracyjne, akceptacje Security Office), które wydłużają cykl wdrożenia względem środowisk nieregulowanych.
- Skala docelowa (1000 raportów) wymaga inwestycji w infrastrukturę wykonawczą (worker pool przeglądarek) — koszt operacyjny musi być uwzględniony w planowaniu budżetu (patrz `10_Deployment.md`).

---

## 8. Słownik pojęć (Glossary)

| Pojęcie | Definicja |
|---|---|
| **Validation Profile** | Zestaw konfiguracji dla pojedynczego raportu: adres, typ platformy, testy, sposób odnajdywania KPI, zapytania SQL, dopuszczalne odchylenia, reguły biznesowe, harmonogram. |
| **Connector** | Moduł implementujący integrację z konkretną platformą raportową (Power BI, SSRS, w przyszłości inne), zgodny ze wspólnym interfejsem `IReportConnector`. |
| **Value Extraction Engine** | Moduł odpowiedzialny za odczyt wartości KPI z wyrenderowanego raportu, działający w oparciu o strategię wielowarstwowego fallbacku. |
| **Health Score** | Zagregowany, liczbowy wskaźnik "zdrowia" raportu w danym momencie/okresie, wyliczany na podstawie wyników testów. |
| **Confidence Score** | Wskaźnik pewności pojedynczego odczytu/testu — informuje, jak bardzo system "ufa" danemu wynikowi (np. z uwagi na warstwę fallbacku, która go dostarczyła). |
| **Test Orchestrator** | Moduł koordynujący wykonanie zestawu testów dla raportu/grupy raportów. |
| **Business Rule** | Reguła biznesowa wykraczająca poza proste porównanie liczby (np. "suma kategorii = wartość całkowita", "wartość nie może być ujemna"). |
| **Obszar biznesowy (zakładka)** | Grupa raportów w Platformie BI odpowiadająca jednostce/tematyce biznesowej (np. Sprzedaż, Kredyty, Ryzyko). |
| **AI Analysis Engine** | Moduł AI interpretujący wyniki testów — wykrywanie anomalii, klastrowanie błędów, generowanie wyjaśnień; nie podejmuje decyzji o poprawności. |

## 9. Kryteria sukcesu projektu

| Kryterium | Docelowa wartość |
|---|---|
| Pokrycie katalogu raportów testami | 100% raportów posiada zdefiniowany Validation Profile w perspektywie 12 miesięcy od startu produkcyjnego |
| Czas pełnego cyklu testowego dla całego katalogu | ≤ 4 godziny dla 1000 raportów (harmonogram nocny/wsadowy — patrz `05_NonFunctional_Requirements.md`) |
| Dostępność systemu (SLA) | ≥ 99.5% w godzinach pracy organizacji |
| Redukcja czasu manualnej weryfikacji | ≥ 80% w stosunku do stanu obecnego |
| Trafność detekcji (niska liczba fałszywych alarmów) | Utrzymanie wskaźnika fałszywych alarmów na poziomie akceptowalnym przez biznes, mierzone i raportowane cyklicznie |

Powiązanie z uzasadnieniem biznesowym — patrz `02_Business_Problem.md`. Plan realizacji w fazach — patrz `11_Roadmap.md`.
