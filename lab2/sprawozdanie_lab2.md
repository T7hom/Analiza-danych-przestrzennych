# Sprawozdanie z ćwiczenia: Analiza danych przestrzennych w bibliotece GeoPandas (Część A)

> [!NOTE]
> **Status dokumentu:** Wersja robocza (_Work in Progress_) – opis zrealizowanych etapów części wektorowej (A1–A4). Kolejne etapy (A5–A7, część rastrowa Rasterio oraz aplikacja Streamlit) są w trakcie realizacji.

**Autor:** Tomasz Broda
**Data wykonania:** 24 października 2026 r.
**Środowisko:** Python 3.12 (wirtualne środowisko `.lab-geo`)
**Wykorzystane biblioteki:** GeoPandas 1.2.0, Shapely 2.2.0, Rasterio 1.5.2, Pandas, Folium / Matplotlib
**Obszar i dane:** Pomiary jakości powietrza (pył zawieszony PM2.5) z czujników na obszarze miasta Opola

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
