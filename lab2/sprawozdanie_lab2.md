# Sprawozdanie z ćwiczenia: Analiza danych przestrzennych w bibliotekach GeoPandas i Rasterio (Część A i B)

> [!NOTE]
> **Status dokumentu:** Ukończono **Część A (GeoPandas, kroki A1–A9)** oraz **Część B (Rasterio, kroki B1–B9)**. Część C (aplikacja Streamlit) stanowi kolejny etap projektu.

**Autor:** Tomasz Broda
**Data wykonania:** 24 października – 21 listopada 2026 r.
**Środowisko:** Python 3.12 (wirtualne środowisko `.lab-geo`)
**Wykorzystane biblioteki:** GeoPandas 1.2.0, Shapely 2.2.0, Rasterio 1.5.2 (GDAL 3.12.2), NumPy, Pandas, Matplotlib, Folium, Mapclassify, Pyproj
**Obszar i dane:** Pomiary jakości powietrza (pył zawieszony PM2.5) z czujników naziemnych, granice 29 dzielnic Opola, lokalizacje szkół podstawowych oraz scena satelitarna Sentinel-2 L2A z 6 października 2026 r. (pasma B04 – Red i B08 – NIR)

---

## 1. Wprowadzenie i scenariusz badawczy

Celem ćwiczenia jest praktyczne opanowanie przetwarzania i analizy danych geoprzestrzennych z wykorzystaniem biblioteki **GeoPandas** oraz narzędzi ekosystemu Pythona.

### Scenariusz

Miasto Opole otrzymało surowy plik CSV (`pm25_sensors.csv`) zawierający odczyty stężenia pyłu zawieszonego PM2.5 zarejestrowane przez sieć czujników zlokalizowanych na terenie miasta. Dane pochodzą bezpośrednio z terenu i obarczone są typowymi błędami pomiarowymi oraz technicznymi (duplikaty pakietów, brakujące współrzędne, zamieniona kolejność długości i szerokości geograficznej, kody błędów czujnika).

Główne pytania badawcze:

1. **Które dzielnice Opola charakteryzują się najgorszą jakością powietrza?**
2. **Jak kształtuje się poziom stężenia pyłu PM2.5 w bezpośrednim sąsiedztwie opolskich szkół podstawowych?**

Kluczowa zasada towarzysząca realizacji:
$$\text{Dane przestrzenne} = \text{Tabela danych} + \text{Geometria} + \text{Układ współrzędnych (CRS)}$$

---

## 2. Krok 0: Przygotowanie środowiska i weryfikacja bibliotek

W katalogu roboczym `lab2/` utworzono dedykowane środowisko wirtualne Pythona (`.lab-geo`) oraz zainstalowano wymagane biblioteki geoinformatyczne.

W notatniku `lab_geopandas.ipynb` zweryfikowano poprawność instalacji oraz wersje kluczowych pakietów:

```python
import geopandas as gpd
import rasterio
import shapely

print("geopandas", gpd.__version__)
print("rasterio ", rasterio.__version__, "| GDAL", rasterio.__gdal_version__)
print("shapely  ", shapely.__version__)
```

**Wynik testu środowiska (Punkt kontrolny 0):**

```text
geopandas 1.2.0
rasterio  1.5.2 | GDAL 3.12.2
shapely   2.2.0
```

Wszystkie biblioteki załadowały się poprawnie; GeoPandas posiada wersję 1.2.0 (spełniając wymóg wersji $\ge 1.0$).

---

## 3. Krok A1: Geometria i GeoDataFrame od podstaw

Zbudowano obiekty geometryczne od zera przy użyciu biblioteki **Shapely** (`Point`, `Polygon`), pamiętając o kolejności osi w systemach GIS:

- Oś pozioma $X$: **długość geograficzna (longitude / lon)**
- Oś pionowa $Y$: **szerokość geograficzna (latitude / lat)**

Zdefiniowano punkt reprezentujący opolski Rynek oraz poligon testowy:

