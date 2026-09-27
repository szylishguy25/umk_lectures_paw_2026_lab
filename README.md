# UnifiedHeatSimulation

Jeden projekt .NET łączący wszystkie warianty symulacji dyfuzji ciepła:
- **2D jednowęzłowa** i **2D wielowęzłowa (rozproszona)**
- **3D jednowęzłowa**
- **3D wielowęzłowa – podział liniowy (1D)**
- **3D wielowęzłowa – podział sześcienny (3D)**

Projekt działa zarówno na Windowsie, jak i na Linuxie. Wynikiem symulacji są statystyki w konsoli oraz plik `.txt` z rozkładem temperatur.

---

## Wymagania

| Platforma | Wymagania |
|-----------|-----------|
| **Windows** | [.NET 9 SDK](https://dotnet.microsoft.com/download) lub JetBrains Rider |
| **Ubuntu / Linux** | `sudo apt-get install -y dotnet-sdk-9.0` |

Sprawdź instalację:
```bash
dotnet --version
# oczekiwane: 9.x.x
```

---

## Konfiguracja nodes.txt

Plik `nodes.txt` definiuje adresy i porty węzłów dla **trybów wielowęzłowych (2, 4 i 5)**.  
Format: `host:port`, jeden węzeł na linię. **Pierwszy** wpis = Master (indeks 0).

```text
192.168.1.117:8091
192.168.1.118:8092
192.168.1.121:8093
192.168.1.120:8094
```

---

## Uruchomienie – JetBrains Rider (Windows)

1. Otwórz JetBrains Rider -> **File -> Open** → wskaż plik `UnifiedHeatSimulation.csproj`
2. Poczekaj na załadowanie projektu
3. Kliknij zielony przycisk **> Run** (lub `Shift+F10`)
4. W konsoli pojawi się interaktywne menu – wpisz numer trybu i Enter

Aby uruchomić konkretny tryb **bez menu** (przez argumenty):  
Rider → **Run/Debug Configurations → Program arguments**: wpisz np. `3d-single`

---

## Uruchomienie – terminal (Windows i Ubuntu)

Przejdź do folderu projektu:
```bash
cd C:\Users\szymo\RiderProjects\UnifiedHeatSimulation   # Windows
cd ~/RiderProjects/UnifiedHeatSimulation                # Ubuntu
```

### Interaktywne menu (wpisz numer trybu po uruchomieniu)
```bash
dotnet run
```

### Bezpośredni wybór trybu przez argument
```bash
dotnet run -- 1          # 2D jednowęzłowa (lokalnie)
dotnet run -- 2          # 2D wielowęzłowa (klaster TCP)
dotnet run -- 3          # 3D jednowęzłowa (lokalnie)
dotnet run -- 4          # 3D wielowęzłowa - komunikacja 1D (klaster TCP)
dotnet run -- 5          # 3D wielowęzłowa - komunikacja 3D (klaster TCP)
```

Można też użyć nazw zamiast cyfr:
```bash
dotnet run -- 2d-single
dotnet run -- 2d-comm
dotnet run -- 3d-single
dotnet run -- 3d-comm1d
dotnet run -- 3d-comm3d
```

---

## Tryby 1 i 3 (lokalne) – jak działają

Po wybraniu trybu program **interaktywnie pyta** o wszystkie parametry symulacji (z wartościami domyślnymi – wystarczy naciskać Enter) - domyślne dane są w []:

```
Szerokość (X) [20]: 
Wysokość  (Y) [20]: 
Głębokość (Z) [20]: 
alpha (współczynnik dyfuzji) [0,1]: 
dt (krok czasowy, 0 = auto) [0]: 
Liczba kroków [100]: 
Temperatura otoczenia [°C] [20]: 
Liczba źródeł ciepła [1]: 
-- Źródło #1 --
  X [10]: 
  Y [10]: 
  Temperatura [°C] [100]: 
```

Po zakończeniu symulacji w katalogu roboczym pojawi się plik wynikowy:

| Tryb | Plik wynikowy |
|------|--------------|
| 2D jednowęzłowa | `result_2d_single.txt` |
| 3D jednowęzłowa | `result_3d_single.txt` (środkowy slice Z) |

---

## Tryby 2, 4 i 5 (rozproszone) – jak uruchomić klaster

### Krok 1 – upewnij się, że `nodes.txt` jest poprawny

```text
192.168.1.117:8091   ← Master (indeks 0)
192.168.1.118:8092   ← Worker 1
192.168.1.121:8093   ← Worker 2
192.168.1.120:8094   ← Worker 3
```

### Krok 2 – skopiuj skompilowany program na każdy węzeł

Opublikuj jako self-contained (nie wymaga .NET na docelowej maszynie):
```bash
dotnet publish -c Release -r linux-x64 --self-contained true
# plik wykonywalny: bin/Release/net9.0/linux-x64/publish/UnifiedHeatSimulation
```

Lub skompiluj do uruchamiania przez `dotnet run` (wymaga .NET SDK na każdej maszynie):
```bash
dotnet build -c Release
```

### Krok 3 – uruchom węzły (każdy na swojej maszynie)

> **Ważne:** Uruchamiaj węzły **w tej samej kolejności** – najpierw workery, potem Master. Master czeka aż wszyscy się połączą.

Na **każdym węźle** uruchamiasz tę samą binarę, ale podajesz **swój własny port**:

```bash
# Węzeł 1 (192.168.1.118) – Worker 1, port 8092
dotnet run -- 4
# Program zapyta: "Mój port [8091]:" → wpisz 8092

# Węzeł 2 (192.168.1.121) – Worker 2, port 8093
dotnet run -- 4
# → wpisz 8093

# Węzeł 3 (192.168.1.120) – Worker 3, port 8094
dotnet run -- 4
# → wpisz 8094

# Węzeł 0 (192.168.1.117) – MASTER, port 8091 – uruchom ostatni!
dotnet run -- 4
# → wpisz 8091
# Następnie Master zapyta o parametry symulacji i rozesle je do workerów
```

### Różnica między trybami w 3D (tryb 4 a 5)

| | Tryb 4 – `3d-comm1d` | Tryb 5 – `3d-comm3d` |
|---|---|---|
| Podział siatki | Tylko po osi Y (plastry) | Po 3 osiach (kostki) |
| Sąsiedzi każdego węzła | Max 2 (góra/dół) | Max 6 (lewo/prawo, góra/dół, przód/tył) |
| Halo wymieniane | 1 płaszczyzna (XZ) | Do 3 płaszczyzn (YZ, XZ, XY) |
| Efektywność | Dobra dla małej liczby węzłów | Lepsza skalowalność dla wielu węzłów |

---

## Struktura plików i za co odpowiadają

```
UnifiedHeatSimulation/
 ├── UnifiedHeatSimulation.csproj
 ├── nodes.txt                          ← Konfiguracja węzłów sieciowych (IP:PORT)
 ├── Program.cs                         ← Punkt wejścia (Main) oraz interaktywne menu główne
 │
 ├── Core/
 │    ├── SimulationConfig.cs           ← Wspólna konfiguracja (rozmiary, dt, kroki) + wczytywanie nodes.txt
 │    ├── ConsoleHelper.cs              ← Helpery I/O konsoli, odczytywanie parametrów od użytkownika
 │    └── SimulationLogger.cs           ← Ujednolicone logowanie czasu (UTC) i statystyk (min/avg/max) do konsoli i plików CSV
 │
 └── Simulations/
      ├── Heat2D_Single.cs              ← Dyfuzja 2D: obliczenia jednowęzłowe na prostej tablicy (tryb 1)
      ├── Heat3D_Single.cs              ← Dyfuzja 3D: obliczenia jednowęzłowe w pętli 3D (tryb 3)
      │
      └── Distributed/                  ← Katalog trybów wielowęzłowych (rozproszonych, klaster przez TCP)
           ├── Heat2D_Comm.cs           ← Główna logika dla trybu 2: podział poziomy w 2D
           ├── Heat3D_Comm1D.cs         ← Główna logika dla trybu 4: podział liniowy w osi Y
           ├── Heat3D_Comm3D.cs         ← Główna logika dla trybu 5: podział sześcienny w osiach XYZ
           │
           └── Components/              ← Elementy współdzielone dla obliczeń po sieci
                ├── NetworkComponents.cs← NetworkManager (TCP/IP), MessageKind, NetMessage, SliceSerializer
                ├── HeatFragment2D.cs   ← Model i matematyka dyfuzji dla siatki 2D pociętej w poziome plastry
                ├── HeatFragment1D.cs   ← Model i matematyka dyfuzji dla siatki 3D pociętej w plastry
                └── HeatFragment3D.cs   ← Model i matematyka dyfuzji dla siatki 3D pociętej w kostki
```

**Krótko o rozbiciu dyfuzji rozproszonej:**
- `Heat2D_Comm.cs`, `Heat3D_Comm1D.cs` / `Comm3D.cs` – uruchamiają Mastera (czyta konfigurację, dzieli pracę, wysyła pakiety Config/Start, czeka na wyniki) oraz Workerów (którzy nasłuchują, otrzymują swój fragment i liczą pętlę czasu).
- `Components/NetworkComponents.cs` – Przesyłają pakiety przez sieć używając `TcpListener` / `TcpClient`, z mechanizmem retry.
- `Components/HeatFragment1D.cs` (oraz 3D) – Czysta matematyka FDM na przypisanym bloku trójwymiarowej tablicy. Posiadają funkcje obliczania środka, jak i krawędzi wymagających sąsiedniego "halo" (danych od sąsiadów z klastra).

---

## Rozwiązywanie problemów

**`dotnet: command not found` (Ubuntu)**
```bash
sudo apt-get update && sudo apt-get install -y dotnet-sdk-9.0
```

**Węzeł nie może połączyć się z innym**  
Sprawdź, czy port jest otwarty w zaporze (firewall):
```bash
# Ubuntu
sudo ufw allow 8091/tcp
sudo ufw allow 8092/tcp
sudo ufw allow 8093/tcp
sudo ufw allow 8094/tcp
```

**Master nie widzi workerów**  
- Upewnij się, że workery są uruchomione **przed** Masterem
- Sprawdź, czy adresy IP w `nodes.txt` są dostępne z każdego węzła (`ping 192.168.1.117`)
- Program automatycznie ponawia próby połączenia przez 15 sekund
