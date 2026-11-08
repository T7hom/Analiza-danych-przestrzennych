# Sprawozdanie z ćwiczenia: Analiza danych przestrzennych w bibliotece GeoPandas (Część A)

> [!NOTE]
> **Status dokumentu:** Wersja robocza (*Work in Progress*) – pełne opracowanie części wektorowej GeoPandas (kroki A1–A9) wraz z weryfikacją w QGIS. Część B (Rasterio) oraz Część C (Streamlit) zostaną dołączone po udostępnieniu danych rastrowych przez prowadzącego.

**Autor:** Tomasz Broda  
**Data wykonania:** 24 października – 8 listopada 2026 r.  
**Środowisko:** Python 3.12 (wirtualne środowisko `.lab-geo`)  
**Wykorzystane biblioteki:** GeoPandas 1.2.0, Shapely 2.2.0, Rasterio 1.5.2, Pandas, Folium / Matplotlib, Mapclassify, Pyproj  
**Obszar i dane:** Pomiary jakości powietrza (pył zawieszony PM2.5) z czujników naziemnych, granice 29 dzielnic Opola oraz lokalizacje szkół podstawowych  

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
* Oś pozioma $X$: **długość geograficzna (longitude / lon)**
* Oś pionowa $Y$: **szerokość geograficzna (latitude / lat)**

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
* `POINT (17.9226 50.6681)`
* Pole powierzchni: `0.00010000000000003127` (wyrażone w jednostkach współrzędnych, czyli stopniach kwadratowych $\text{deg}^2$, co obrazuje konieczność późniejszej transformacji do układu metrycznego przy liczeniu pól i odległości fizycznych),
* Relacja przestrzenna `contains`: `True` (punkt leży wewnątrz zdefiniowanego poligonu).

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
* Zbiór wejściowy: **145 wierszy** i **5 kolumn** (`sensor_id`, `measured_at`, `lat`, `lon`, `pm25`).
* Wykryte anomalie:
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
> * Wiersze z odwróconymi współrzędnymi naprawiono, ponieważ sam pomiar stężenia oraz fizyczna lokalizacja były w pełni poprawne – błąd tkwił wyłącznie w kolejności zapisu kolumn w protokole transmisji.
> * Wiersze z wartością `-999` bezwzględnie usunięto, ponieważ wartość ta nie reprezentuje żadnego pomiaru fizycznego, lecz status awarii sprzętowej; włączenie jej do analizy zafałszowałoby średnie stężenie PM2.5 w mieście.

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
* **Typ geometrii:** `Point`
* **Układ współrzędnych (CRS):** `EPSG:4326` (WGS 84)
* **Zasięg przestrzenny (total_bounds):** `[17.785622, 50.616593, 18.020378, 50.767895]`
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
* **EPSG:3857 (Web Mercator):** odległość wyliczona wynosi **908,1 km** – błąd zniekształcenia wynosi aż **+61,50%** (skutek rzutu walcowego wiernokątnego rozciągającego obszary w wyższych szerokościach geograficznych).
* **EPSG:2180 (Układ 1992):** odległość wyliczona wynosi **561,8 km** – błąd wynosi zaledwie **-0,09%**. Potwierdza to konieczność używania układu `EPSG:2180` do analiz przestrzennych na obszarze Polski.

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
* **Liczba obiektów:** 29 dzielnic.
* **CRS wejściowy:** `EPSG:4326` $\to$ przekształcony do `EPSG:2180`.
* **Całkowita powierzchnia Opola:** **148,8 km²**.
* **Ekstrema powierzchniowe:**
  * Największa dzielnica: **Czarnowąsy** (ok. 14,1 km²),
  * Najmniejsza dzielnica: **Borki** (ok. 0,66 km²).

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
* Wszystkie **130 czujników** zostało pomyślnie zlokalizowanych wewnątrz granic administracyjnych (0 braków dopasowania).
* **3 dzielnice** nie posiadają ani jednego czujnika.

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
* Liczba szkół podstawowych: **19**.
* Szkoły z czujnikiem w promieniu 500 m: **11** (58% placówek).
* Średnie stężenie PM2.5 w strefie przyszkolnej: **22,3 $\mu\text{g/m}^3$** (wyższe niż średnia dla całego miasta wynosząca 18,9 $\mu\text{g/m}^3$, ze względu na lokalizację placówek w gęściej zabudowanych rejonach).
* Pokrycie miasta buforami 500 m: **9,5%** powierzchni Opola.

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
> *Otwórz `wyniki/pm25.gpkg` w QGIS (przeciągnij plik do okna) i sprawdź, czy obie warstwy są na miejscu.*

Wygenerowany plik `pm25.gpkg` załadowano do projektu QGIS. Obie warstwy wektorowe (`pm25 — districts` oraz `pm25 — sensors`) zaimportowały się poprawnie, zachowując geometrię, spójność topologiczną oraz pełne tabele atrybutowe na podkładzie OpenStreetMap.

![Weryfikacja warstw GeoPackage w programie QGIS](screenshots/Screenshot%202026-11-08%20at%2018.25.41.png)

*Rysunek 1. Warstwy `pm25 — districts` oraz `pm25 — sensors` z pliku GeoPackage otwarte w programie QGIS na tle OpenStreetMap.*

---

## 12. Podsumowanie części A (Pipeline wektorowy)

Zrealizowany etap wektorowy podsumowuje kanoniczny przepływ pracy (*data processing pipeline*) w bibliotece **GeoPandas**:

$$\text{read\_csv} \longrightarrow \text{czyszczenie danych} \longrightarrow \text{GeoDataFrame} \longrightarrow \text{to\_crs (EPSG:2180)} \longrightarrow \text{read\_file} \longrightarrow \text{sjoin / buffer} \longrightarrow \text{to\_file (GPKG)}$$

---

## 13. Status realizacji i kolejne kroki

Część wektorowa (Część A) została sfinalizowana w 100%. Kolejne etapy realizacji ćwiczenia obejmują:

1. **Część B: Rasterio (`lab_rasterio.ipynb`):**
   * Wczytanie i eksploracja zobrazowań satelitarnych Sentinel-2 L2A (pasma B04 – Red oraz B08 – NIR).
   * Interpretacja transformacji afinicznej i wartości NoData.
   * Reprojekcja rastrów do `EPSG:2180` i kalkulacja wskaźnika wegetacji NDVI.
   * Statystyki strefowe (agregacja wartości NDVI w granicach dzielnic Opola).
   *(Oczekiwanie na przekazanie plików GeoTIFF przez prowadzącego).*
2. **Część C: Aplikacja Streamlit:**
   * Połączenie analiz wektorowych (PM2.5) i rastrowych (NDVI) w interaktywny dashboard analityczny w bibliotece Streamlit.