```python
from shapely.geometry import Point, Polygon

rynek = Point(17.9226, 50.6681)  # (x, y) = (lon, lat)
square = Polygon([(17.92, 50.66), (17.93, 50.66), (17.93, 50.67), (17.92, 50.67)])

print(rynek)
print(square.area)
print(square.contains(rynek))
```

**Uzyskane wyniki:**

- `POINT (17.9226 50.6681)`
- Pole powierzchni: `0.00010000000000003127` (wyrażone w jednostkach współrzędnych, czyli stopniach kwadratowych $\text{deg}^2$, co obrazuje konieczność późniejszej transformacji do układu metrycznego przy liczeniu pól i odległości fizycznych),
- Relacja przestrzenna `contains`: `True` (punkt leży wewnątrz zdefiniowanego poligonu).

Następnie zintegrowano geometrie z tabelą atrybutową w strukturę **GeoDataFrame**, przypisując układ odniesienia **WGS 84 (EPSG:4326)**:

```python
import geopandas as gpd

places = gpd.GeoDataFrame(
    {"name": ["Rynek", "Square"]},
    geometry=[rynek, square],
    crs="EPSG:4326",
)
```

Weryfikacja właściwości obiektu potwierdziła typ `GeoDataFrame`, zdefiniowany układ współrzędnych `EPSG:4326` oraz typy geometrii `Point` i `Polygon`.

---

## 4. Krok A2: Wczytanie i inspekcja surowych danych pomiarowych

Załadowano plik `dane/pm25_sensors.csv` zawierający surowe pomiary czujników za pomocą biblioteki **Pandas** z parsowaniem kolumny czasowej:

```python
import pandas as pd

raw = pd.read_csv("dane/pm25_sensors.csv", parse_dates=["measured_at"])
print(raw.shape)
raw[["lat", "lon", "pm25"]].describe()
raw.isna().sum()
raw.duplicated().sum()
```

**Diagnoza jakościowa surowego zbioru (Punkt kontrolny A2):**

- Zbiór wejściowy: **145 wierszy** i **5 kolumn** (`sensor_id`, `measured_at`, `lat`, `lon`, `pm25`).
- Wykryte anomalie:
  1. **Duplikaty:** 5 w pełni zduplikowanych rekordów wynikających z wielokrotnego przesłania tego samego pakietu przez sensor.
  2. **Braki współrzędnych:** 3 braki danych w kolumnie `lat` oraz 4 braki w kolumnie `lon` (uniemożliwiające lokalizację na mapie).
  3. **Błąd zamiany współrzędnych ($X \leftrightarrow Y$):** minimalna wartość `lat` wynosiła ok. $17,87^\circ$. Ponieważ Opole leży na szerokości geograficznej ok. $50,7^\circ$, a w Polsce długość to ok. $14^\circ - 24^\circ$, część czujników miała zamienione wartości szerokości i długości geograficznej.
  4. **Wartości ujemne:** minimalna wartość `pm25` wynosiła `-999`. Jest to kod błędu sensora (np. awaria optycznego toru pomiarowego), a nie rzeczywiste stężenie pyłu.

---

## 5. Krok A3: Czyszczenie danych (Data Cleaning Pipeline)

Zaprojektowano i zaimplementowano czteroetapowy potok czyszczenia, rejestrując liczbę wierszy na każdym etapie w słowniku kontrolnym `report`:

```python
df = raw.copy()
report = {"input rows": len(df)}

# 1. Usunięcie duplikatów
df = df.drop_duplicates()
report["after drop_duplicates"] = len(df)

# 2. Usunięcie wierszy bez współrzędnych lokalizacyjnych
df = df.dropna(subset=["lat", "lon"])
report["after dropna"] = len(df)

# 3. Naprawa zamienionych współrzędnych (lat: 14-24.5 i lon: 49-55)
swapped = df["lat"].between(14, 24.5) & df["lon"].between(49, 55)
df.loc[swapped, ["lat", "lon"]] = df.loc[swapped, ["lon", "lat"]].to_numpy()
report["swapped rows fixed"] = int(swapped.sum())

# 4. Usunięcie kodów błędu aparatury (pm25 < 0)
df = df[df["pm25"] >= 0].reset_index(drop=True)
report["clean rows"] = len(df)
```

