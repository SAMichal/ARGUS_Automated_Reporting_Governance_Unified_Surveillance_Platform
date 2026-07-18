# 00. Technical Feasibility Study
## Analiza techniczna metod integracji z platformą BI oraz metod ekstrakcji danych z raportów

**Projekt:** ARGUS — Automated Reporting Governance & Unified Surveillance Platform \
**Dokument:** Studium wykonalności technicznej \
**Odbiorcy:** Architektura IT, Zespół Bezpieczeństwa, Zespół Deweloperski \
**Status:** Do akceptacji

---

## 1. Cel dokumentu

Dokument odpowiada na dwa fundamentalne pytania techniczne, od których zależy cała dalsza architektura systemu:

1. **W jaki sposób ARGUS ma technicznie "dotrzeć" do raportu** opublikowanego na Platformie BI (Power BI oraz SSRS/Report Server), tak aby móc stwierdzić, czy raport się otwiera i renderuje poprawnie?
2. **W jaki sposób ARGUS ma odczytać konkretną wartość liczbową (KPI)** z wyrenderowanego raportu, aby porównać ją z wynikiem zapytania SQL do hurtowni danych?

Odpowiedzi na te pytania nie są oczywiste, ponieważ Power BI i SSRS to dwie różne technologie, o różnym stopniu otwartości API, różnym modelu autoryzacji i różnej charakterystyce renderowania (Power BI — SPA renderowany po stronie klienta w iframe/JS; SSRS — częściowo server-side HTML/ReportViewer). Dokument analizuje wszystkie realistyczne podejścia integracyjne, ocenia je względem jednolitego zestawu kryteriów i rekomenduje architekturę docelową.

## 2. Metodologia oceny

Każde podejście oceniane jest według siedmiu kryteriów, spójnych z wymaganiami z briefu projektowego:

| Kryterium | Pytanie oceniające |
|---|---|
| Wymagania | Czego technicznie/organizacyjnie potrzeba, aby wdrożyć podejście? |
| Zalety | Co zyskujemy? |
| Ograniczenia | Czego podejście nie potrafi lub gdzie zawodzi? |
| Wpływ na wydajność | Jaki jest koszt czasowy/zasobowy per raport? |
| Odporność na zmiany raportów | Czy drobna zmiana layoutu raportu psuje integrację? |
| Zastosowanie w środowisku bankowym | Czy podejście jest akceptowalne pod kątem bezpieczeństwa, sieci, compliance, licencji? |
| Złożoność implementacji / utrzymania | Ile kosztuje zbudowanie i utrzymanie w perspektywie 1000 raportów? |

Ocena końcowa każdego podejścia wyrażona jest w skali: 
* 🟢 Wysoka przydatność 
* 🟡 Przydatność ograniczona
* 🔴 Niska przydatność
* ⚫ Nie dotyczy (n/d) dla danej platformy.

---

## 3. Część I — Metody integracji z platformą raportową (dostęp do raportu, sprawdzenie dostępności i renderowania)

### 3.1 Browser Automation (Playwright)

**Opis:** Sterowanie prawdziwą przeglądarką (Chromium/WebKit/Firefox) w trybie headless lub headed, nawigacja do adresu URL raportu, oczekiwanie na zdarzenia sieciowe/DOM sygnalizujące zakończenie renderowania, przechwytywanie stanu strony (DOM, zrzut ekranu, żądania sieciowe).

| Kryterium | Ocena |
|---|---|
| Wymagania | Środowisko z zainstalowanymi silnikami przeglądarek, konto techniczne z dostępem SSO (Windows Integrated Authentication / Kerberos), dostęp sieciowy do `pbi.bank.com.pl` i `bgkssrs`, ewentualnie profil systemu Windows/Linux z obsługą NTLM/Kerberos (SPNEGO). |
| Zalety | **Działa identycznie jak realny użytkownik** — jest niezależne od tego, czy dany raport ma API. Obsługuje SSO tak samo jak przeglądarka pracownika. Pozwala zweryfikować pełny łańcuch: sieć → autoryzacja → renderowanie → interaktywność (np. slicery, filtry). Jedno podejście dla **obu** platform (PBI i SSRS) — ujednolica architekturę. Pozwala też wykonać zrzut ekranu jako dowód (evidence) w audycie. |
| Ograniczenia | Wyższe zużycie CPU/RAM niż podejścia API (pełny silnik przeglądarki na raport). Renderowanie Power BI jest asynchroniczne i oparte o WebSocket/JS — wymaga jawnych strategii oczekiwania (nie da się polegać wyłącznie na `networkidle`). Selektory DOM mogą się zmieniać wraz z aktualizacjami produktu Microsoft (mitygowalne, patrz sekcja 5). |
| Wpływ na wydajność | Otwarcie pojedynczego raportu Power BI: ok. 3–8 s (zależnie od złożoności wizualizacji i cache Power BI). SSRS: zwykle szybciej (1–4 s), bo renderowanie jest w większym stopniu server-side. Przy 1000 raportach wymaga **równoległości** (worker pool), a nie wykonania sekwencyjnego — patrz `05_NonFunctional_Requirements.md`. |
| Odporność na zmiany | Średnia–wysoka, pod warunkiem stosowania odpornych strategii selekcji (role/aria-label, atrybuty semantyczne Power BI, a nie pozycja pikselowa czy klasy CSS generowane przez bundler). |
| Zastosowanie bankowe | Wysokie — Playwright jest open-source (Apache 2.0), nie wysyła telemetrii do zewnętrznych usług przy odpowiedniej konfiguracji, może działać w pełni offline w sieci wewnętrznej banku, wspiera autoryzację Windows (NTLM/Kerberos) natywnie przez kontekst przeglądarki uruchomiony na koncie domenowym / przekazanie tokenu. |
| Złożoność / utrzymanie | Średnia. Wymaga zespołu, który rozumie zarówno automatyzację przeglądarki, jak i specyfikę renderowania Power BI/SSRS. Utrzymanie selektorów jest powtarzalnym kosztem, ale mitygowalnym przez centralizację (Page Object Pattern) i strategię wielopoziomowych selektorów. |
| **Ocena łączna** | 🟢 **Wysoka — jedyne podejście uniwersalne dla obu platform** |

