# RAPORT – „Mona Lisa Problem” (aproksymacja obrazu prymitywami geometrycznymi)

## 1. Opis zagadnienia i cel projektu

Rozpatrywane zagadnienie dotyczy **aproksymacji (przybliżania) obrazu rastrowego** przy pomocy ograniczonego zbioru prostych prymitywów geometrycznych. Zamiast przechowywać obraz jako macierz pikseli, próbujemy odtworzyć go jako „malunek” składający się z wielu półprzezroczystych kształtów rysowanych warstwowo na płótnie.

**Cel projektu**: zaprojektować reprezentację rozwiązania oraz procedury optymalizacji, które automatycznie dobierają parametry kształtów tak, aby wygenerowany obraz był jak najbardziej podobny do obrazu docelowego (np. „Mona Lisa”). W projekcie wykorzystano metody inspirowane algorytmami ewolucyjnymi i lokalnym przeszukiwaniem: strategię ewolucyjną z elityzmem oraz podejście konstruktywne typu incremental hill climbing.

W praktyce jest to odmiana problemu znanego w literaturze popularnej jako **„Evolutionary Art / Mona Lisa Problem”**: zamiast klasycznej optymalizacji funkcji analitycznej optymalizujemy parametry grafiki generatywnej.

## 2. Definicja problemu optymalizacji

### 2.1. Reprezentacja rozwiązania (osobnik, gen)

Rozwiązanie kodujemy jako **osobnika** (ang. *individual*), czyli listę genów. Każdy gen opisuje pojedynczy **wielokąt foremny** rysowany na obrazie w trybie RGBA.

W implementacji gen ma postać:

- `[(x, y), d, angle, [r, g, b, a]]`

gdzie:

- `(x, y)` – współrzędne środka wielokąta w pikselach,
- `d` – promień/rozmiar (odległość wierzchołków od środka),
- `angle` – obrót w radianach,
- `[r, g, b, a]` – kolor z kanałem alfa (0–255).

Dodatkowo w wielu eksperymentach stały jest parametr `p` – liczba wierzchołków wielokąta (np. `p=3` trójkąt, `p=5` pięciokąt). Wtedy osobnik składa się z `t` genów opisujących `t` wielokątów tego samego typu.

### 2.2. Przestrzeń poszukiwań

Dla obrazu o wymiarach `W × H` (szerokość × wysokość), a przy stałej liczbie wielokątów `t`, przestrzeń poszukiwań można opisać jako zbiór wszystkich wektorów parametrów:

- pozycja: $x \in \{0,1,\dots,W-1\}$, $y \in \{0,1,\dots,H-1\}$,
- rozmiar: $d \in [d_{min}, d_{max}]$ (w implementacji domyślnie `d_min=5`, `d_max=200`),
- obrót: $angle \in [0, 2\pi)$ (w mutacjach może wyjść poza zakres, ale rysowanie przez sin/cos jest okresowe),
- kolor: $r,g,b \in \{0,\dots,255\}$,
- alfa: $a \in [a_{min}, a_{max}]$ (domyślnie `a_min=50`, `a_max=255`).

Cały osobnik to lista długości `t`, więc liczba zmiennych rośnie liniowo z `t`. Dla `t=100` mamy 100 kształtów i w praktyce bardzo wysokowymiarową optymalizację.

### 2.3. Funkcja celu (fitness)

Jako miarę niepodobieństwa użyto **sumy kwadratów różnic** (SSE – *sum of squared errors*) liczonej na obrazach w RGB:

$$
\mathrm{SSE}(I, \hat{I}) = \sum_{y=1}^{H} \sum_{x=1}^{W} \sum_{c \in \{R,G,B\}} \left(I_{y,x,c} - \hat{I}_{y,x,c}\right)^2
$$

gdzie:

- $I$ – obraz docelowy,
- $\hat{I}$ – obraz wygenerowany przez osobnika.

**Minimalizujemy** SSE, czyli im mniejsza wartość, tym lepsze dopasowanie.

W implementacji, aby uniknąć przepełnień dla `uint8`, różnice są liczone po rzutowaniu na typ całkowity większego zakresu (`int64`).

### 2.4. Funkcja dekodująca (renderowanie)

Dekodowanie polega na wyrenderowaniu osobnika na płótnie RGBA:

1. Tworzymy obraz bazowy `base_img` w RGBA z ustalonym kolorem tła.
2. Dla każdego genu wyznaczamy wierzchołki wielokąta foremnego.
3. Rysujemy wielokąt na przezroczystej warstwie i łączymy z obrazem bazowym przez kompozycję alfa (`alpha_composite`).
4. Na końcu konwertujemy do RGB i dopiero wtedy liczymy SSE.