> [!IMPORTANT]
> **Dlaczego użyto `.to_numpy()` w punkcie 3?**
> Pandas domyślnie dopasowuje wartości według etykiet kolumn (indeksów). Bez konwersji do tablicy NumPy (`.to_numpy()`), przypisanie kolumn `["lon", "lat"]` do `["lat", "lon"]` dopasowałoby je ponownie po nazwach, anulując operację zamiany.

> [!NOTE]
> **Uzasadnienie odmiennej obsługi błędów:**
>
> - Wiersze z odwróconymi współrzędnymi naprawiono, ponieważ sam pomiar stężenia oraz fizyczna lokalizacja były w pełni poprawne – błąd tkwił wyłącznie w kolejności zapisu kolumn w protokole transmisji.
> - Wiersze z wartością `-999` bezwzględnie usunięto, ponieważ wartość ta nie reprezentuje żadnego pomiaru fizycznego, lecz status awarii sprzętowej; włączenie jej do analizy zafałszowałoby średnie stężenie PM2.5 w mieście.

**Raport z przebiegu czyszczenia:**
| Etap czyszczenia | Liczba wierszy | Różnica |
| :--- | :---: | :---: |
| Wejściowa liczba wierszy | **145** | – |
| Po usunięciu duplikatów | **140** | -5 |
| Po usunięciu braków współrzędnych | **134** | -6 |
| Liczba naprawionych wierszy z odwróconymi współrzędnymi | **3** | (naprawiono) |
| **Ostateczna liczba czystych wierszy** | **130** | -4 (usunięto -999) |

**Statystyki opisowe wyczyszczonego zbioru:**

```text
          lat     lon    pm25
count  130.00  130.00  130.00
mean    50.67   17.92   18.86
std      0.03    0.04    5.19
min     50.62   17.79    9.20
25%     50.65   17.90   14.65
50%     50.67   17.92   19.00
75%     50.69   17.94   22.22
max     50.77   18.02   35.30
```

Po procesie czyszczenia szerokość geograficzna mieści się w realistycznym zakresie dla Opola ($50,62^\circ - 50,77^\circ$), a stężenia PM2.5 przyjmują fizycznie prawdopodobne wartości od $9,20$ do $35,30\ \mu\text{g/m}^3$.

---

## 6. Krok A4: Konstrukcja warstwy wektorowej (GeoDataFrame) i eksploracja mapowa

Na bazie oczyszczonej tabeli utworzono docelową warstwę wektorową `sensors`, łącząc współrzędne w obiekty punktowe:

```python
sensors = gpd.GeoDataFrame(
    df,
    geometry=gpd.points_from_xy(df["lon"], df["lat"]),  # X = lon, Y = lat!
    crs="EPSG:4326",
)

print(sensors.geom_type.unique())
print(sensors.crs)
print(sensors.total_bounds)
```

**Weryfikacja parametrów przestrzennych (Punkt kontrolny A4):**

- **Typ geometrii:** `Point`
- **Układ współrzędnych (CRS):** `EPSG:4326` (WGS 84)
- **Zasięg przestrzenny (total_bounds):** `[17.785622, 50.616593, 18.020378, 50.767895]`
  Wszystkie punkty pomiarowe mieszczą się w granicach administracyjnych Opola.

Wygenerowano interaktywną mapę podglądową za pomocą metody `.explore()` ze skalą barwną zależną od stężenia pyłu PM2.5:

```python
sensors.explore(column="pm25", cmap="Oranges", marker_kwds={"radius": 6})
```

Wszystkie czujniki poprawnie uplasowały się na siatce miejskiej Opola, tworząc gotową warstwę do dalszych analiz relacji przestrzennych.

---

## 7. Krok A5: Układy współrzędnych (CRS) i znaczenie transformacji

Jednym z kluczowych zagadnień w GIS jest świadomy dobór układu odniesienia. W układzie geograficznym `EPSG:4326` współrzędne wyrażone są w stopniach kątowych, co uniemożliwia bezpośrednie obliczanie odległości i buforów metrycznych:

```python
a, b = sensors.geometry.iloc[0], sensors.geometry.iloc[1]
a.distance(b)
# Wynik: 0.0573... [stopnie]
```

Wykonano reprojekcję czujników do polskiego państwowego układu metrycznego **PL-1992 (EPSG:2180)**:

```python
sensors_m = sensors.to_crs("EPSG:2180")
sensors_m.geometry.iloc[0].distance(sensors_m.geometry.iloc[1])
# Wynik: 4296.2 m (~4.3 km)
```

Przeprowadzono również eksperyment porównawczy zniekształceń skali na trasie Gdańsk–Zakopane (odległość geodezyjna obliczona na elipsoidzie WGS84: **562,3 km**):

- **EPSG:3857 (Web Mercator):** odległość wyliczona wynosi **908,1 km** – błąd zniekształcenia wynosi aż **+61,50%** (skutek rzutu walcowego wiernokątnego rozciągającego obszary w wyższych szerokościach geograficznych).
- **EPSG:2180 (Układ 1992):** odległość wyliczona wynosi **561,8 km** – błąd wynosi zaledwie **-0,09%**. Potwierdza to konieczność używania układu `EPSG:2180` do analiz przestrzennych na obszarze Polski.

---

## 8. Krok A6: Poligony – Granice dzielnic Opola

Wczytano drugą warstwę wektorową przedstawiającą granice 29 dzielnic Opola (`dane/opole_districts.geojson`) za pomocą funkcji `gpd.read_file`:

```python
districts = gpd.read_file("dane/opole_districts.geojson")
print(len(districts), districts.crs)
districts = districts.to_crs("EPSG:2180")
districts["area_km2"] = districts.area / 1e6        # m² -> km²

print(f"City area: {districts['area_km2'].sum():.1f} km²")
districts.sort_values("area_km2", ascending=False).head()
```

**Wyniki i punkt kontrolny A6:**

- **Liczba obiektów:** 29 dzielnic.
- **CRS wejściowy:** `EPSG:4326` $\to$ przekształcony do `EPSG:2180`.
- **Całkowita powierzchnia Opola:** **148,8 km²**.
- **Ekstrema powierzchniowe:**
  - Największa dzielnica: **Czarnowąsy** (ok. 14,1 km²),
  - Najmniejsza dzielnica: **Borki** (ok. 0,66 km²).

Wygenerowano statyczną mapę kartograficzną powierzchni dzielnic w bibliotece `matplotlib`:

```python
import matplotlib.pyplot as plt

ax = districts.plot(column="area_km2", cmap="Blues", legend=True,
                    edgecolor="white", figsize=(7, 7))
ax.set_title("Opole districts by area [km²]")
ax.set_axis_off()
plt.show()
```

> [!NOTE]
> **Dlaczego liczenie `.area` w EPSG:4326 jest błędem?**
> Bez uprzedniej konwersji do układu metrycznego (`.to_crs("EPSG:2180")`), metoda `.area` wylicza pole powierzchni w stopniach kwadratowych ($\text{deg}^2$). Ponieważ długość 1 stopnia długości geograficznej zależy od szerokości geograficznej ($\cos(\text{lat})$), stopień kwadratowy nie ma stałego wymiaru fizycznego. GeoPandas generuje wówczas ostrzeżenie o rzucie geograficznym.

---

## 9. Krok A7: Złączenie przestrzenne (Spatial Join)

Aby przypisać poszczególne czujniki do dzielnic, w których się znajdują, wykonano złączenie przestrzenne `gpd.sjoin` z predykatem `within`:

```python
joined = gpd.sjoin(
    sensors_m,                                   # warstwa punktowa
    districts[["name", "geometry"]],             # warstwa poligonowa
    how="left",
    predicate="within",                          # punkt leży wewnątrz poligonu
)
print(len(joined), joined["name"].isna().sum())
```

**Wyniki złączenia:**

- Wszystkie **130 czujników** zostało pomyślnie zlokalizowanych wewnątrz granic administracyjnych (0 braków dopasowania).
- **3 dzielnice** nie posiadają ani jednego czujnika.