### 3.2 Selenium (alternatywa dla Playwright)

**Opis:** Klasyczny framework do automatyzacji przeglądarki, sterujący przez protokół WebDriver.

| Kryterium | Ocena |
|---|---|
| Wymagania | Analogiczne do Playwright — silniki przeglądarek + sterowniki (ChromeDriver/GeckoDriver), zarządzane osobno od wersji przeglądarki. |
| Zalety | Bardzo dojrzały, ogromna baza wiedzy, wsparcie w praktycznie każdym języku (w tym Python — `selenium`). Szerokie wsparcie w narzędziach RPA korporacyjnych, co może ułatwić integrację z istniejącymi standardami banku, jeśli takie już funkcjonują. |
| Ograniczenia | Wolniejszy i mniej stabilny niż Playwright w scenariuszach z dynamicznym JS (Power BI = ciężki SPA). Brak wbudowanego auto-waitingu na poziomie akcji (częstsze `flaky tests`). Zarządzanie wersjami sterowników (`*Driver` binaries) to dodatkowy koszt operacyjny (mitygowalne przez Selenium Manager / Selenium Grid, ale nadal bardziej złożone niż w Playwright). Brak natywnego przechwytywania ruchu sieciowego (wymaga dodatkowych proxy typu BrowserMob). |
| Wpływ na wydajność | Porównywalny rząd wielkości do Playwright, ale zazwyczaj 20–40% wolniejszy w testach z dużą liczbą interakcji z powodu mniej efektywnego protokołu komunikacji (WebDriver classic vs. CDP/BiDi używany przez Playwright). |
| Odporność na zmiany | Podobna do Playwright — zależy od jakości strategii selektorów, nie od samego narzędzia. |
| Zastosowanie bankowe | Wysokie, tak samo dopuszczalne licencyjnie (Apache 2.0) i sieciowo jak Playwright. |
| Złożoność / utrzymanie | Wyższa niż Playwright — więcej "ruchomych części" (menedżer sterowników, Selenium Grid do równoległości, mniej ergonomiczne API asynchroniczne). |
| **Ocena łączna** | 🟡 **Przydatność ograniczona — Playwright dominuje Selenium w praktycznie każdym kryterium dla tego zastosowania (nowoczesny stack, natywna równoległość, natywne auto-waiting, natywne przechwytywanie sieci potrzebne do detekcji błędów renderowania)** |

### 3.3 Power BI REST API

**Opis:** Oficjalne REST API Microsoftu (`api.powerbi.com` w chmurze lub Power BI Report Server REST API on-prem) umożliwiające operacje na poziomie metadanych: listy raportów, workspace'y, odświeżanie datasetów, eksport raportu do pliku, sprawdzanie statusu odświeżenia.

| Kryterium | Ocena |
|---|---|
| Wymagania | Rejestracja aplikacji w Azure AD / Entra ID (service principal) **lub** — w wariancie Power BI Report Server (on-prem, częstszy w bankach) — API oparte o inny mechanizm autoryzacji (Windows Auth). Wymaga potwierdzenia, czy organizacja korzysta z Power BI Service (chmura) czy Power BI Report Server (on-prem) — adres `pbi.bank.com.pl` sugeruje **wdrożenie on-prem / self-hosted**, co **ogranicza dostępność części funkcji REST API dostępnych wyłącznie w wariancie chmurowym** (np. część endpointów Admin API). |
| Zalety | Brak potrzeby renderowania w przeglądarce dla operacji administracyjnych (np. sprawdzenie statusu ostatniego odświeżenia datasetu, listy raportów w workspace). Bardzo szybkie (odpowiedzi JSON, brak renderowania UI). Dobrze udokumentowane, stabilne kontraktowo (wersjonowane API). |
| Ograniczenia | **REST API nie pozwala odczytać wartości z wizualizacji raportu** (nie zwraca "co widzi użytkownik na wykresie") — to kluczowe ograniczenie dla wymogu odczytu KPI. Służy głównie do zarządzania (metadata, refresh, eksport). Dostępność i zakres endpointów różni się istotnie między Power BI Service a Power BI Report Server (on-prem) — wymaga wczesnej weryfikacji z zespołem administrującym platformą BI. |
| Wpływ na wydajność | Bardzo dobry — wywołania rzędu 100–300 ms. |
| Odporność na zmiany | Wysoka dla operacji, które obsługuje (kontrakt API), ale nie dotyczy odczytu KPI. |
| Zastosowanie bankowe | Zależne od modelu wdrożenia Power BI (cloud vs. on-prem) i polityk sieciowych dot. wywołań do `api.powerbi.com` (chmura Microsoft) — może wymagać dodatkowej zgody Security/Compliance na ruch wychodzący do chmury publicznej, co w bankowości bywa procesem długotrwałym. |
| Złożoność / utrzymanie | Niska dla zakresu, który obsługuje. |
| **Ocena łączna** | 🟡 **Przydatność ograniczona — cenne wsparcie dla operacji administracyjnych i triggerowania odświeżeń, ale niewystarczające jako główny mechanizm odczytu dostępności i wartości KPI** |

