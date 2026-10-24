# Sprawozdanie z ćwiczenia: Analiza wysokorozdzielczych danych CIR, obliczanie wskaźnika NDVI oraz statystyka strefowa (Zonal Statistics) w programie QGIS

**Autor:** Tomasz Broda  
**Data wykonania:** 24 października 2026 r.  
**Obszar opracowania:** Opole – rejon Zaodrza / Śródmieścia (arkusz ortofotomapy: `M-33-48-D-d-3-4`)  
**Dane źródłowe:** Lotnicza ortofotomapa wysokorozdzielcza CIR (GUGiK / Geoportal.gov.pl), piksel 0,25 m, pozyskana 13.06.2025 r.

---

## 1. Wprowadzenie teoretyczne

### Czym jest obrazowanie CIR (Color Infrared)?
**CIR (Color Infrared)**, czyli obrazowanie w barwach umownych (fałszywych barwach) z wykorzystaniem pasma bliskiej podczerwieni (*Near Infrared, NIR*), to technika wizualizacji wielospektralnej, w której jeden z kanałów światła widzialnego zastępowany jest kanałem bliskiej podczerwieni. Standardowo stosuje się następujące przypisanie kanałów do składowych wyświetlania RGB:
* **Czerwony (R):** kanał bliskiej podczerwieni (**NIR**)
* **Zielony (G):** kanał światła czerwonego (**Red**)
* **Niebieski (B):** kanał światła zielonego (**Green**)

### Różnice pomiędzy RGB a CIR
* **Obrazowanie RGB (barwy naturalne):** Roślinność obserwujemy jako zieloną, tak jak rejestruje ją ludzkie oko, ze względu na obecność barwnika chlorofilu.
* **Obrazowanie CIR (barwy umowne):** Zdrowa, gęsta roślinność silnie odbija promieniowanie w zakresie bliskiej podczerwieni (ze względu na wewnętrzną strukturę komórkową miękiszu gąbczastego liści) przy jednoczesnym silnym pochłanianiu światła czerwonego przez chlorofil. Na obrazie CIR roślinność przyjmuje jaskrawe odcienie czerwieni, purpury i karminu.
* Obiekty antropogeniczne (asfalt, beton, dachy) oraz woda wykazują niskie odbicie w NIR, co sprawia, że na kompozycji CIR są szare, błękitne lub niemal czarne. Ułatwia to natychmiastowe odróżnienie zieleni od sztucznych nawierzchni.

---

## 2. Cel ćwiczenia

1. Pozyskanie wysokorozdzielczej ortofotomapy lotniczej w kompozycji CIR z państwowego zasobu geodezyjnego i kartograficznego za pośrednictwem serwisu **Geoportal.gov.pl**.
2. Wczytanie i poprawna interpretacja kanałów ortofotomapy w środowisku **QGIS** (ze szczególnym uwzględnieniem mapowania kanałów: Kanał 1 = NIR, Kanał 2 = Red).
3. Samodzielne obliczenie wskaźnika wegetacji **NDVI (Normalized Difference Vegetation Index)** w kalkulatorze rastrów dla danych o rozdzielczości terenowej 25 cm.
4. Utworzenie tymczasowej warstwy wektorowej (*Scratch Layer*) oraz zwektoryzowanie zróżnicowanych obszarów testowych (działek/stref zagospodarowania terenu).
5. Wykonanie analizy statystyk strefowych (**Zonal Statistics**) w celu ilościowego porównania wartości NDVI dla poszczególnych form pokrycia terenu.

---

## 3. Przebieg realizacji ćwiczenia

### Krok 1: Wyszukanie i pobranie danych z Geoportal.gov.pl

