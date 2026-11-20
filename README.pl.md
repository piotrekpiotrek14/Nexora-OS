<p align="center">
  <img src="docs/screenshots/banner.png" alt="NEXORA OS" width="100%">
</p>

<p align="center">
  <b>System operacyjny dla komputerów osobistych, nastawiony na prywatność, oparty na Debianie 13 „trixie” i KDE Plasma 6.</b><br>
  Usługi systemowe w Rust · asystent AI w Pythonie · instalowalny obraz ISO
</p>

<p align="center">
  <a href="README.md">🇬🇧 English version</a> ·
  <a href="docs/roadmap.md">Plan prac</a> ·
  <a href="docs/architecture.md">Architektura</a> ·
  <a href="docs/threat-model.md">Model zagrożeń</a>
</p>

> **Stan: w trakcie tworzenia — wersja 0.3.0, etap 4 z 8.**
> Obraz ISO się buduje i uruchamia, wygląd NEXORA jest gotowy od menu startowego
> po ekran logowania, a pierwsza własna usługa NEXORA (menedżer pobierania)
> przechodzi wszystkie testy. Część funkcji opisanych niżej jest dopiero
> zaplanowana — ta strona wyraźnie oznacza, co działa już dziś.

---

## Czym jest NEXORA OS?

NEXORA OS to dystrybucja Linuksa do codziennego użytku, która poważnie traktuje
prywatność, a przy tym pozostaje prosta w obsłudze. Powstaje na bazie
**Debiana Stable** — jednej z najpopularniejszych i najstabilniejszych podstaw
systemów linuksowych — i dodaje do niego:

- spójny **ciemny wygląd** od pierwszego ekranu startowego aż po pulpit,
- **ustawienia przyjazne prywatności** (brak telemetrii, losowe adresy Wi-Fi,
  zabezpieczony Firefox),
- **własne narzędzia**: menedżer pobierania, **asystenta AI pod Ctrl+E**, który
  może działać w całości na komputerze użytkownika, narzędzie do przenoszenia
  plików z Windowsa oraz narzędzia prywatności sieciowej.

### Zasady

| Zasada | Co to oznacza w praktyce |
|---|---|
| **Brak telemetrii** | Nic nie jest zbierane ani wysyłane bez wyraźnej zgody użytkownika. |
| **Uczciwe obietnice** | Tor Browser ukrywa ruch tylko samej przeglądarki Tor, a nie całego systemu — i NEXORA mówi to wprost, zamiast obiecywać „anonimowość”. |
| **Minimalne uprawnienia** | Aplikacje na pulpicie nigdy nie działają jako root. Usługi mają tylko te uprawnienia, których potrzebują, profile AppArmor i izolację systemd. |
| **Bezpieczeństwo w projekcie** | Asystent AI nigdy nie wykona niebezpiecznego polecenia bez wyraźnego potwierdzenia. Narzędzie migracji tylko czyta dane z Windowsa — nigdy nie zmienia ani nie usuwa instalacji Windows. |
| **Debian pod spodem** | Oprogramowanie pochodzi z pakietów Debiana, więc poprawki bezpieczeństwa przychodzą automatycznie. |

---

## Zrzuty ekranu

### Menu startowe (GRUB)

<p align="center"><img src="docs/screenshots/grub-menu.png" alt="Menu startowe NEXORA" width="80%"></p>

Motyw menu startowego NEXORA wyrenderowany przez sam program GRUB w
automatycznym teście wizualnym.

### Ekran logowania (SDDM)

<p align="center"><img src="docs/screenshots/sddm-login.png" alt="Ekran logowania NEXORA" width="90%"></p>

Motyw logowania NEXORA wyświetlony przez program logowania SDDM w
automatycznym teście (użytkownik „ana” to konto testowe).

### Tapety

| Orbit | Horizon | Aurora |
|---|---|---|
| ![Orbit](docs/screenshots/wallpaper-orbit.png) | ![Horizon](docs/screenshots/wallpaper-horizon.png) | ![Aurora](docs/screenshots/wallpaper-aurora.png) |

Wszystkie grafiki są oryginalne i narysowane jako SVG, więc są ostre na
każdym ekranie.

### Ustawienia prywatności Firefoksa

<p align="center"><img src="docs/screenshots/firefox-privacy-policies.png" alt="Strona about:policies Firefoksa w NEXORA" width="70%"></p>

Strona `about:policies` Firefoksa w NEXORA: telemetria, badania i treści
sponsorowane wyłączone, tryb „tylko HTTPS” włączony, ochrona przed śledzeniem
i fingerprintingiem włączona.

<!-- BOOT-SCREENSHOTS -->

---

## Funkcje i ich stan