### 3.4 XMLA Endpoint (Power BI Premium/Premium-per-User)

**Opis:** Endpoint analityczny (protokół XML for Analysis, jak w SSAS Tabular) pozwalający wykonywać zapytania **DAX/MDX bezpośrednio na modelu danych** leżącym pod raportem Power BI, z pominięciem warstwy wizualnej.

| Kryterium | Ocena |
|---|---|
| Wymagania | Power BI Premium, Premium-per-User lub Fabric capacity (XMLA Endpoint **nie jest dostępny w licencji Pro** ani standardowo w Power BI Report Server on-prem w tej samej formie co w chmurze) — kluczowe pytanie licencyjne do wyjaśnienia z administratorem platformy BI na starcie projektu. Narzędzie klienckie obsługujące XMLA/DAX (np. biblioteka `Microsoft.AnalysisServices.AdomdClient` po stronie .NET; z poziomu Pythona — komunikacja przez ADOMD.NET via `pythonnet` lub pośredni serwis .NET). |
| Zalety | **Najbardziej precyzyjna metoda odczytu wartości** — zapytanie DAX zwraca dokładnie tę samą liczbę, którą silnik Power BI wyliczyłby dla wizualizacji, bez zależności od warstwy graficznej. Bardzo szybkie. Odporne na zmiany layoutu raportu (bo nie dotyka warstwy wizualnej), wrażliwe tylko na zmiany modelu danych. |
| Ograniczenia | Wymaga wiedzy o strukturze modelu (nazwy miar/tabel) dla każdego raportu — administrator definiujący Validation Profile musi znać definicję miary DAX, nie tylko "co widać na ekranie". Nie weryfikuje **renderowania** (czy wizualizacja faktycznie się wyświetla poprawnie użytkownikowi) — nie zastępuje testu dostępności/UI, jedynie uzupełnia test wartości. Zależność licencyjna (Premium) może wykluczać część raportów. |
| Wpływ na wydajność | Bardzo dobry (dziesiątki–setki ms na zapytanie). |
| Odporność na zmiany | Bardzo wysoka wobec zmian UI; niska wobec zmian modelu (przemianowanie miary łamie zapytanie — ale to zmiana rzadka i celowa, łatwa do zarządzania proceduralnie). |
| Zastosowanie bankowe | Wysokie pod względem bezpieczeństwa (ruch wewnętrzny, ten sam mechanizm auth co reszta platformy), warunkowe pod względem licencji. |
| Złożoność / utrzymanie | Średnia–wysoka z Pythona (naturalny język docelowy projektu) ze względu na brak natywnej biblioteki Python dla XMLA/ADOMD — zwykle wymaga mostka .NET/pythonnet lub osobnego mikroserwisu. |
| **Ocena łączna** | 🟡 **Przydatność ograniczona do podzbioru raportów (Premium) — rekomendowana jako opcjonalny, wysoko-precyzyjny connector "drugiej linii" dla raportów, dla których jest dostępna, nie jako rozwiązanie uniwersalne** |

### 3.5 SSRS Web Services (SOAP) i SSRS REST API (Report Server 2017+)

**Opis:** SSRS udostępnia klasyczne API SOAP (`ReportService2010.asmx`) do zarządzania katalogiem raportów oraz — od wersji 2017 — REST API (`api/v2.0`) do operacji na katalogu, a także URL Access (parametryzowany link renderujący raport w wybranym formacie: HTML, PDF, XML, CSV, Excel) poprzez `ReportExecution2005` / `/Reserve.aspx`.

| Kryterium | Ocena |
|---|---|
| Wymagania | Dostęp sieciowy do endpointów SOAP/REST Report Servera, autoryzacja Windows Integrated (NTLM/Kerberos) po stronie klienta HTTP — w Pythonie realizowalne (`requests` + `requests-negotiate-sspi` / `requests-kerberos`, lub `zeep` dla SOAP). |
| Zalety | Pozwala **programowo wygenerować raport w formacie danych** (np. XML lub CSV) zamiast HTML — co daje **strukturalny, łatwy do sparsowania wynik** bez potrzeby przeglądarki. Możliwość pobrania metadanych katalogu (lista raportów, parametry, harmonogramy) — przydatne dla automatycznej inwentaryzacji raportów w systemie ARGUS. Brak zależności od silnika przeglądarki dla tej ścieżki integracji. |
| Ograniczenia | URL Access renderuje **cały raport na nowo w wybranym formacie eksportu** — dla dużych raportów może być wolniejsze niż odczyt jednej wartości. Nie weryfikuje **wizualnego** renderowania w sensie "czy webowy interfejs użytkownika faktycznie poprawnie wyświetla stronę" (np. błędy CSS, błędy w kontrolce ReportViewer na stronie portalu, timeout w warstwie webowej niewidoczny z poziomu samego silnika raportowego) — jest to uzupełnienie, nie substytut testu end-to-end. SOAP API bywa postrzegane jako starzejące się (choć w środowiskach SSRS on-prem nadal standardowe i w pełni wspierane). |
| Wpływ na wydajność | Dobry dla metadanych (SOAP `ListChildren` itp. — setki ms); zmienny dla renderowania pełnego raportu w formacie eksportu (zależny od rozmiaru raportu, od setek ms do kilku–kilkunastu sekund dla dużych raportów tabelarycznych). |
| Odporność na zmiany | Wysoka dla warstwy API/katalogu; średnia dla parsowania wyeksportowanego formatu (zmiana struktury raportu = zmiana układu kolumn w eksporcie). |
| Zastosowanie bankowe | Wysokie — mechanizm natywny dla SSRS on-prem, używany od lat w środowiskach korporacyjnych, w pełni kompatybilny z Windows Integrated Authentication. |
| Złożoność / utrzymanie | Niska–średnia. |
| **Ocena łączna** | 🟢 **Wysoka dla SSRS — rekomendowana jako natywny connector dla platformy SSRS, komplementarny do Browser Automation** |

