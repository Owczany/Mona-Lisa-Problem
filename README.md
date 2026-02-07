# Mona Lisa Project

## Description

Celem projektu jest aproksymacja obrazu (np. „Mona Lisa”) przy pomocy prostych prymitywów geometrycznych oraz algorytmów inspirowanych ewolucją.
Każde rozwiązanie (osobnik) koduje obraz jako listę wielokątów foremnych z kanałem alfa (RGBA). Jakość rozwiązania oceniamy porównując wygenerowany obraz z obrazem docelowym.


# Core Idea

- Reprezentacja: obraz = suma warstw (wielokąty foremne) rysowanych na płótnie RGBA.
- Gen (wielokąt) jest opisany przez: środek `(x, y)`, rozmiar/promień `d`, obrót `angle`, kolor `[r, g, b, a]` oraz liczbę wierzchołków `p`.
- Funkcja celu (fitness): suma kwadratów różnic pikseli (SSE) pomiędzy obrazem docelowym i wygenerowanym w RGB (mniej = lepiej).


## Problems

- Duża przestrzeń rozwiązań: nawet kilkaset–kilka tysięcy wielokątów daje bardzo wysokowymiarową optymalizację.
- Koszt obliczeń: renderowanie i liczenie fitnessu jest drogie, więc warto ograniczać liczbę ocen albo liczyć przyrostowo.
- Lokalne minima: proste mutacje łatwo „utykają”, dlatego potrzebne są strategie iteracyjne i/lub selekcja najlepszych.


# Results

- Projekt generuje przybliżenia obrazu docelowego (podgląd na żywo w notebooku) oraz może zapisywać wyniki do plików w folderze `outputs/`.

# Algorithms

Poniżej krótki opis algorytmów użytych w implementacji (szczegóły w notebooku `main.ipynb`).

## Evolution Strategy (ES) z elityzmem

- Utrzymujemy populację rozmiaru `n`, gdzie osobnik zawiera `t` wielokątów.
- W każdej generacji wybieramy `m` najlepszych (elita/rodzice).
- Pozostałe `n-m` miejsc zastępujemy dziećmi: kopiujemy rodzica i wykonujemy mutację pojedynczego genu (pozycja / rozmiar / kąt / kanał koloru).
- Ocena: SSE względem obrazu docelowego.

## Incremental / Constructive Hill Climbing (IHC)

- Start od pustego (jednolitego) tła.
- Iteracyjnie dokładamy po jednym wielokącie: losujemy gen startowy, lokalnie go optymalizujemy losowymi mutacjami i akceptujemy najlepszy znaleziony wariant.
- Dla przyspieszenia, podczas lokalnej optymalizacji pojedynczego genu używamy przyrostowej oceny na fragmencie obrazu (bbox wielokąta), zamiast przeliczać cały SSE od zera.

## Contributors

| Name | Surrname | Nickname | Studnet Number |
| --- | --- | --- | --- |
| Piotr | Pijanowski | Owca | 346952 |
| Mateusz | Fąferko | Makler | 345736 |