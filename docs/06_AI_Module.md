# 06. AI Module

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform \
**Dokument:** Rola, granice i architektura modułu AI \
**Odbiorcy:** Architektura IT, Security Office, Compliance, Zespół Deweloperski

---

## 1. Zasada nadrzędna

> **AI nie podejmuje decyzji o poprawności raportu. Za poprawność odpowiada wyłącznie deterministyczny silnik walidacji (Validation Engine) oparty na jawnie zdefiniowanych testach.**

Jest to najważniejsza zasada projektowa dotycząca modułu AI i jest traktowana jako **twardy wymóg architektoniczny (guardrail)**, weryfikowany technicznie (rozdzielenie odpowiedzialności w kodzie — AI Analysis Engine nie ma technicznej możliwości zapisu statusu PASS/FAIL) oraz proceduralnie (testy akceptacyjne, przegląd bezpieczeństwa).

Powód tej zasady jest fundamentalny dla kontekstu bankowego: modele językowe mogą halucynować, są niedeterministyczne i nie dają twardych gwarancji poprawności. Powierzenie im ostatecznej decyzji o poprawności danych finansowych byłoby nieakceptowalnym ryzykiem regulacyjnym i operacyjnym. AI dostarcza **wartości interpretacyjnej i przyspieszającej pracę człowieka**, nigdy nie zastępuje kontrolowanego, testowalnego procesu walidacji.

## 2. Rola AI — co AI **może** robić

| Funkcja | Opis |
|---|---|
| **Interpretacja wyników testów** | Tłumaczenie technicznego wyniku Validation Engine (np. "KPI_SPRZEDAZ_TOTAL: raport=1 204 500,00, SQL=1 189 300,00, odchylenie=1.28%, próg=1.0%") na czytelne, biznesowe wyjaśnienie w języku naturalnym. |
| **Wykrywanie anomalii na tle historii** | Analiza trendu Health Score/wyników w czasie w celu wskazania nietypowych wzorców (np. "wartość KPI dla tego raportu odchyla się drugi dzień z rzędu, mimo że mieści się formalnie w tolerancji — warto zwrócić uwagę"). |
| **Wskazywanie najbardziej prawdopodobnych przyczyn** | Na podstawie kontekstu (typ błędu, historia, powiązane raporty w tym samym obszarze/źródle danych) AI proponuje hipotezy przyczyny (np. "opóźnienie odświeżenia datasetu", "zmiana definicji miary", "problem z hurtownią danych widoczny również w innych raportach obszaru Kredyty"). |
| **Grupowanie podobnych błędów** | Klastrowanie wielu jednostkowych alertów w jeden zrozumiały wzorzec (np. "12 raportów w obszarze Sprzedaż wykazuje ten sam wzorzec błędu — prawdopodobna wspólna przyczyna źródłowa"), redukując szum informacyjny dla analityka. |
| **Generowanie czytelnych raportów dla użytkowników** | Streszczenia okresowe (np. tygodniowe podsumowanie stanu jakości katalogu raportów) w języku naturalnym, na bazie zagregowanych, twardych danych z Results Repository. |
| **Proponowanie działań naprawczych** | Sugestie kolejnych kroków dla analityka/administratora (np. "sprawdź harmonogram odświeżania datasetu X", "porównaj z raportem Y, który wykazuje tę samą anomalię") — sugestie, nie polecenia wykonywane automatycznie. |
| **Wspieranie konfiguracji (opcjonalnie, Could-have)** | Propozycja wstępnej konfiguracji Validation Profile na podstawie analizy struktury raportu — zawsze wymaga świadomej akceptacji przez człowieka przed aktywacją (patrz `04_Functional_Requirements.md`, FR-019). |

## 3. Czego AI **nie może** robić