### 3.6 Eksport raportów (Export-to-File: PDF / Excel / CSV / XML)

**Opis:** Zarówno Power BI, jak i SSRS oferują mechanizm eksportu raportu do pliku (poprzez UI, API lub URL Access). Eksport CSV/XML jest ustrukturyzowany i łatwy do sparsowania programowo; PDF/Excel wymaga dodatkowego parsowania.

| Kryterium | Ocena |
|---|---|
| Wymagania | Dla SSRS — URL Access z parametrem `rs:Format`; dla Power BI — Power BI REST API `Export to File` (asynchroniczne, wymaga odpytywania statusu zadania) lub eksport z poziomu UI przez Browser Automation. |
| Zalety | Format danych (CSV/XML) eliminuje potrzebę parsowania HTML/DOM — bardzo wysoka niezawodność odczytu wartości, gdy dostępny. Dobry "dowód" (artefakt) do celów audytu — plik można zarchiwizować. |
| Ograniczenia | Nie jest to test end-to-end doświadczenia użytkownika (nie wykrywa np. błędu renderowania konkretnej wizualizacji w warstwie webowej, błędu JS, timeoutu interfejsu). Eksport bywa wolniejszy niż odczyt pojedynczej wartości (generowany jest cały raport). Dla Power BI eksport przez REST API jest operacją asynchroniczną z limitami (rate limiting), co przy 1000 raportach wymaga starannego zarządzania kolejką. |
| Wpływ na wydajność | Średni–wysoki koszt czasowy per raport (generowanie pełnego dokumentu), ale wykonywalny w tle/asynchronicznie. |
| Odporność na zmiany | Wysoka dla formatów strukturalnych (CSV/XML), niska dla PDF (parsowanie PDF jest kruche wobec zmian layoutu). |
| Zastosowanie bankowe | Wysokie — mechanizm natywny, brak dodatkowych zależności bezpieczeństwa. |
| Złożoność / utrzymanie | Niska–średnia. |
| **Ocena łączna** | 🟢 **Wysoka jako uzupełniająca metoda ekstrakcji wartości (patrz Część II), ograniczona jako jedyna metoda testu dostępności/renderowania** |

### 3.7 Analiza statyczna HTML/DOM (bez pełnego silnika przeglądarki, np. `requests` + `BeautifulSoup`)

**Opis:** Pobranie odpowiedzi HTTP strony raportu bez wykonywania JavaScript i sparsowanie surowego HTML.

| Kryterium | Ocena |
|---|---|
| Wymagania | Biblioteka HTTP z obsługą Windows Auth (Kerberos/NTLM), parser HTML (`BeautifulSoup`, `lxml`). |
| Zalety | Bardzo szybkie i lekkie zasobowo (brak silnika przeglądarki). |
| Ograniczenia | **Nieadekwatne dla Power BI** — raporty Power BI to Single Page Application renderowany w całości przez JavaScript po stronie klienta; surowa odpowiedź HTTP nie zawiera wartości KPI (tylko szkielet aplikacji JS). Częściowo adekwatne dla SSRS w trybie klasycznego renderowania HTML, ale nowoczesny portal SSRS (od 2017) także w dużej mierze opiera się na komponentach JS (ReportViewer/Report Portal). W praktyce niewystarczające jako samodzielna metoda dla żadnej z platform w tym projekcie. |
| Wpływ na wydajność | Bardzo dobry, ale nieistotny wobec ograniczeń funkcjonalnych. |
| Odporność na zmiany | n/d — metoda niefunkcjonalna dla Power BI. |
| Zastosowanie bankowe | Wysokie technicznie, ale niskie praktycznie z powodu ograniczeń funkcjonalnych. |
| Złożoność / utrzymanie | Niska, ale niska wartość dostarczana. |
| **Ocena łączna** | 🔴 **Niska — nie nadaje się jako samodzielna metoda integracji dla Power BI; ograniczone zastosowanie dla starszych, czysto server-side raportów SSRS** |

### 3.8 Inne rozważane podejścia