✅ gotowe i przetestowane · 🔧 w trakcie · 📝 zaprojektowane · ⏳ zaplanowane

| Obszar | Funkcja | Stan |
|---|---|---|
| Podstawa | Debian 13 „trixie”, Linux 6.12 LTS, KDE Plasma 6.3, Qt 6.8 | ✅ |
| Podstawa | Uruchamialny obraz ISO (BIOS i UEFI) z sesją live, automatyczny test uruchamiania | ✅ |
| Podstawa | Instalator (Calamares) | 🔧 dołączony z ustawieniami Debiana; wersja NEXORA z szyfrowaniem dysku LUKS2 w etapie 6 |
| Podstawa | Secure Boot (podpisany łańcuch startowy Debiana) | 🔧 włączony w budowie; test na komputerze z włączonym Secure Boot jeszcze przed nami |
| Wygląd | Logo, 3 tapety, menu startowe, ekran ładowania (Plymouth), ekran logowania, układ Plasmy (górny pasek + dok), kolory Nexora Dark | ✅ kontrast kolorów sprawdzony według WCAG |
| Aplikacje | Dolphin (menedżer plików), Firefox ESR, VS Code, Konsole, Kate, Okular, Gwenview, Ark, Discover + Flatpak, KeePassXC, Spectacle, menedżer partycji, Timeshift i Vorta (kopie zapasowe), TestDisk | ✅ dołączone |
| Sprzęt | Wi-Fi (NetworkManager), Bluetooth, dźwięk PipeWire, drukowanie (CUPS), dyski USB (NTFS, exFAT) | ✅ dołączone · ⏳ test na prawdziwym sprzęcie |
| Prywatność | Telemetria Firefoksa i VS Code wyłączona | ✅ przetestowane |
| Prywatność | Losowy adres MAC podczas skanowania Wi-Fi, osobny MAC dla każdej sieci, prywatne adresy IPv6, nazwa komputera niewysyłana do sieci | ✅ przetestowane |
| Bezpieczeństwo | Zapora (firewalld), AppArmor, automatyczne aktualizacje bezpieczeństwa | ✅ dołączone · ⏳ ustawienia NEXORA w etapie 6 |
| Tor | Tor Browser (przez sieć Tor idzie tylko ruch samej przeglądarki Tor) | ✅ dołączony · ⏳ widżet stanu |
| NEXORA | **Menedżer pobierania** (usługa w Rust): pauza, wznawianie, anulowanie, historia, sprawdzanie sumy SHA-256, bezpieczne nazwy plików | ✅ usługa i testy · ⏳ okno |
| NEXORA | **Prywatność sieciowa**: kill switch na nftables (tylko VPN / tylko Tor), zmiana tożsamości („zmieniacz IP”), łańcuchy proxy | 📝 [projekt](docs/network-privacy.md) |
| NEXORA | **Lokalna AI z Ollamą**: modele działają na komputerze użytkownika, w izolacji, z wyłączonymi funkcjami chmurowymi | 📝 [projekt](docs/local-ai-ollama.md) |
| NEXORA | **Asystent AI pod Ctrl+E**: Ollama na komputerze albo Claude od Anthropic z własnym kluczem użytkownika (domyślnie wyłączony) | ⏳ etap 5 |
| NEXORA | **Migracja z Windowsa**: kopiuje pliki z Windowsa, tylko do odczytu | ⏳ etap 4 |
| NEXORA | **Panel prywatności** i narzędzia odzyskiwania systemu | ⏳ etap 6 |

---

## Technologie

| Część | Technologia |
|---|---|
| System bazowy | Debian 13 „trixie” (Stable), Linux 6.12 LTS, systemd |
| Pulpit | KDE Plasma 6.3, Qt 6.8 / QML |
| Usługi systemowe | **Rust** 1.85: tokio, zbus (D-Bus), reqwest z rustls, SQLite |
| AI i automatyzacja | **Python** 3.13 |
| Obraz ISO i instalator | live-build, Calamares |
| Pakiety | pakiety Debiana (debhelper), sprawdzane przez lintian |
| Budowanie i testy | kontener Docker, testy uruchamiania w QEMU/OVMF, `cargo test`, `pytest` |

## Jak projekt jest testowany

- Wszystko buduje się jednym poleceniem w czystym kontenerze Debiana — na
  Windowsie albo na Linuksie.
- **Kontrole statyczne**: shellcheck dla wszystkich skryptów, sprawdzanie
  plików XML/JSON/TOML, kontrola pakietów Debiana (lintian).
- **Testy jednostkowe**: usługi w Rust, ustawienia asystenta w Pythonie i
  system projektowy (w tym kontrast kolorów).