1. **Nie ustala statusu PASS/FAIL/WARNING** — ten status jest wyłączną kompetencją Validation Engine, obliczaną deterministycznie na podstawie zdefiniowanych testów i progów tolerancji.
2. **Nie modyfikuje wyników testów ani danych źródłowych.**
3. **Nie uruchamia automatycznie działań naprawczych** bez udziału i akceptacji człowieka (brak trybu "autonomicznej naprawy").
4. **Nie zastępuje testu SQL ani reguły biznesowej** — nie "ocenia" poprawności liczby na podstawie własnej wiedzy ogólnej, wyłącznie interpretuje wynik już wyliczony deterministycznie.
5. **Nie ma dostępu zapisu** do Validation Profile, Results Repository czy statusu raportu — z perspektywy uprawnień systemowych AI Analysis Engine posiada **wyłącznie prawo odczytu** danych historycznych/wyników i prawo zapisu do odrębnej, oznaczonej przestrzeni "AI Insights" (nigdy do encji wyniku testu).

## 4. Wymóg wyjaśnialności (Explainability)

**Każda wygenerowana przez AI interpretacja musi zawierać jawne uzasadnienie odwołujące się do konkretnych, sprawdzalnych danych** (numer testu, wartości porównywane, historia, powiązane raporty) — nigdy ogólnikowe stwierdzenie bez podstawy.

Realizacja techniczna:
- Prompt systemowy AI Analysis Engine wymusza strukturalną odpowiedź zawierającą sekcję `evidence` (dane źródłowe, na których oparto wyjaśnienie) obok sekcji `explanation` (treść dla użytkownika) — odpowiedź bez `evidence` jest odrzucana przez warstwę walidacji odpowiedzi AI przed wyświetleniem użytkownikowi.
- Każda interpretacja AI jest logowana wraz z pełnym kontekstem wejściowym (dane wejściowe przekazane do modelu, wersja promptu, wersja modelu) w celu audytu i możliwości odtworzenia/wyjaśnienia decyzji post factum (NFR-C-04 w `05_NonFunctional_Requirements.md`).
- GUI jednoznacznie oznacza treści AI wizualnie (odrębny styl/etykieta "Wygenerowane przez AI — treść wspierająca") — patrz `04_Functional_Requirements.md`, NFR-U-02.

## 5. Architektura modułu AI

### 5.1 Umiejscowienie w architekturze

`AI Analysis Engine` działa jako usługa **downstream** względem `Validation Engine` i `Results Repository` — konsumuje już zakończone, deterministyczne wyniki testów (nigdy nie jest wywoływany w ścieżce krytycznej samej walidacji, co gwarantuje, że nawet całkowita niedostępność usługi AI **nie wpływa** na zdolność systemu do wykonywania i raportowania testów — jedynie na dostępność warstwy interpretacyjnej).

```
Validation Engine ──▶ Results Repository ──▶ AI Analysis Engine ──▶ AI Insights Store ──▶ Dashboard
                                                     │
                                                     ▼
                                     (kontekst: historia, dane porównawcze,
                                      metadane raportu — READ ONLY)
```

### 5.2 Komponenty wewnętrzne AI Analysis Engine

| Komponent | Odpowiedzialność |
|---|---|
| **Context Builder** | Zbiera i przygotowuje kontekst dla zapytania do modelu: wynik bieżącego testu, historia N ostatnich przebiegów, dane powiązanych raportów tego samego obszaru/źródła — wyłącznie dane niewrażliwe (patrz sekcja 6). |
| **Anomaly Detector** | Komponent statystyczny/regułowy (niekoniecznie LLM) wykrywający odchylenia od wzorca historycznego — dostarcza sygnał wejściowy dla warstwy LLM, a także może działać niezależnie jako prosty detektor trendu. |
| **LLM Orchestration Layer** | Warstwa komunikacji z modelem językowym (Azure OpenAI / OpenAI API), zarządzanie promptami, wymuszanie strukturalnej odpowiedzi (np. schema JSON z polami `explanation`, `evidence`, `probable_causes`, `suggested_actions`, `confidence`). |
| **Response Validator** | Waliduje odpowiedź modelu pod kątem zgodności ze schematem i obecności `evidence` przed zapisem/prezentacją — odrzuca i loguje odpowiedzi niespełniające wymogu wyjaśnialności. |
| **Clustering Engine** | Grupowanie podobnych błędów (może wykorzystywać embeddings + podobieństwo wektorowe lub reguły oparte na metadanych — decyzja implementacyjna do doprecyzowania w Fazie 2, patrz `11_Roadmap.md`). |
| **AI Insights Store** | Odrębna tabela/przestrzeń przechowująca wygenerowane interpretacje, powiązana z wynikiem testu, ale niewpływająca na jego status (patrz `07_Data_Model.md`). |