- **Power BI Embedded Analytics (`powerbi-client` JS SDK w kontrolowanej aplikacji embed)** — wymagałoby przebudowy sposobu publikacji raportów przez organizację (osadzenie raportów we własnej aplikacji embed) — poza zakresem projektu, który zakłada integrację z **istniejącą** platformą BI, a nie jej zastąpienie.
- **Komercyjne narzędzia RPA (UiPath, Automation Anywhere, Power Automate Desktop)** — funkcjonalnie zdolne do realizacji podobnego zadania (sterowanie przeglądarką), ale wprowadzają istotny koszt licencyjny, zależność od zewnętrznego środowiska wykonawczego (Robot) oraz mniejszą elastyczność integracji z natywnym stackiem Python/FastAPI wskazanym w projekcie; rozważane jako alternatywa, ale niższa transparentność kodu (low-code) utrudnia utrzymanie w modelu Clean Architecture/Test-Driven wymaganym dla systemu tej klasy.
- **Power BI Report Server On-Premises Data Gateway / PowerShell cmdlets (`ReportingServicesTools`)** — użyteczne do zadań administracyjnych (inwentaryzacja, harmonogramy), rozważane jako pomocnicze narzędzie dla modułu `Configuration Repository` przy imporcie listy raportów, nie jako mechanizm testowy.
- **Sonda syntetyczna na poziomie sieci (health check TCP/HTTP endpointu)** — zbyt płytka (potwierdza jedynie, że serwer odpowiada na warstwie sieciowej, nie że raport się renderuje poprawnie) — rozważana wyłącznie jako uzupełniający, tani "sygnał wczesnego ostrzegania" w module Monitoring, nie jako metoda walidacji raportu.

### 3.9 Tabela porównawcza — integracja z platformą raportową

| Podejście | PBI | SSRS | Wydajność | Odporność na zmiany | Bankowość | Złożoność | Rekomendacja |
|---|:---:|:---:|:---:|:---:|:---:|:---:|---|
| Browser Automation (Playwright) | 🟢 | 🟢 | 🟡 | 🟢 | 🟢 | 🟡 | **Warstwa bazowa — obowiązkowa** |
| Selenium | 🟢 | 🟢 | 🟡 | 🟢 | 🟢 | 🔴 | Odrzucone na rzecz Playwright |
| Power BI REST API | 🟡 | ⚫ | 🟢 | 🟢 | 🟡 | 🟢 | Wsparcie administracyjne / trigger odświeżeń |
| XMLA Endpoint | 🟡* | ⚫ | 🟢 | 🟢 | 🟢 | 🟡 | Opcjonalny connector "drugiej linii" (tylko Premium) |
| SSRS Web Services / URL Access | ⚫ | 🟢 | 🟢 | 🟡 | 🟢 | 🟢 | **Natywny connector SSRS** |
| Eksport (CSV/XML) | 🟡 | 🟢 | 🟡 | 🟢 | 🟢 | 🟡 | Uzupełniająca metoda ekstrakcji wartości |
| Statyczny HTML/DOM (bez JS) | 🔴 | 🟡 | 🟢 | 🔴 | 🟢 | 🟢 | Odrzucone |

*\* zależne od licencji Premium/PPU/Fabric.*

---

## 4. Rekomendowana architektura integracji

**ARGUS przyjmuje architekturę wielowarstwową (multi-connector), w której Browser Automation (Playwright) pełni rolę uniwersalnej warstwy bazowej, a natywne API (SSRS Web Services, Power BI REST API, opcjonalnie XMLA) są wykorzystywane tam, gdzie podnoszą precyzję lub wydajność, bez zastępowania testu end-to-end.**

### Uzasadnienie wyboru Browser Automation jako warstwy bazowej

1. **Jest to jedyne podejście działające identycznie dla obu technologii (PBI i SSRS)**, co pozwala zbudować **jeden spójny rdzeń silnika testowego** i sprowadzić różnice platformowe wyłącznie do warstwy Connectora (zgodnie z zasadą Open/Closed — patrz `03_Architecture.md`).
2. **Jest to jedyne podejście, które realnie weryfikuje to, co widzi użytkownik końcowy** — łącznie z autoryzacją SSO, czasem ładowania, błędami JS, błędami wizualizacji, a nie tylko dostępność API. Cel biznesowy projektu to zastąpienie analityka, który "otwiera raport i patrzy" — Browser Automation jest jedynym podejściem odwzorowującym dokładnie tę czynność.
3. **Nie wymaga żadnych dodatkowych licencji** (Premium/PPU) ani zmiany modelu publikacji raportów — działa z dnia na dzień na już istniejącej platformie BI, bez ingerencji w sposób jej wdrożenia.
4. **Skaluje się horyzontalnie** — worker pool przeglądarek headless można uruchamiać równolegle na wielu instancjach/kontenerach, co jest kluczowe dla celu 1000 raportów.

### Rola API i eksportów jako "drugiej linii"

Tam, gdzie są one dostępne, natywne API **nie zastępują** testu Browser Automation, lecz **wzbogacają Validation Profile o dodatkowy, bardziej precyzyjny kanał odczytu wartości** (patrz Część II, sekcja 6) oraz odciążają warstwę przeglądarkową z zadań czysto administracyjnych (np. wymuszenie odświeżenia datasetu przed testem, pobranie metadanych katalogu SSRS do automatycznej inwentaryzacji raportów w `Configuration Repository`).

Ta hybrydowa strategia jest realizowana architektonicznie przez wzorzec **Strategy + Chain of Responsibility** wewnątrz `Value Extraction Engine` — patrz sekcja 7 poniżej oraz `03_Architecture.md`.