Ważna konsekwencja: kolejność genów ma znaczenie (późniejsze kształty mogą przykrywać wcześniejsze).

## 3. Użyte algorytmy i metody optymalizacji

W projekcie zastosowano dwa podejścia:

1. **Evolution Strategy (ES) z elityzmem** – populacyjny algorytm ewolucyjny.
2. **Incremental / Constructive Hill Climbing (IHC)** – podejście konstruktywne (iteracyjne dokładanie kształtów), wzbogacone o lokalną optymalizację pojedynczego genu.

### 3.1. Evolution Strategy (ES) z elityzmem

#### Opis idei

Utrzymujemy populację `n` osobników. W każdej generacji:

- wybieramy `m` najlepszych osobników (najmniejszy SSE) – to elita/rodzice,
- tworzymy dzieci przez kopiowanie rodziców i wykonanie mutacji,
- dzieci zastępują pozostałe `n-m` miejsc w populacji,
- fitness dla nowych osobników jest przeliczany przez pełne renderowanie i SSE.

W implementacji liczba dzieci na rodzica jest wyznaczana przez:

- `per_parent_children = (n - m) // m`

czyli przez dzielenie całkowite. Jeśli `(n-m)` nie jest wielokrotnością `m`, w danej generacji część populacji może pozostać niepodmieniona.

#### Operator mutacji

Mutacja działa **in-place** na losowo wybranym genie osobnika i wybiera jeden z typów zmiany:

- przesunięcie środka `(x, y)` o losowy krok w zakresie `±pos_step`,
- zmiana rozmiaru `d` o losowy krok `±dist_step` (z przycięciem do `[d_min, d_max]`),
- zmiana kąta `angle` o losowy krok `±ang_step`,
- zmiana jednego kanału z `[r,g,b,a]` o `±col_step` (przycięcie do `[0,255]`).

#### Schemat działania (skrót)

1. Inicjalizacja populacji losowej.
2. Obliczenie fitness dla wszystkich.
3. Dla każdej generacji:
   - selekcja najlepszych `m`,
   - generowanie dzieci przez kopiowanie + mutację,
   - ocena dzieci,
   - (opcjonalnie) logowanie postępu.

### 3.2. Incremental / Constructive Hill Climbing (IHC)

#### Opis idei

W podejściu konstruktywnym nie optymalizujemy od razu wszystkich `t` genów. Zamiast tego:

- startujemy od obrazu tła,
- w każdej iteracji dokładamy **jeden** nowy wielokąt,
- przed zaakceptowaniem nowego wielokąta wykonujemy jego **lokalną optymalizację**, aby wpasował się w aktualne różnice między obrazem docelowym a aktualną bazą.

To podejście przypomina klasyczne „malowanie” obrazu: dokładamy kolejne pociągnięcia pędzla, każde dopasowane do bieżącego stanu.

#### Lokalna optymalizacja genu

Dla pojedynczego genu:

1. Bierzemy gen startowy (losowy).
2. Wykonujemy do `trials` prób:
   - mutujemy gen (tworzymy kandydata),
   - obliczamy zmianę w funkcji celu,
   - jeśli kandydat poprawia wynik, akceptujemy go jako nowy najlepszy.
3. Jeśli przez `patience` kolejnych prób nie ma poprawy, kończymy wcześniej.

#### Przyspieszenie: ocena przyrostowa na fragmencie (bbox)

Pełne SSE dla obrazu jest kosztowne, bo wymaga przetworzenia `H×W×3` wartości. W lokalnej optymalizacji genu zastosowano heurystykę:

- wyznaczamy **bounding box** wielokąta (z marginesem `pad`),
- SSE aktualizujemy przyrostowo tylko dla tego fragmentu:

$$
\mathrm{SSE}_{new} \approx \mathrm{SSE}_{best} - \mathrm{SSE}(\hat{I}_{patch}, I_{patch}) + \mathrm{SSE}(\hat{I}'_{patch}, I_{patch})
$$

Dzięki temu większość prób w lokalnym przeszukiwaniu jest tańsza, bo liczy SSE na wycinku obrazu.

## 4. Opis implementacji

### 4.1. Technologie i organizacja

Implementacja jest wykonana w Pythonie w notebooku [main.ipynb](main.ipynb). Wymagane biblioteki są w [requirements.txt](requirements.txt):

- `numpy` – operacje tablicowe i obliczenia SSE,
- `Pillow (PIL)` – renderowanie kształtów i kompozycja alfa,
- `matplotlib` – podgląd wyników i animacja,
- `pandas` – obecne w wymaganiach (nie jest kluczowe dla rdzenia algorytmu).

### 4.2. Struktury danych

- **Gen**: lista `[ (x,y), d, angle, [r,g,b,a] ]`.
- **Osobnik**: lista genów.
- **Populacja**: lista osobników.

Zwrócono uwagę na problem współdzielenia obiektów w Pythonie: w ES używana jest funkcja kopiująca osobnika „głębiej” (kopiuje listy i krotki), aby mutacje dzieci nie zmieniały rodziców.

### 4.3. Renderowanie i kompozycja

Wielokąt jest rysowany na osobnej warstwie RGBA, a następnie łączony z obrazem bazowym przez `Image.alpha_composite`. To upraszcza obsługę przezroczystości i pozwala składać obraz warstwowo.

### 4.4. Wizualizacja postępu i zapisy wyników

Zaimplementowano logger postępu, który:

- może wyświetlać postęp „na żywo” w notebooku (nadpisywanie wyjścia),
- przechowuje klatki i generuje animację (JSHTML / GIF),
- pozwala ograniczać liczbę klatek (`max_frames`) i zmniejszać rozdzielczość podglądu (`downscale_to`).

Wyniki (obrazy `.png` oraz animacje `.gif`) są zapisywane do katalogu `outputs/`.

## 5. Uzyskane wyniki

### 5.1. Artefakty wynikowe

W katalogu `outputs/` znajdują się przykładowe wyniki dla kilku obrazów testowych (m.in. Mona Lisa, flowers, dog, car) w dwóch wariantach:

- wyniki ES: np. `outputs/mona_lisa_es_best_1.png`, `outputs/mona_lisa_es_progress_1.gif`,
- wyniki IHC: np. `outputs/mona_lisa_ihc_best.png`, `outputs/mona_lisa_ihc_progress.gif`.

Wyniki mają charakter wizualny: obserwuje się stopniowe „wyłanianie się” kształtów i kolorów z tła wraz z kolejnymi iteracjami/generacjami.

### 5.2. Przykładowe ustawienia eksperymentów (z notebooka)

#### ES (przykład: Mona Lisa)

- `n = 20` (rozmiar populacji)
- `m = 5` (elita)
- `t = 100` (wielokąty na osobnika)
- `generations = 20000`
- `p = 3` (trójkąty)

Dla innych obrazów stosowano również krótsze przebiegi (np. `generations = 2000`).

#### IHC (przykład: Mona Lisa)

- `num_polygons = 400` lub `2000` (ile wielokątów dokładamy)
- `p = 5` (pięciokąty)
- `gene_trials = 1000` do `10000` (prób lokalnej optymalizacji genu)
- `vis_every` rzędu `5–20`

### 5.3. Obserwacje jakościowe

- ES dobrze eksploruje przestrzeń rozwiązań dzięki populacji, ale każda ocena jest kosztowna, bo wymaga pełnego renderowania i SSE.
- IHC często szybciej daje „sensowny zarys” obrazu, ponieważ każdy nowy kształt jest dopasowywany do aktualnego błędu, a ocena w lokalnym przeszukiwaniu jest przyspieszana przez bbox.
- Kanał alfa (półprzezroczystość) jest kluczowy: umożliwia miękkie przejścia i nakładanie się kształtów.

## 6. Wnioski końcowe i perspektywy rozwoju

### 6.1. Wnioski

1. Problem aproksymacji obrazu prymitywami geometrycznymi da się naturalnie sformułować jako problem minimalizacji SSE.
2. Reprezentacja „lista wielokątów RGBA” jest prosta, ale skuteczna i dobrze współgra z mutacjami.
3. Strategia ewolucyjna z elityzmem zapewnia stabilne ulepszanie najlepszych rozwiązań, jednak jest kosztowna obliczeniowo.
4. Konstruktywny hill climbing z lokalną optymalizacją genu jest praktycznym kompromisem: buduje obraz stopniowo i wykorzystuje heurystyki przyspieszające ocenę.

### 6.2. Perspektywy rozwoju

Możliwe, sensowne kierunki rozbudowy projektu:

- Zastosowanie innych funkcji celu (np. metryk percepcyjnych), bo SSE nie zawsze dobrze oddaje subiektywną jakość.
- Adaptacyjne kroki mutacji (zmniejszanie „kroku” wraz z postępem) i/lub osobne kroki dla różnych parametrów.
- Mieszanie typów prymitywów (różne `p`, koła/linie) zamiast jednego rodzaju wielokąta.
- Równoleglenie oceny osobników w ES (wiele renderowań niezależnych).

## 7. Jak uruchomić (minimalnie)

1. Zainstalować zależności: `pip install -r requirements.txt`.
2. Otworzyć [main.ipynb](main.ipynb) i uruchamiać komórki.
3. Wyniki zapisują się do katalogu `outputs/`.