Zgodnie z procedurą skorzystano z serwisu [Geoportal.gov.pl](https://www.geoportal.gov.pl/):
1. Przejście do modułu mapowego (*Pobierz dane*).
2. W drzewie warstw aktywowano: **Dane do pobrania** → **Ortofotomapa** → **Ortofotomapa wg rozdzielczości**.
3. Zlokalizowano obszar Opola (rejon Ratusza i Zaodrza).
4. Za pomocą narzędzia identyfikacji wybrano arkusz ortofotomapy o godle **M-33-48-D-d-3-4** wykonany **13.06.2025 r.** w barwach CIR.

Metadane pobranego arkusza:
* **Godło:** `M-33-48-D-d-3-4`
* **Aktualność:** 2025-06-13
* **Wielkość piksela (GSD):** 0,25 m (25 cm)
* **Układ współrzędnych:** PL-1992 (EPSG:2180)
* **Typ barw:** CIR
* **Rozmiar pliku:** ok. 54,9 MB

![Wyszukanie i identyfikacja arkusza CIR na Geoportalu](screenshots/Screenshot%202026-10-24%20at%2015.08.28.png)

*Rysunek 1. Identyfikacja arkusza ortofotomapy M-33-48-D-d-3-4 CIR w serwisie Geoportal.gov.pl.*

Pobrany plik GeoTIFF (`83854_1558844_M-33-48-D-d-3-4.tif`) zapisano w katalogu roboczym projektu.

![Zapisany plik w katalogu roboczym](screenshots/Screenshot%202026-10-24%20at%2015.08.37.png)

*Rysunek 2. Plik rastrowy ortofotomapy oraz projekt QGIS w katalogu roboczym.*

---

### Krok 2: Załadowanie ortofotomapy do QGIS i wizualizacja CIR

Wczytano pobrany raster do programu QGIS wraz z podkładem referencyjnym Google Maps Hybrid.

Kanały ortofotomapy CIR zostały zmapowane w kompozycji fałszywych barw:
* Kanał 1 (NIR) → Składowa Czerwona (Red)
* Kanał 2 (Red) → Składowa Zielona (Green)
* Kanał 3 (Green) → Składowa Niebieska (Blue)

Dzięki temu roślinność (parki, ogrody działkowe, szpalery drzew wzdłuż ulic) uwidoczniła się w wyrazistych, intensywnie czerwonych i bordowych odcieniach.

![Wizualizacja ortofotomapy CIR w QGIS](screenshots/Screenshot%202026-10-24%20at%2015.08.48.png)

*Rysunek 3. Wizualizacja ortofotomapy w kompozycji CIR w QGIS na tle podkładu satelitarnego.*

---

### Krok 3: Obliczenie wskaźnika NDVI w Kalkulatorze rastrów

Do wyznaczenia znormalizowanego różnicowego wskaźnika wegetacji (NDVI) wykorzystano narzędzie **Raster Calculator (Kalkulator rastrów)**. 

Wzór definiujący NDVI to:
$$\text{NDVI} = \frac{\text{NIR} - \text{RED}}{\text{NIR} + \text{RED}}$$

Mając na uwadze specyfikację ortofotomapy CIR, gdzie **kanał 1 odpowiada pasmu NIR**, a **kanał 2 odpowiada pasmu czerwieni (RED)**, zdefiniowano wyrażenie rastrowe:

```text
("83854_1558844_M-33-48-D-d-3-4@1" - "83854_1558844_M-33-48-D-d-3-4@2") / ("83854_1558844_M-33-48-D-d-3-4@1" + "83854_1558844_M-33-48-D-d-3-4@2")
```

Parametry zapisu:
* Warstwa wyjściowa: `NDVI_High_Res.tif`
* Format: **GeoTIFF**
* Zakres przestrzenny (Extent): zgodny z rastrem wejściowym (North: 313960, South: 311611.75, West: 422727.5, East: 424968.5)
* Rozdzielczość przestrzenna: 0,25 m/px

![Konfiguracja kalkulatora rastrów dla NDVI](screenshots/Screenshot%202026-10-24%20at%2015.11.16.png)

*Rysunek 4. Definicja wyrażenia w Kalkulatorze rastrów QGIS.*

---

### Krok 4: Analiza wysokorozdzielczego rastra NDVI

W wyniku obliczeń wygenerowano raster o wymiarach 8964 × 9393 pikseli i rozdzielczości terenowej 25 cm. Zakres uzyskanych wartości mieści się w granicach od ok. -1,0 (powierzchnie nieorganiczne i woda) do +0,69 (bujna, zdrowa zieleń wysokiej gęstości).

Dzięki rozdzielczości 0,25 m widoczne są szczegóły niemożliwe do zaobserwowania na zobrazowaniach satelitarnych Sentinel-2 (piksel 10 m z Laboratorium 1) – m.in. pojedyncze korony drzew, zieleńce przydrożne, granice działek rekreacyjnych oraz zróżnicowanie wegetacji na przydomowych trawnikach.

![Wynikowy raster NDVI wysokiej rozdzielczości](screenshots/Screenshot%202026-10-24%20at%2015.11.56.png)

*Rysunek 5. Wynikowy raster NDVI (NDVI_High_Res) o rozdzielczości 25 cm w programie QGIS.*

---

### Krok 5: Wektoryzacja poligonów testowych (warstwa tymczasowa)

W celu przeprowadzenia analizy strefowej:
1. Utworzono nową warstwę tymczasową (*New Temporary Scratch Layer* / **Warstwa tymczasowa**) o nazwie **`Działki`** o typie geometrii **Polygon** (poligon).
2. Przeprowadzono digitalizację (wektoryzację) kilku reprezentatywnych obszarów testowych w Opolu (m.in. obszary zielone/ogrody działkowe, tereny zabudowy mieszkaniowej oraz strefy utwardzone).

![Wektoryzacja poligonów na warstwie tymczasowej](screenshots/Screenshot%202026-10-24%20at%2015.37.37.png)

*Rysunek 6. Zwektoryzowane poligony na warstwie „Działki” przygotowane do analizy statystyk strefowych.*

---

### Krok 6: Statystyka strefowa (Zonal Statistics)

Uruchomiono narzędzie **Processing Toolbox** → **Raster analysis** → **Zonal Statistics (Statystyka strefowa)**:
* **Warstwa wektorowa (Input layer):** `Działki`
* **Warstwa rastrowa (Raster layer):** `NDVI_High_Res`
* **Wyliczane parametry statystyczne:**
  * `_count` – liczba pikseli rastra wewnątrz poligonu,
  * `_sum` – suma wartości NDVI pikseli w obszarze,
  * `_mean` – średnia wartość NDVI charakteryzująca dany poligon.

W wyniku działania algorytmu powstała warstwa wynikowa **`Zonal Statistics`**, wzbogacona o pola atrybutów zawierające podsumowanie kondycji roślinności w obrębie każdej wydzielonej strefy.

![Zastosowanie narzędzia Zonal Statistics](screenshots/Screenshot%202026-10-24%20at%2015.41.30.png)

*Rysunek 7. Wynik wykonania narzędzia Zonal Statistics w QGIS.*

Dla stref z dominacją drzewostanu i ogrodów działkowych średnia wartość wskaźnika NDVI osiągnęła wartości wysokie (dodatnie, w granicach 0,45–0,60), natomiast dla stref zabudowanych i parkingów średnie NDVI spadło w okolice zera lub wartości ujemnych.

---

## 4. Porównanie z danymi satelitarnymi Sentinel-2 (Lab 1 vs Lab 1 Add-on)

| Cecha / Parametr | Laboratorium 1 (Sentinel-2 L2A) | Laboratorium 1 Add-on (Ortofotomapa CIR GUGiK) |
| :--- | :--- | :--- |
| **Platforma pozyskania** | Satelita Sentinel-2 (ESA) | Nalot lotniczy (GUGiK / Geoportal) |
| **Rozdzielczość przestrzenna (GSD)** | 10 m / piksel | **0,25 m / piksel (25 cm)** |
| **Układ kanałów** | Kanały w osobnych plikach (B04=RED, B08=NIR) | Plik 3-kanałowy CIR (Band 1=NIR, Band 2=RED, Band 3=GREEN) |
| **Poziom szczegółowości** | Skala regionalna / krajobrazowa (lasy, jeziora, miasta) | **Skala mikro / miejska (pojedyncze drzewa, krzewy, dachy, działki)** |
| **Zastosowane metody** | Raster Calculator, stylizacja Pseudocolor | Raster Calculator, CIR False Color, wektoryzacja, **Zonal Statistics** |

---

## 5. Podsumowanie i wnioski

1. **Efektywność zobrazowań CIR:** Zastąpienie pasma światła niebieskiego bliską podczerwienią (NIR) umożliwia natychmiastową, wizualną ocenę stanu biomasy roślinnej już na etapie wstępnej analizy kompozycji barwnej.
2. **Kluczowa rola kolejności kanałów:** Przy pracy z danymi lotniczymi CIR kluczowe jest zweryfikowanie kolejności pasm w pliku GeoTIFF. Zgodnie ze specyfikacją standardu GUGiK pasmo 1 to NIR, a pasmo 2 to RED, co determinuje prawidłowy zapis formuły NDVI w kalkulatorze rastrów.
3. **Przewaga wysokiej rozdzielczości (25 cm):** W odróżnieniu od danych satelitarnych o rozdzielczości 10 m, ortofotomapa lotnicza pozwala na analizę struktury zieleni na poziomie poszczególnych nieruchomości, działek geodezyjnych czy pasów drogowych.
4. **Zastosowanie Zonal Statistics:** Narzędzie statystyk strefowych stanowi most między danymi ciągłymi (rastry wskaźnikowe NDVI) a danymi dyskretnymi (obszary wektorowe), umożliwiając automatyczną ocenę wskaźnikową powierzchni biologicznie czynnych na potrzeby urbanistyki, gospodarki przestrzennej i monitoringu środowiskowego.