---

## 5. Strategia odporności na zmiany raportów (Resilience Strategy)

Ponieważ liczba raportów (docelowo 1000) uniemożliwia ręczne utrzymanie sztywnych, "kruchych" selektorów, ARGUS wdraża następujące zasady projektowe od samego początku:

1. **Selektory semantyczne, nie pozycyjne/wizualne** — wykorzystanie atrybutów `aria-label`, ról ARIA, nazw elementów Power BI (np. nazwa wizualizacji nadana przez autora raportu) zamiast współrzędnych pikselowych czy generowanych klas CSS.
2. **Multi-selector fallback** — dla każdego punktu ekstrakcji Validation Profile może definiować więcej niż jedną strategię odnalezienia elementu (np. "po nazwie wizualizacji" → "po pozycji w siatce" → "po indeksie w kolejności renderowania"), próbowane sekwencyjnie.
3. **Confidence Score** ekstrakcji — każda odczytana wartość otrzymuje wskaźnik pewności odczytu (np. czy selektor trafił jednoznacznie w jeden element, czy zastosowano fallback niższego poziomu) — widoczny w GUI i wykorzystywany przez moduł AI do priorytetyzacji przeglądu.
4. **Automatyczna detekcja "podejrzanej zmiany strukturalnej"** — jeśli struktura DOM raportu zmieni się istotnie względem poprzedniego udanego przebiegu, system oznacza to jako alert dla administratora Validation Profile, zamiast cicho zwracać błędny odczyt.
5. **Wersjonowanie Validation Profile** — każda zmiana selektora/profilu jest wersjonowana i audytowalna (kto, kiedy, dlaczego zmienił definicję testu).

---

## 6. Część II — Value Extraction Engine: metody odczytu wartości KPI z raportu

To najbardziej newralgiczny komponent całego systemu — jego niezawodność determinuje zaufanie użytkowników biznesowych do wyników ARGUS.

### 6.1 Analiza DOM (po wyrenderowaniu przez przeglądarkę)

| Kryterium | Ocena |
|---|---|
| Zalety | Odczyt dokładnie tego tekstu/liczby, który widzi użytkownik. Działa dla wizualizacji renderowanych jako tekst/tabela (karty KPI, tabele, matryce) — najczęstszy typ wizualizacji używany do prezentacji pojedynczych wskaźników. |
| Ograniczenia | Nie działa dla wartości renderowanych wyłącznie jako grafika wektorowa/Canvas bez odpowiadającego tekstu w DOM (niektóre typy wykresów Power BI renderują się częściowo na `<canvas>`). Wymaga strategii selekcji odpornej na zmiany (patrz sekcja 5). |
| Odporność na zmiany | Średnia–wysoka przy zastosowaniu selektorów semantycznych. |
| Wpływ na wydajność | Bardzo dobry — odczyt po stronie już załadowanej strony, rzędu dziesiątek ms. |
| Niezawodność | Wysoka dla kart KPI/tabel (dominujący przypadek użycia dla "wartości KPI" wg briefu). |
| Power BI | 🟢 Bardzo dobre dla kart, tabel, macierzy. |
| SSRS | 🟢 Bardzo dobre — SSRS renderuje tabele/textboxy jako elementy HTML/SVG z tekstem w DOM. |

### 6.2 Wykorzystanie natywnego API / zapytań do modelu danych (XMLA/DAX, SSRS SOAP dataset)

| Kryterium | Ocena |
|---|---|
| Zalety | Najwyższa możliwa precyzja — wartość pobrana bezpośrednio z silnika obliczeniowego, tożsama co do bitu z tym, co obliczyłaby wizualizacja. Brak zależności od layoutu wizualnego raportu. |
| Ograniczenia | Wymaga znajomości definicji miary/zapytania leżącego pod wizualizacją (utrzymywane w Validation Profile przez administratora/analityka). Dla Power BI ograniczone do licencji Premium/PPU/Fabric (XMLA). Nie potwierdza, że wizualizacja faktycznie renderuje tę wartość poprawnie na ekranie (możliwy błąd formatowania/mapowania w samej wizualizacji, niewykrywalny tą metodą). |
| Odporność na zmiany | Bardzo wysoka wobec zmian UI, niska wobec zmian nazw miar (rzadkie, kontrolowane zmiany). |
| Wpływ na wydajność | Bardzo dobry. |
| Niezawodność | Bardzo wysoka tam, gdzie dostępne. |
| Power BI | 🟡 Dobre, warunkowe licencyjnie. |
| SSRS | 🟡 Częściowo — poprzez `ReportExecution2005.Render` do formatu danych (XML/CSV) można odczytać wartość bezpośrednio z zestawu danych raportu. |

### 6.3 Eksport danych (CSV/XML/Excel) i parsowanie strukturalne

