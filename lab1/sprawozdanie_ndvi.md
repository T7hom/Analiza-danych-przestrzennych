# Sprawozdanie z ćwiczenia: Obliczanie i wizualizacja wskaźnika wegetacji NDVI w programie QGIS

**Autor:** Tomasz Broda
**Data wykonania:** 10 października 2026 r.
**Obszar opracowania:** Mierzeja jeziora Jamno / Mielno–Unieście
**Dane satelitarne:** Sentinel-2 L2A (scena z dnia 01.10.2026 r.)

---

## 1. Cel ćwiczenia

Celem zadania było:

1. Pozyskanie wielospektralnych danych satelitarnych Sentinel-2 z platformy **Copernicus Browser**.
2. Wczytanie surowych kanałów spektralnych do środowiska GIS (**QGIS**).
3. Samodzielne obliczenie znormalizowanego różnicowego wskaźnika wegetacji (**NDVI**) za pomocą kalkulatora rastrów.
4. Wizualizacja i analiza przestrzenna uzyskanych wartości wskaźnika na tle ortofotomapy / kompozycji barwnej.

---

## 2. Krok 1: Wybór obszaru i pobranie danych z Copernicus Browser

W serwisie [Copernicus Browser](https://browser.dataspace.copernicus.eu/) zdefiniowano poligon badawczy (AOI) obejmujący mierzeję odcinającą jezioro Jamno od Morza Bałtyckiego (powierzchnia ok. 16,46 km²). Wybrano bezchmurne zobrazowanie z kolekcji **Sentinel-2 L2A** zarejestrowane w dniu **01.10.2026 r.**

![Zdefiniowanie obszaru zainteresowania (AOI)](screenshots/Screenshot%202026-10-10%20at%2016.45.56.png)

_Rysunek 1. Wybór obszaru badawczego (Mielno / jezioro Jamno) w Copernicus Browser._

W module pobierania (**Analytical**) skonfigurowano parametry eksportu:

- **Format:** TIFF (32-bit float),
- **Rozdzielczość:** Własna (CUSTOM) – 10 m/px (w osiach X i Y),
- **Układ współrzędnych:** WGS 84 (EPSG:4326),
- **Pobrane warstwy:**
  - Kanał czerwony: **B04** (665 nm, rozdzielczość 10 m),
  - Kanał bliskiej podczerwieni: **B08** (842 nm, rozdzielczość 10 m),
  - Warstwa referencyjna **True color** oraz podgląd gotowego indeksu **NDVI**.

![Parametry pobierania danych w Copernicus Browser](screenshots/Screenshot%202026-10-10%20at%2016.46.13.png)

_Rysunek 2. Konfiguracja eksportu pasm spektralnych w zakładce Analytical._

Pobrane pliki rastrowe zapisano na dysku w katalogu roboczym.

![Pobrane pliki rastrowe](screenshots/Screenshot%202026-10-10%20at%2016.46.30.png)

_Rysunek 3. Pobrane kanały GeoTIFF w folderze projektowym._

---

## 3. Krok 2: Obliczenie wskaźnika NDVI w QGIS

Pobrane kanały `B04` i `B08` wczytano do projektu QGIS. Następnie uruchomiono narzędzie **Raster Calculator (Kalkulator rastrów)** (**Raster** → **Raster Calculator...**).

Wskaźnik NDVI zdefiniowany jest wzorem:
$$\text{NDVI} = \frac{\text{NIR} - \text{RED}}{\text{NIR} + \text{RED}}$$

W kalkulatorze wprowadzono wyrażenie:

```text
("B08_NIR@1" - "B04_RED@1") / ("B08_NIR@1" + "B04_RED@1")
```

Plik wynikowy zapisano w formacie **GeoTIFF** pod nazwą roboczą na dysku, zachowując zasięg i rozdzielczość pasm wejściowych.

![Obliczenie NDVI w kalkulatorze rastrów QGIS](screenshots/Screenshot%202026-10-10%20at%2016.48.27.png)

_Rysunek 4. Konfiguracja wyrażenia rastrowego w Raster Calculator._

---

## 4. Krok 3: Analiza i wizualizacja wyników

Po wygenerowaniu warstwy wskaźnika NDVI uzyskano surowy raster monochromatyczny. Wartości wskaźnika mieszczą się w teoretycznym zakresie od -1,0 do +1,0.

![Surowy raster wynikowy NDVI](screenshots/Screenshot%202026-10-10%20at%2017.24.15.png)

_Rysunek 5. Surowy raster NDVI w skali szarości na tle mapy bazowej._

### Wizualizacja w pseudokolorze

Warstwę ostylowano za pomocą trybu **Singleband pseudocolor (Jednokanałowy pseudokolor)** z paletą barwną od czerwieni do zieleni:

- **Wartości ujemne i bliskie zera (odcienie czerwieni):** akweny wodne (Bałtyk, jezioro Jamno), plaża, piasek oraz tereny gęsto zurbanizowane.
- **Wartości wysokie powyżej 0,6–0,8 (odcienie zieleni):** tereny porośnięte bujną, zdrową roślinnością, lasy nadmorskie oraz pola uprawne.

![Wizualizacja NDVI w palecie barwnej](screenshots/Screenshot%202026-10-10%20at%2017.24.25.png)

_Rysunek 6. Wizualizacja wskaźnika NDVI w palecie barwnej._

Dla celów weryfikacji i porównania wczytano również warstwę **True Color** (kompozycja naturalna), która potwierdza poprawność detekcji szaty roślinnej na mierzei i terenach przyległych.

![Kompozycja w barwach rzeczywistych (True Color)](screenshots/Screenshot%202026-10-10%20at%2016.49.43.png)

_Rysunek 7. Porównanie z kompozycją w barwach naturalnych (True Color)._

---

## 5. Podsumowanie i wnioski

1. Wykorzystanie surowych kanałów `B04` i `B08` sensora MSI satelity Sentinel-2 pozwoliło na uzyskanie mapy wskaźnika NDVI o wysokiej rozdzielczości przestrzennej (10 m).
2. Wskaźnik poprawnie separuje obszary wodne (ujemne wartości NDVI) oraz obszary zurbanizowane/plaże od pasów zieleni leśnej i łąkowej na mierzei.
3. Narzędzie **Raster Calculator** w programie QGIS umożliwia wydajne i bezbłędne przetwarzanie danych wielospektralnych bezpośrednio z formatu GeoTIFF.