Następnie zagregowano pomiary pyłu PM2.5 per dzielnica, wyliczając średnią oraz liczbę czujników:

```python
stats = (joined.groupby("name")
               .agg(pm25_mean=("pm25", "mean"), sensor_count=("pm25", "size"))
               .round(1))

summary = districts.merge(stats, on="name", how="left")
reliable = summary[summary["sensor_count"] >= 3]
reliable.nlargest(3, "pm25_mean")[["name", "pm25_mean", "sensor_count"]]
```

**Najbardziej zanieczyszczone dzielnice (przy próbie min. 3 czujników):**

1. **Stare Miasto:** średnie PM2.5 = **29,6 $\mu\text{g/m}^3$** (3 czujniki)
2. **Śródmieście:** średnie PM2.5 = **26,8 $\mu\text{g/m}^3$** (7 czujników)
3. **Zaodrze:** średnie PM2.5 = **22,5 $\mu\text{g/m}^3$** (6 czujników)

Dla celów wizualizacji utworzono interaktywny kartogram z podziałem kwantylowym (`scheme="quantiles"`, $k=5$), gdzie dzielnice bez czujników oznaczono kolorem szarym (`missing_kwds`).

---

## 10. Krok A8: Analiza strefowa wokół szkół podstawowych (Bufory)

Wczytano warstwę szkół podstawowych (`dane/opole_schools.geojson`) i utworzono strefy buforowe o promieniu **500 metrów** w układzie metrycznym `EPSG:2180`:

```python
schools = gpd.read_file("dane/opole_schools.geojson").to_crs("EPSG:2180")

buffers = schools.copy()
buffers["geometry"] = schools.buffer(500)  # promień 500 m w układzie metrycznym
```

Przeprowadzono analizę ekspozycji szkół na zanieczyszczenia poprzez złączenie przestrzenne czujników ze strefami buforowymi:

```python
near_schools = gpd.sjoin(sensors_m, buffers[["name", "geometry"]], predicate="within")

print("schools:", len(schools))
print("schools with a sensor within 500 m:", near_schools["name"].nunique())
print(f"mean PM2.5 near schools: {near_schools['pm25'].mean():.1f} µg/m³")
```

Wyznaczono również stopień pokrycia powierzchni całego miasta przez strefy przyszkolne:

```python
city = districts.union_all()
coverage = buffers.union_all().intersection(city).area / city.area
print(f"{coverage:.1%} of the city")
```

**Wyniki analizy przyszkolnej (Punkt kontrolny A8):**

- Liczba szkół podstawowych: **19**.
- Szkoły z czujnikiem w promieniu 500 m: **11** (58% placówek).
- Średnie stężenie PM2.5 w strefie przyszkolnej: **22,3 $\mu\text{g/m}^3$** (wyższe niż średnia dla całego miasta wynosząca 18,9 $\mu\text{g/m}^3$, ze względu na lokalizację placówek w gęściej zabudowanych rejonach).
- Pokrycie miasta buforami 500 m: **9,5%** powierzchni Opola.

Zbudowano wielowarstwową interaktywną mapę Folium integrującą granice dzielnic, bufory 500 m oraz lokalizacje czujników.

---

## 11. Krok A9: Eksport danych przestrzennych i weryfikacja w QGIS

Końcowe rezultaty analiz wektorowych wyeksportowano do standardowych formatów wymiany danych przestrzennych:

1. **GeoPackage (`wyniki/pm25.gpkg`):** wielowarstwowy plik geobazy zawierający warstwę `districts` (z agregatami PM2.5) oraz warstwę `sensors` w metrycznym układzie `EPSG:2180`.
2. **GeoJSON (`wyniki/pm25_districts.geojson`):** zreprojektowany do `EPSG:4326`, dedykowany do publikacji internetowej i aplikacji webowych.