| Kryterium | Ocena |
|---|---|
| Zalety | Bardzo wysoka niezawodność parsowania (dane w formie ustrukturyzowanej, nie tekstu wizualnego). Dobry "ślad audytowy" — plik eksportu można zachować jako dowód. |
| Ograniczenia | Generowanie pełnego eksportu bywa wolniejsze niż odczyt jednej wartości z DOM. Zmiana układu kolumn w raporcie wymaga aktualizacji mapowania w Validation Profile. |
| Odporność na zmiany | Wysoka (format danych), średnia (układ kolumn). |
| Wpływ na wydajność | Średni (generacja pełnego dokumentu), ale wykonywalne asynchronicznie/w tle. |
| Niezawodność | Wysoka. |
| Power BI | 🟡 Dobre (`Export to File` REST API), z ograniczeniami rate-limit i asynchronicznością. |
| SSRS | 🟢 Bardzo dobre (URL Access `rs:Format=CSV/XML` — natywne, szybkie do wdrożenia). |

### 6.4 Analiza HTML (statyczny parser, bez wykonania JS)

Jak w sekcji 3.7 — nieadekwatne dla Power BI (SPA renderowany przez JS), ograniczone dla SSRS. Odrzucone jako samodzielna metoda ekstrakcji.

### 6.5 Browser Automation jako mechanizm ekstrakcji (odczyt po renderowaniu — accessibility tree, `innerText`, atrybuty)

Rozszerzenie DOM Analysis (6.1) o pełne możliwości Playwright: odczyt drzewa dostępności (Accessibility Tree), które Power BI aktywnie utrzymuje dla potrzeb czytników ekranu — jest to **bardzo stabilne semantycznie źródło**, bo Microsoft utrzymuje je pod kątem zgodności z WCAG, niezależnie od zmian czysto wizualnych.

| Power BI | 🟢 Bardzo dobre — accessibility tree Power BI jest zaskakująco stabilnym i mało wykorzystywanym źródłem odczytu wartości kart/tabel. |
| SSRS | 🟢 Dobre. |

### 6.6 Computer Vision (detekcja układu, template matching, analiza wykresu)

| Kryterium | Ocena |
|---|---|
| Zalety | Jedyna metoda działająca dla wizualizacji renderowanych czysto graficznie (np. niektóre wykresy na `<canvas>` bez warstwy tekstowej w DOM), gdzie DOM/API zawodzą. Może wykrywać także **anomalie wizualne** niezwiązane z konkretną liczbą (np. "wykres jest pusty", "brak legendy", "nakładające się etykiety"). |
| Ograniczenia | Wysoka złożoność implementacyjna. Niższa precyzja numeryczna niż DOM/API (zwłaszcza dla wartości z wieloma miejscami po przecinku). Wrażliwe na zmiany kolorystyki/motywu raportu. |
| Odporność na zmiany | Średnia. |
| Wpływ na wydajność | Wyższy koszt obliczeniowy (przetwarzanie obrazu, ewentualnie model ML). |
| Niezawodność | Średnia — rekomendowane jako metoda **ostatniej linii fallbacku**, nie podstawowa. |
| Power BI | 🟡 Fallback dla wykresów canvas. |
| SSRS | 🟡 Rzadziej potrzebne (SSRS częściej renderuje tekst w DOM/SVG). |

### 6.7 OCR (Optical Character Recognition)

| Kryterium | Ocena |
|---|---|
| Zalety | Działa "na obrazie", niezależnie od technologii renderującej — uniwersalny fallback ostatniej instancji (np. dla zrzutu ekranu całego raportu, gdy żadna inna metoda nie zidentyfikowała elementu). |
| Ograniczenia | Najniższa precyzja i niezawodność spośród wszystkich metod (ryzyko błędnego odczytu cyfr, separatorów dziesiętnych, jednostek). Najwyższy koszt czasowy. Wymaga starannej walidacji formatu liczby po odczycie (np. reguła "odczytana wartość musi pasować do wzorca liczby, inaczej odrzuć wynik i oznacz jako niepewny"). |
| Odporność na zmiany | Wysoka na zmiany strukturalne DOM (bo nie zależy od struktury), ale niska precyzja per se. |
| Wpływ na wydajność | Najwyższy koszt spośród analizowanych metod. |
| Niezawodność | Niska–średnia — wyłącznie jako ostateczny fallback z obowiązkowym oznaczeniem niskiego Confidence Score i skierowaniem do przeglądu (ewentualnie z udziałem modułu AI jako "podpowiedzi", nigdy jako źródła automatycznej decyzji). |
| Power BI | 🟡 Fallback ostatniej instancji. |
| SSRS | 🟡 Fallback ostatniej instancji. |

### 6.8 Tabela porównawcza — Value Extraction Engine

| Metoda | Niezawodność | Wydajność | Odporność na zmiany | PBI | SSRS | Warstwa strategii |
|---|:---:|:---:|:---:|:---:|:---:|---|
| DOM Analysis (karty/tabele) | 🟢 Wysoka | 🟢 | 🟡 | 🟢 | 🟢 | **Warstwa 1 — podstawowa** |
| Accessibility Tree | 🟢 Wysoka | 🟢 | 🟢 | 🟢 | 🟢 | **Warstwa 1 — podstawowa** |
| API / model danych (XMLA/DAX, SSRS dataset) | 🟢 Bardzo wysoka | 🟢 | 🟢 | 🟡* | 🟡 | **Warstwa 0 — precyzyjna, gdy dostępna** |
| Eksport strukturalny (CSV/XML) | 🟢 Wysoka | 🟡 | 🟢 | 🟡 | 🟢 | **Warstwa 1b — alternatywna** |
| Computer Vision | 🟡 Średnia | 🟡 | 🟡 | 🟡 | 🟡 | Warstwa 2 — fallback |
| OCR | 🔴 Niska–średnia | 🔴 | 🟢 | 🟡 | 🟡 | Warstwa 3 — fallback ostateczny |
| Statyczny HTML (bez JS) | 🔴 Niska | 🟢 | 🔴 | 🔴 | 🟡 | Odrzucone |