- **Testy pakietów**: każdy pakiet NEXORA jest instalowany, aktualizowany i
  usuwany w kontenerze, a żaden plik NEXORA nie może kolidować z plikiem
  jakiegokolwiek pakietu Debiana.
- **Testy menedżera pobierania** z lokalnym serwerem testowym: wznawianie po
  zerwanym połączeniu, plik zmieniony na serwerze, błędna suma kontrolna,
  podejrzane nazwy plików (sztuczki ze ścieżkami), żadnego nadpisywania plików,
  prywatne pobrania nigdy niezapisywane w historii.
- **Testy wizualne** renderują prawdziwe menu startowe, ekran logowania i
  stronę ustawień Firefoksa (zrzuty ekranu powyżej).
- **Test uruchamiania**: obraz ISO uruchamia się w maszynie wirtualnej aż do
  ekranu logowania i musi przedstawić się jako NEXORA OS.

---

## Jak zbudować samemu

Wszystko buduje się w kontenerze Debiana — na komputerze potrzebny jest tylko
Docker.

**Windows:** Docker Desktop z WSL 2. W folderze projektu:

```powershell
.\build\nexora.ps1 check    # kontrole, testy, pakiety .deb i ich testy
.\build\nexora.ps1 iso      # budowa obrazu ISO (30–90 minut)
.\build\nexora.ps1 smoke    # uruchomienie ISO w maszynie wirtualnej
```

**Linux:** te same polecenia ze skryptem `build/nexora.sh` (Docker lub Podman).

`-VSCode none` buduje obraz bez Microsoft VS Code (zob. uwagę o licencji
poniżej). Gotowy obraz ISO pojawia się w `out/iso/`.

## Struktura projektu

```
artwork/      logo, tapety, motywy ekranu startowego i logowania (źródła SVG)
build/        skrypty budowania: kontener, obraz ISO, testy uruchamiania
crates/       usługi w Rust (nexora-common, nexora-downloads)
debian/       pakiety Debiana
design/       tokeny projektowe (kolory, czcionki, odstępy)
distro/       konfiguracja live-build obrazu ISO
docs/         architektura, model zagrożeń, plan prac, projekty
interfaces/   interfejsy D-Bus między usługami a pulpitem
plasma/       ustawienia KDE Plasma, układy, zasady Firefoksa
python/       asystent AI (Python)
security/     profile AppArmor, ustawienia prywatności sieci
tests/        testy pakietów, projektu graficznego i testy wizualne
tools/        narzędzia pomocnicze do budowania
```

## Plan prac

| Etap | Zakres | Stan |
|---|---|---|
| 1 | Projekt: architektura, model zagrożeń, wybór technologii, system projektowy | ✅ |
| 2 | System budowania, pakiety Debiana, uruchamialny obraz ISO | ✅ |
| 3 | Pulpit: identyfikacja wizualna, motywy, układ, ustawienia prywatności | ✅ |
| 4 | Menedżer pobierania, prywatność sieciowa (kill switch, zmieniacz IP, łańcuchy proxy, nftables), lokalna AI (Ollama), migracja z Windowsa, widżet Tor | 🔧 |
| 5 | Asystent AI pod Ctrl+E ze sprawdzaniem bezpieczeństwa poleceń | ⏳ |
| 6 | Wzmocnienie zabezpieczeń, instalator z szyfrowaniem dysku, aktualizacje, panel prywatności, odzyskiwanie systemu | ⏳ |
| 7 | Testy i dokumentacja dla użytkownika | ⏳ |
| 8 | Sprawdzenie wersji do wydania | ⏳ |

Szczegóły i kryteria odbioru: [docs/roadmap.md](docs/roadmap.md) (po
angielsku); bieżący postęp: [PROGRESS.md](PROGRESS.md).

## O projekcie

NEXORA OS to osobisty projekt, którego autorem jest **[twoje imię i nazwisko]**.
Autor określił cele, funkcje i zasady projektu (prywatność, bezpieczeństwo,
uczciwość co do ograniczeń) i prowadzi każdy etap prac. Projekt powstaje z
pomocą Claude Code — asystenta programowania AI firmy Anthropic; każda część
jest sprawdzana przez opisane wyżej automatyczne budowanie i testy.

## Licencja

Kod i grafika NEXORA: GPL-3.0-or-later. Oprogramowanie innych autorów
zachowuje własne licencje. Obraz ISO zbudowany z domyślnymi opcjami zawiera
Microsoft Visual Studio Code, który ma licencję zastrzeżoną, dlatego obrazy ISO
nie są tu publikowane; opcja `-VSCode none` buduje obraz w całości wolny.

Debian, KDE, Firefox, Tor, Ollama i Claude są znakami towarowymi swoich
właścicieli; NEXORA OS jest niezależnym projektem i nie jest z nimi powiązany.