```python
from pathlib import Path

out = Path("wyniki")
out.mkdir(exist_ok=True)

# Eksport warstw do GeoPackage (EPSG:2180)
summary.to_file(out / "pm25.gpkg", layer="districts", driver="GPKG")
sensors_m.to_file(out / "pm25.gpkg", layer="sensors", driver="GPKG")

# Eksport do GeoJSON (EPSG:4326)
summary.to_crs("EPSG:4326").to_file(out / "pm25_districts.geojson", driver="GeoJSON")

print(gpd.list_layers(out / "pm25.gpkg"))
```

### Weryfikacja integralności pliku w środowisku QGIS

Zgodnie z poleceniem laboratoryjnym:

> _Otwórz `wyniki/pm25.gpkg` w QGIS (przeciągnij plik do okna) i sprawdź, czy obie warstwy są na miejscu._

Wygenerowany plik `pm25.gpkg` załadowano do projektu QGIS. Obie warstwy wektorowe (`pm25 — districts` oraz `pm25 — sensors`) zaimportowały się poprawnie, zachowując geometrię, spójność topologiczną oraz pełne tabele atrybutowe na podkładzie OpenStreetMap.

![Weryfikacja warstw GeoPackage w programie QGIS](screenshots/Screenshot%202026-11-08%20at%2018.25.41.png)

_Rysunek 1. Warstwy `pm25 — districts` oraz `pm25 — sensors` z pliku GeoPackage otwarte w programie QGIS na tle OpenStreetMap._

---

## 12. Podsumowanie części A (Pipeline wektorowy)

Zrealizowany etap wektorowy podsumowuje kanoniczny przepływ pracy (_data processing pipeline_) w bibliotece **GeoPandas**:

$$\text{read\_csv} \longrightarrow \text{czyszczenie danych} \longrightarrow \text{GeoDataFrame} \longrightarrow \text{to\_crs (EPSG:2180)} \longrightarrow \text{read\_file} \longrightarrow \text{sjoin / buffer} \longrightarrow \text{to\_file (GPKG)}$$

---

# Część B: Analiza danych rastrowych w bibliotece Rasterio (`lab_rasterio.ipynb`)

## 13. Krok B1: Dane rastrowe, GeoTIFF i wskaźnik NDVI

W drugiej części ćwiczenia przeanalizowano zobrazowanie satelitarne **Sentinel-2 L2A** z dnia **6 października 2026 r.** obejmujące Opole i okolice, zapisane w dwóch plikach GeoTIFF:

- `dane/S2_L2A_2026-10-06_B04.tiff` — pasmo czerwone **Red** (665 nm),
- `dane/S2_L2A_2026-10-06_B08.tiff` — pasmo bliskiej podczerwieni **NIR** (842 nm).

Celem analizy jest obliczenie znormalizowanego różnicowego wskaźnika wegetacji **NDVI** (_Normalized Difference Vegetation Index_):

$$\text{NDVI} = \frac{\text{NIR} - \text{Red}}{\text{NIR} + \text{Red}} = \frac{B08 - B04}{B08 + B04} \in [-1, 1]$$

| Zakres NDVI     | Typowe pokrycie terenu    |
| --------------- | ------------------------- |
| $< 0$           | Woda                      |
| $0 - 0{,}2$     | Zabudowa, goła gleba      |
| $0{,}2 - 0{,}5$ | Rzadka / słaba roślinność |
| $> 0{,}5$       | Gęsta, zdrowa roślinność  |

---

## 14. Krok B2: Otwarcie pliku i inspekcja metadanych

Otwarcie pliku przez `rasterio.open(...)` w menedżerze kontekstu `with` odczytuje nagłówki GeoTIFF bez ładowania macierzy pikseli do pamięci:

```python
with rasterio.open(B04_PATH) as src:
    print(src.profile)
    print(src.crs, src.bounds, src.res, src.shape, src.nodata)
```

**Metadane wejściowe obu pasm (`B04` i `B08`):**

- **Układ współrzędnych (CRS):** `EPSG:4326` (WGS 84, jednostki w stopniach)
- **Rozmiar macierzy (`shape`):** `1717` wierszy × `2083` kolumn
- **Rozdzielczość piksela (`res`):** `8.999e-05° × 8.998e-05°` (na szerokości geograficznej Opola ok. `6,3 m × 9,9 m` — piksel nie jest kwadratowy)
- **Zasięg (`bounds`):** `lon ∈ [17.895355, 18.082809]`, `lat ∈ [50.591084, 50.745580]`
- **Typ danych i kompresja:** `float32`, `compress='deflate'`, `nodata=None`