*\* zależne od licencji.*

---

## 7. Rekomendowana architektura Value Extraction Engine — strategia wielowarstwowego fallbacku

System **nie wybiera jednej metody globalnie**, lecz stosuje **łańcuch strategii ekstrakcji per punkt danych**, skonfigurowany deklaratywnie w Validation Profile, zgodnie ze wzorcem projektowym **Chain of Responsibility** opakowanym w **Strategy Pattern** (każda strategia implementuje wspólny interfejs `IExtractionStrategy`):

```
Punkt ekstrakcji KPI w Validation Profile
        │
        ▼
[Warstwa 0] Native API / model danych  ──── dostępne? ──▶ zwróć wartość (Confidence: bardzo wysoki)
        │ niedostępne / błąd
        ▼
[Warstwa 1] DOM Analysis + Accessibility Tree ──▶ trafiony jednoznacznie? ──▶ zwróć wartość (Confidence: wysoki)
        │ niejednoznaczne / brak elementu
        ▼
[Warstwa 1b] Eksport strukturalny (CSV/XML) ──▶ sukces? ──▶ zwróć wartość (Confidence: wysoki)
        │ niedostępne
        ▼
[Warstwa 2] Computer Vision (template/region matching) ──▶ sukces? ──▶ zwróć wartość (Confidence: średni)
        │ niepewne
        ▼
[Warstwa 3] OCR na zrzucie ekranu regionu ──▶ zwróć wartość (Confidence: niski) + flaga "wymaga przeglądu"
        │ brak jakiegokolwiek odczytu
        ▼
Status: EXTRACTION_FAILED → alert + zadanie w kolejce przeglądu manualnego
```

**Uzasadnienie:**

1. **Determinizm i przewidywalność kosztu** — najdroższe metody (CV/OCR) uruchamiane są tylko wtedy, gdy tańsze i pewniejsze zawiodły, co optymalizuje koszt obliczeniowy przy skali 1000 raportów.
2. **Confidence Score jako obywatel pierwszej klasy** — każdy wynik niesie ze sobą informację o pewności odczytu, co jest kluczowe dla modułu AI (priorytetyzacja anomalii) oraz dla zaufania użytkownika biznesowego do automatyzacji.
3. **Odporność na zmiany raportów** — awaria jednej warstwy nie unieważnia całego testu, tylko degraduje pewność wyniku i uruchamia kolejną strategię.
4. **Rozszerzalność (Open/Closed)** — dodanie nowej metody ekstrakcji (np. w przyszłości integracja z nowym modelem AI vision) to dodanie nowej klasy implementującej `IExtractionStrategy`, bez zmian w rdzeniu silnika.

Pełny opis wzorców i interfejsów — patrz `03_Architecture.md`, sekcja "Value Extraction Engine".

---

## 8. Otwarte pytania do wyjaśnienia w Fazie 0 (Discovery / PoC)

Poniższe kwestie wymagają potwierdzenia przez zespół administrujący Platformą BI przed rozpoczęciem implementacji (patrz `11_Roadmap.md`, Faza 0):

1. Czy Power BI jest wdrożone jako **Power BI Report Server on-prem**, czy jako usługa hybrydowa z komponentem chmurowym (wpływa na dostępność REST API i XMLA)?
2. Czy dostępna jest licencja Premium/PPU/Fabric dla choćby części raportów (kwalifikacja do connectora XMLA jako opcji "drugiej linii")?
3. Jaki jest dokładny mechanizm SSO — Kerberos/SPNEGO czy NTLM — oraz czy konto techniczne ARGUS będzie mogło działać w modelu delegacji (impersonacji) czy wyłącznie jako dedykowane konto serwisowe z bezpośrednim dostępem nadanym do raportów?
4. Czy istnieje (lub może powstać) scentralizowany katalog metadanych raportów (spis 300/1000 raportów wraz z adresami i obszarami biznesowymi), czy ARGUS musi zbudować własną inwentaryzację od zera w module `Configuration Repository`?
5. Jakie są polityki sieciowe dot. ruchu wychodzącego z serwerów ARGUS do `pbi.bank.com.pl` oraz `bgkssrs` (segmentacja sieci, wymagane wyjątki firewall)?

## 9. Podsumowanie rekomendacji

| Obszar | Rekomendacja |
|---|---|
| Warstwa integracji z raportem | **Playwright (Browser Automation)** jako warstwa bazowa, uniwersalna dla PBI i SSRS |
| Connector SSRS — wzbogacenie | SSRS Web Services / URL Access do metadanych katalogu i eksportu strukturalnego |
| Connector Power BI — wzbogacenie | Power BI REST API do operacji administracyjnych; XMLA jako opcjonalny connector precyzyjny (Premium) |
| Odczyt wartości KPI | Silnik wielowarstwowego fallbacku: API/model danych → DOM/Accessibility Tree → Eksport strukturalny → Computer Vision → OCR |
| Zasada nadrzędna | Każda decyzja o niedostępności/błędzie raportu musi być **deterministyczna i wyjaśnialna** — AI komentuje, nie decyduje (patrz `06_AI_Module.md`) |
