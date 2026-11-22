# Sprawozdanie z ćwiczenia: Wykrywanie obiektów na zdjęciach lotniczych — Simple Cars Detector (Lab 3)

**Autor:** Tomasz Broda  
**Data wykonania:** 21 listopada 2026 r.  
**Środowisko:** Python 3.12 (wirtualne środowisko `lab3/simple_cars_detector/venv`)  
**Wykorzystane biblioteki:** Streamlit, Ultralytics (`yolo11n-obb.pt`), Rasterio (`MemoryFile`), Plotly, Pandas, NumPy, Pillow  
**Obszar i dane:** Ortofotomapa fragmentu centrum Opola (`data/centrum_10cm.tif`, rozdzielczość `0,10 m/px`, układ `EPSG:2180`)

---

## 1. Cel ćwiczenia i architektura aplikacji

Celem laboratorium było zbudowanie krok po kroku aplikacji webowej w **Streamlit** (`simple_cars_detector/app.py`), która:
1. Wczytuje wielkoformatowe zdjęcia lotnicze / ortofotomapy w formacie **GeoTIFF** z zachowaniem georeferencji.
2. Udostępnia interaktywny podgląd obrazu z możliwością płynnego przybliżania i przesuwania (**Plotly**).
3. Wykrywa pojazdy za pomocą modelu detekcji zorientowanych ramek (**YOLO11n-obb**, zbiór DOTA) w trybie kafelkowym z nakładką.
4. Pozwala dynamicznie filtrować wyniki progiem pewności i prezentuje tabelę detekcji wraz ze współrzędnymi mapowymi (`EPSG:2180`).

---

## 2. Etap B1: Szkielet aplikacji i model wykonania Streamlit

Utworzono aplikację `app.py` z konfiguracją strony `st.set_page_config(page_title="Simple Cars Detector", page_icon="🚗", layout="wide")`.

Kluczowa zasada działania Streamlit: każda interakcja użytkownika (wgranie pliku, kliknięcie przycisku, zmiana suwaka) powoduje ponowne wykonanie całego skryptu od góry do dołu. Z tego względu:
* kosztowne operacje dekodowania rastra i kompresji podglądu buforuje się przez `@st.cache_data`,
* obiekt modelu sieci neuronowej ładuje się jednokrotnie przez `@st.cache_resource`,
* wyniki detekcji przechowuje się w `st.session_state`, aby nie znikały po przesunięciu suwaka progu pewności.

---

## 3. Etap B2: Wczytywanie GeoTIFF, metadane i interaktywny podgląd

### Odczyt z pamięci i normalizacja (`read_geotiff`, `to_uint8`)
Plik przesłany przez `st.file_uploader` trafia do aplikacji jako bufor bajtów, dlatego odczyt zrealizowano przez `rasterio.io.MemoryFile`:
* Pobierane są maksymalnie pierwsze 3 pasma (`RGB`); obrazy jednokanałowe powielane są do 3 kanałów.
* Funkcja `to_uint8` rozciąga obrazy inne niż 8-bitowe (np. 16-bitowe lub zmiennoprzecinkowe) w zakresie 2.–98. percentyla.

**Odczytane metadane dla `data/centrum_10cm.tif`:**
* **Rozmiar obrazu:** `8000 × 8000 px`, `3` pasma (`uint8`)
* **Układ współrzędnych (CRS):** `EPSG:2180` (PL-1992, jednostki w metrach)
* **Rozmiar piksela:** `0,10 × 0,10 m`
* **Obszar w terenie:** $8000 \cdot 0{,}10\text{ m} = 800\text{ m}$ boku, czyli **$800 \times 800\text{ m}$ ($64\text{ ha}$)**. Typowy samochód osobowy ($\approx 4{,}5\text{ m}$) ma długość ok. **45 pikseli**.

### Zadanie dodatkowe: współrzędne narożników w panelu bocznym
Zgodnie z zadaniem z etapu B2 w panelu bocznym (`st.sidebar`) dopisano wyznaczanie narożników obrazu przy użyciu transformacji afinicznej `meta["transform"]`:
* **Lewy górny narożnik `(0, 0)`:** `(423584.00, 312174.00)`
* **Prawy dolny narożnik `(8000, 8000)`:** `(424384.00, 311374.00)`