### 5.3 Wybór dostawcy modelu

Zgodnie z zaleceniem projektowym: **LangChain** (lub równoważny framework orkiestracji, np. lekki, autorski wrapper — decyzja w Fazie 0 zależna od faktycznej złożoności potrzebnych łańcuchów) wykorzystywany **wyłącznie w module AI**, oraz **Azure OpenAI** jako rekomendowany dostawca modelu w kontekście bankowym (dzierżawa Azure w ramach umowy korporacyjnej, kontraktowe gwarancje nieużywania danych do trenowania, rezydencja danych w UE, zgodność z politykami bezpieczeństwa organizacji) — alternatywnie hostowany model open-weight w infrastrukturze wewnętrznej, jeśli polityka Security Office wykluczy jakikolwiek ruch do usług chmurowych nawet w modelu kontraktowo kontrolowanym (do potwierdzenia w Fazie 0, patrz `00_Technical_Feasibility_Study.md`, sekcja 8, oraz `12_Risks.md`).

## 6. Dane wykorzystywane przez AI i ochrona danych wrażliwych

- AI Analysis Engine otrzymuje **wyłącznie zagregowane wyniki testów i metadane** (nazwy KPI, wartości liczbowe, statusy, historię, nazwy raportów/obszarów biznesowych) — **nigdy surowych danych klienckich** ani danych sklasyfikowanych jako wrażliwe/osobowe, zgodnie z zasadą minimalizacji danych.
- Definicje testów (zapytania SQL) mogą być przekazywane w ograniczonym zakresie wyłącznie jako metadane strukturalne (np. nazwa testowanej miary), nie jako pełna treść zapytania, jeśli zawiera ona elementy wrażliwe — zasada do doprecyzowania z Data Governance w Fazie 0.
- W przypadku wykorzystania zewnętrznego dostawcy modelu obowiązuje wymóg NFR-SEC-07 (`05_NonFunctional_Requirements.md`) — kontraktowa gwarancja nieużywania danych do trenowania, przetwarzanie w regionie zgodnym z polityką rezydencji danych.

## 7. Ryzyka związane z AI i mitygacje

| Ryzyko | Mitygacja |
|---|---|
| Halucynacja modelu (wymyślone "fakty" nieoparte na danych) | Wymóg strukturalnej odpowiedzi z obowiązkowym polem `evidence`, walidacja odpowiedzi przed prezentacją (Response Validator), jednoznaczne oznaczenie treści jako wspierającej. |
| Nadmierne zaufanie użytkowników do sugestii AI ("automation bias") | Wyraźne oznaczenie wizualne treści AI jako odrębnej od werdyktu deterministycznego (NFR-U-02); szkolenie użytkowników; możliwość oznaczenia sugestii jako nietrafnej (feedback loop). |
| Wyciek danych wrażliwych do zewnętrznego dostawcy modelu | Minimalizacja danych przekazywanych do modelu (sekcja 6), wybór dostawcy zgodnego z polityką organizacji, możliwość wariantu w pełni on-prem. |
| Niedostępność usługi AI (zewnętrzne API) | AI Analysis Engine jest komponentem non-blocking — niedostępność nie wpływa na wykonanie i raportowanie testów (sekcja 5.1); degradacja funkcjonalna ograniczona do braku interpretacji. |
| Koszt operacyjny wywołań modelu przy skali 1000 raportów | Wywoływanie AI selektywnie — wyłącznie dla przypadków FAIL/WARNING lub wykrytych anomalii, nie dla każdego pojedynczego zielonego wyniku (optymalizacja kosztowa opisana w `09_Python_Architecture.md`). |

## 8. Powiązanie z dalszą dokumentacją

- Model danych dla `AI Insights` — `07_Data_Model.md`.
- Endpointy API do pobierania interpretacji AI — `08_API.md`.
- Wybór bibliotek (LangChain, klient Azure OpenAI) — `09_Python_Architecture.md`.
- Ryzyka projektowe związane z AI jako pozycja w rejestrze ryzyk — `12_Risks.md`.