---

## 15. Krok B3: Transformacja afiniczna i mapowanie piksel ↔ współrzędne

Transformacja afiniczna (`src.transform`) przelicza indeksy macierzy `(row, col)` na współrzędne terenowe `(x, y)`. Z użyciem `rasterio.warp.transform` oraz odczytu okienkowego `window=` zweryfikowano pojedynczy piksel kontrolny w Opolu:

```python
lon, lat = 17.989082, 50.668332
(x,), (y,) = transform("EPSG:4326", red_src.crs, [lon], [lat])
row, col = red_src.index(x, y)
red_val = red_src.read(1, window=((row, row + 1), (col, col + 1)))[0, 0]
nir_val = nir_src.read(1, window=((row, row + 1), (col, col + 1)))[0, 0]
```

**Wynik dla punktu kontrolnego `(17.989082, 50.668332)`:**

- Indeks piksela: `(row, col) = (858, 1041)`
- Wartości spektralne: `Red (B04) = 0.0620`, `NIR (B08) = 0.2895`
- Lokalny wskaźnik roślinności: `NDVI = 0.647` (gęsta roślinność).

---

## 16. Krok B4: Odczyt pasm, statystyki i histogramy

Pasma wczytano jako tablice maskowane (`masked=True`, `float32`), dzięki czemu ewentualne piksele `nodata` są automatycznie pomijane w statystykach:

- **Red (`B04`):** `min = 0.0000`, `max = 1.5514`, `mean = 0.0571`, `nodata_px = 0`
- **NIR (`B08`):** `min = 0.0000`, `max = 1.5103`, `mean = 0.2013`, `nodata_px = 0`

Wygenerowano mapy obu pasm oraz histogramy rozkładu odbicia spektralnego.

---

## 17. Krok B5: Wspólna siatka w metrach (reprojekcja do `EPSG:2180`)

Chociaż oba pasma źródłowe mają identyczną siatkę w `EPSG:4326`, do dokładnego obliczania powierzchni (w $\text{m}^2$ i $\text{ha}$) oraz łączenia z warstwami wektorowymi w układzie PUWG 1992 niezbędna jest reprojekcja do metrycznego układu **`EPSG:2180`** o rozdzielczości `10 × 10 m`:

```python
dst_transform, width, height = calculate_default_transform(
    red_src.crs, "EPSG:2180", red_src.width, red_src.height, *red_src.bounds, resolution=10
)
```

Przeprowadzono reprojekcję dwuliniową (`Resampling.bilinear`) z wartością `dst_nodata=np.nan`:

- **Przed reprojekcją:** `EPSG:4326`, kształt `(1717, 2083)`, piksel w stopniach
- **Po reprojekcji:** `EPSG:2180`, kształt `(1736, 1348)`, piksel kwadratowy `10.0 × 10.0 m` ($100\text{ m}^2$).

---

## 18. Krok B6: Obliczenie wskaźnika NDVI

Wskaźnik NDVI obliczono operacją wektorową NumPy w bloku `np.errstate(divide="ignore", invalid="ignore")`, maskując wartości nieprawidłowe (`NaN` / `inf`):

- **Średnie NDVI:** `0.553`
- **Mediana NDVI:** `0.626`
- **Zakres wartości:** `-0.636 ... 0.931`
- **Ekstrema:** minimum `-0.636` w pikselu `(57, 353)` (woda), maksimum `0.931` w pikselu `(174, 1117)` (gęsta roślinność).

Zwizualizowano mapę przestrzenną NDVI w palecie `RdYlGn` oraz histogram rozkładu wartości z zaznaczonymi progami klasyfikacji (`0.0`, `0.2`, `0.5`).

---

## 19. Krok B7: Klasyfikacja pokrycia terenu i powierzchnie w hektarach