### Interaktywny podgląd Plotly (`image_source`, `interactive_view`)
Aby uniknąć przesyłania surowego obrazu $8000 \times 8000 \times 3$ (~192 MB) do przeglądarki, funkcję `image_source` wykorzystano do zmniejszenia kopii podglądowej do maks. `3000 px` (`JPEG`, jakość 85) i wyliczenia współczynnika `scale` (`dx`, `dy`), dzięki czemu osie wykresu Plotly pozostają w pikselach oryginalnego rastra.

---

## 4. Etap B3: Dobór i ładowanie modelu YOLO

Zastosowano model **`yolo11n-obb.pt`** (wytrenowany na zbiorze lotniczo-satelitarnym **DOTA**), który zwraca obrócone ramki (*Oriented Bounding Boxes*) dopasowane do kątów parkowania pojazdów. Model ładowany jest raz przez `@st.cache_resource`.

Analiza całego obrazu `8000 × 8000 px` naraz jest nieskuteczna (przed predykcją sieć skaluje wejście do ok. 640–1024 px, przez co samochód o długości 45 px kurczy się do ~5–6 px). Z tego względu w etapie B4 zastosowano detekcję kafelkową w natywnej rozdzielczości modelu.

---

## 5. Etap B4: Kafelkowanie, detekcja, próg pewności i współrzędne mapowe

### Kafelkowanie z nakładką (`tiles`, `detect_tile`)
* Rozmiar kafelka: `TILE = 1024 px`, zakładka: `OVERLAP = 128 px` (krok `896 px`).
* Dla ortofotomapy `8000 × 8000 px` generowanych jest **$9 \times 9 = 81$ kafelków**.
* Predykcja odbywa się z `imgsz=1024`, `conf=0.1` (`MIN_CONF`) oraz `agnostic_nms=True` (aby jeden pojazd nie był wykrywany podwójnie jako `small vehicle` i `large vehicle`).
* Zostawiane są wyłącznie klasy `{"small vehicle", "large vehicle"}`, a duplikaty na granicach kafelków usuwane są przez sprawdzenie, czy środek ramki `(cx, cy)` leży w wewnętrznej części kafelka (poza pasem `OVERLAP / 2 = 64 px` od krawędzi sąsiadujących z innymi kafelkami).

### Wizualizacja, filtrowanie i tabela wyników (`results_table`)
* Wszystkie ramki rysowane są jedną serią `go.Scatter` (rozdzielone `None`), a niewidoczne markery w środkach ramek wyświetlają klasę i pewność po najechaniu kursorem.
* Suwak **Próg pewności** (`0.10 – 0.95`, domyślnie `0.25`) filtruje listę z `st.session_state.boxes` bez ponownego uruchamiania modelu.
* Tabela wyników przelicza środki ramek z pikseli (`float64`) na współrzędne mapowe `meta["transform"] * (col, row)`.

### Wyniki detekcji dla `data/centrum_10cm.tif`
* **Wszystkie surowe detekcje (`conf >= 0.10`):** `1563` pojazdów
* **Detekcje dla domyślnego progu (`conf >= 0.25`):** **`1422` pojazdów**, w tym:
  * `small vehicle`: **`1391`**
  * `large vehicle`: **`31`**
* **Przykładowy rekord z tabeli wyników:** `small vehicle`, pewność `0.82`, `(kolumna, wiersz) = (177, 605)`, współrzędne w `EPSG:2180`: `x = 423601.67 m`, `y = 312113.49 m`.

---

## 6. Podsumowanie

Aplikacja `simple_cars_detector/app.py` realizuje wszystkie etapy instrukcji **B1–B4** wraz z zadaniem dodatkowym z etapu B2. Połączenie kafelkowania z nakładką (`1024 px` / `128 px`) i buforowania (`@st.cache_data`, `@st.cache_resource`, `st.session_state`) pozwala na płynną, interaktywną analizę dużej ortofotomapy (`8000 × 8000 px`) i uzyskanie `1422` detekcji pojazdów wraz z ich dokładnymi współrzędnymi w układzie `EPSG:2180`.