Za pomocą `np.digitize` podzielono wartości NDVI na 4 klasy pokrycia terenu i przeliczono zliczenia pikseli (`np.bincount`) na hektary ($1\text{ piksel} = 100\text{ m}^2 = 0{,}01\text{ ha}$):

| Klasa pokrycia terenu                  | Liczba pikseli | Powierzchnia [ha] | Udział [%] |
| -------------------------------------- | -------------: | ----------------: | ---------: |
| `0: Woda (< 0.0)`                      |         17 876 |            178,76 |      0,84% |
| `1: Zabudowa / goła gleba (0.0 – 0.2)` |        247 865 |          2 478,65 |     11,71% |
| `2: Rzadka roślinność (0.2 – 0.5)`     |        544 724 |          5 447,24 |     25,74% |
| `3: Gęsta roślinność (> 0.5)`          |      1 305 382 |         13 053,82 |     61,70% |

Ponad **61,7%** analizowanego obszaru stanowi gęsta roślinność, a obszary zabudowane i odkryta gleba zajmują **11,71%** powierzchni.

---

## 20. Krok B8: Statystyki strefowe dla dzielnic Opola (GeoPandas + Rasterio)

Połączono warstwę wektorową granic dzielnic (`dane/opole_districts.geojson`) w układzie `EPSG:2180` z rastrem NDVI przy użyciu funkcji `rasterio.mask.mask(..., crop=True, nodata=np.nan)`. Dla każdego poligonu obliczono pokrycie sceną (`coverage`) oraz średnie NDVI (`ndvi_mean`).

**Wyniki analizy strefowej:**

- **12 z 29 dzielnic** ma pokrycie zasięgiem sceny poniżej `50%` — zostały one wykluczone z rankingu jako niereprezentatywne (oznaczone kreskowaniem na mapie).
- Wśród dzielnic o pełnym lub wysokim pokryciu najwyższym średnim wskaźnikiem zieleni charakteryzują się:
  1. **Grudzice:** `ndvi_mean = 0,717` (`coverage = 100,0%`)
  2. **Nadodrze:** `ndvi_mean = 0,661` (`coverage = 100,1%`)
  3. **Malina:** `ndvi_mean = 0,539` (`coverage = 100,0%`)
  4. **Grotowice:** `ndvi_mean = 0,509` (`coverage = 84,9%`)
  5. **Kolonia Gosławicka:** `ndvi_mean = 0,492` (`coverage = 100,0%`)

---

## 21. Krok B9: Zapis wyniku do GeoTIFF i weryfikacja

Wynikowy raster NDVI zapisano do pliku `wyniki/ndvi.tif` w typie `float32` (koniecznym dla wartości ułamkowych z przedziału $[-1, 1]$), z `nodata=np.nan` oraz bezstratną kompresją `deflate`:

```python
profile.update(
    crs=dst_crs, transform=dst_transform, width=width, height=height,
    dtype="float32", count=1, nodata=np.nan, compress="deflate",
)
with rasterio.open(OUT_DIR / "ndvi.tif", "w", **profile) as dst:
    dst.write(ndvi.filled(np.nan).astype("float32"), 1)
    dst.set_band_description(1, "NDVI")
```

Zapisano także statystyki dzielnic do pliku wektorowego `wyniki/ndvi_districts.geojson`. Ponowny odczyt `wyniki/ndvi.tif` potwierdził poprawność układu `EPSG:2180`, kształtu `(1736, 1348)`, opisu pasma (`NDVI`) oraz zakresu wartości (`-0.636 ... 0.931`).

---

## 22. Podsumowanie części B (Pipeline rastrowy)

Zrealizowany etap rastrowy podsumowuje kanoniczny przepływ pracy w bibliotece **Rasterio**:

$$\text{rasterio.open} \longrightarrow \text{read(masked=True)} \longrightarrow \text{reproject (EPSG:2180)} \longrightarrow \text{algebra NumPy (NDVI)} \longrightarrow \text{digitize} \longrightarrow \text{rasterio.mask} \longrightarrow \text{GeoTIFF}$$

---
