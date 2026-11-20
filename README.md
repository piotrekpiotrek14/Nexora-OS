<p align="center">
  <img src="docs/screenshots/banner.png" alt="NEXORA OS" width="100%">
</p>

<p align="center">
  <b>A privacy-focused desktop operating system built on Debian 13 "trixie" and KDE Plasma 6.</b><br>
  System services in Rust · AI assistant in Python · installable ISO
</p>

<p align="center">
  <a href="README.pl.md">🇵🇱 Wersja polska</a> ·
  <a href="docs/roadmap.md">Roadmap</a> ·
  <a href="docs/architecture.md">Architecture</a> ·
  <a href="docs/threat-model.md">Threat model</a>
</p>

> **Status: in development — version 0.3.0, phase 4 of 8.**
> The ISO builds and boots, the NEXORA look is complete from the boot menu to
> the login screen, and the first NEXORA service (the download manager) passes
> its tests. Some features below are still planned; this page marks clearly
> which ones work today.

---

## What is NEXORA OS?

NEXORA OS is a Linux distribution for everyday computers that takes privacy
seriously without being hard to use. It is built on **Debian Stable**, one of
the most widely used and stable Linux foundations, and adds:

- a consistent **dark design** from the first boot screen to the desktop,
- **privacy-friendly defaults** (no telemetry, random Wi-Fi addresses, a
  hardened Firefox),
- **its own tools**: a download manager, a **Ctrl+E AI assistant** that can run
  entirely on your computer, a Windows migration tool and network privacy tools.

### Principles

| Principle | What it means in practice |
|---|---|
| **No telemetry** | Nothing is collected or sent anywhere without your explicit consent. |
| **Honest privacy claims** | Tor Browser hides the traffic of Tor Browser only, not of the whole system, and NEXORA says so instead of promising "anonymity". |
| **Least privilege** | Desktop apps never run as root. Services run with their own minimal rights, AppArmor profiles and systemd sandboxing. |
| **Safe by design** | The AI assistant never runs a destructive command without your explicit confirmation. The migration tool only reads from Windows: it never modifies or deletes the Windows installation. |
| **Debian underneath** | Software comes from Debian packages, so security updates arrive automatically. |

---

## Screenshots

### Boot menu (GRUB)

<p align="center"><img src="docs/screenshots/grub-menu.png" alt="NEXORA GRUB boot menu" width="80%"></p>

The NEXORA boot menu theme, rendered by GRUB itself in an automated visual test.

### Login screen (SDDM)

<p align="center"><img src="docs/screenshots/sddm-login.png" alt="NEXORA login screen" width="90%"></p>

The NEXORA login theme, rendered by SDDM's own greeter in an automated test
(the user "ana" is a test account).

### Wallpapers

| Orbit | Horizon | Aurora |
|---|---|---|
| ![Orbit](docs/screenshots/wallpaper-orbit.png) | ![Horizon](docs/screenshots/wallpaper-horizon.png) | ![Aurora](docs/screenshots/wallpaper-aurora.png) |

All artwork is original and drawn as SVG, so it stays sharp on every screen.

### Firefox privacy defaults

<p align="center"><img src="docs/screenshots/firefox-privacy-policies.png" alt="Firefox about:policies in NEXORA" width="70%"></p>

Firefox's own `about:policies` page in NEXORA: telemetry, studies and
sponsored content off, HTTPS-only mode on, tracking and fingerprinting
protection on.

<!-- BOOT-SCREENSHOTS -->

---

## Features and status

✅ done and tested · 🔧 in progress · 📝 designed · ⏳ planned

| Area | Feature | Status |
|---|---|---|
| Base | Debian 13 "trixie", Linux 6.12 LTS, KDE Plasma 6.3, Qt 6.8 | ✅ |
| Base | Bootable ISO (BIOS and UEFI) with a live session, automated boot test | ✅ |
| Base | Installer (Calamares) | 🔧 included with Debian's settings; NEXORA setup with LUKS2 disk encryption in phase 6 |
| Base | Secure Boot (Debian's signed boot chain) | 🔧 enabled in the build; test on a machine with Secure Boot on is pending |
| Look | Logo, 3 wallpapers, boot menu, boot splash (Plymouth), login screen, Plasma layout (top bar + dock), Nexora Dark colours | ✅ colour contrast checked against WCAG |
| Apps | Dolphin (file manager), Firefox ESR, VS Code, Konsole, Kate, Okular, Gwenview, Ark, Discover + Flatpak, KeePassXC, Spectacle, Partition Manager, Timeshift and Vorta (backups), TestDisk | ✅ included |
| Hardware | Wi-Fi (NetworkManager), Bluetooth, PipeWire audio, printing (CUPS), USB drives (NTFS, exFAT) | ✅ included · ⏳ test on real hardware |
| Privacy | Firefox and VS Code telemetry off | ✅ tested |
| Privacy | Random Wi-Fi MAC while scanning, separate MAC per network, IPv6 privacy addresses, computer name not sent to the network | ✅ tested |
| Security | Firewall (firewalld), AppArmor, automatic security updates | ✅ included · ⏳ NEXORA settings in phase 6 |
| Tor | Tor Browser (only Tor Browser's own traffic goes through Tor) | ✅ included · ⏳ status widget |
| NEXORA | **Download manager** service (Rust): pause, resume, cancel, history, SHA-256 check, safe file names | ✅ service and tests · ⏳ window |
| NEXORA | **Network privacy**: nftables kill switch (VPN only / Tor only), identity change ("IP changer"), proxy chains | 📝 [design](docs/network-privacy.md) |
| NEXORA | **Local AI with Ollama**: models run on your computer, in a sandbox, with cloud features off | 📝 [design](docs/local-ai-ollama.md) |
| NEXORA | **Ctrl+E AI assistant**: Ollama on your computer, or Claude by Anthropic with your own key (off by default) | ⏳ phase 5 |
| NEXORA | **Windows migration**: copies your files from Windows, read-only | ⏳ phase 4 |
| NEXORA | **Privacy Dashboard** and recovery tools | ⏳ phase 6 |

---

## Technology

| Part | Technology |
|---|---|
| Base system | Debian 13 "trixie" (Stable), Linux 6.12 LTS, systemd |
| Desktop | KDE Plasma 6.3, Qt 6.8 / QML |
| System services | **Rust** 1.85: tokio, zbus (D-Bus), reqwest with rustls, SQLite |
| AI and automation | **Python** 3.13 |
| ISO and installer | live-build, Calamares |
| Packaging | Debian packages (debhelper), checked by lintian |
| Build and tests | Docker builder container, QEMU/OVMF boot tests, `cargo test`, `pytest` |

## How it is tested

- Everything builds in a clean Debian container with one command, on Windows
  or Linux.
- **Static checks**: shellcheck for all scripts, XML/JSON/TOML validation,
  Debian package checks (lintian).
- **Unit tests**: Rust services, the Python assistant settings and the design
  system (including colour contrast).
- **Package tests**: every NEXORA package is installed, upgraded and removed
  in a container, and no NEXORA file may collide with a file of any Debian
  package.
- **Download manager tests** against a local test web server: resuming after
  a dropped connection, a file changed on the server, a wrong checksum,
  hostile file names (path tricks), never overwriting files, private
  downloads never stored in history.
- **Visual tests** render the real boot menu, login screen and Firefox policy
  page (the screenshots above).
- **Boot test**: the ISO boots in a virtual machine to a login prompt and must
  identify itself as NEXORA OS.

---

## Build it yourself

Everything builds inside a Debian container; the computer only needs Docker.

**Windows:** Docker Desktop with WSL 2. From the project folder:

```powershell
.\build\nexora.ps1 check    # lint, tests, .deb packages and package tests
.\build\nexora.ps1 iso      # build the ISO (30–90 minutes)
.\build\nexora.ps1 smoke    # boot the ISO in a virtual machine
```

**Linux:** the same commands with `build/nexora.sh` (Docker or Podman).

`-VSCode none` builds an ISO without Microsoft VS Code (see the licence note
below). The ISO appears in `out/iso/`.

## Project structure

```
artwork/      logo, wallpapers, boot and login themes (SVG sources)
build/        build scripts: builder container, ISO, boot tests
crates/       Rust services (nexora-common, nexora-downloads)
debian/       Debian packaging
design/       design tokens (colours, fonts, spacing)
distro/       live-build configuration of the ISO
docs/         architecture, threat model, roadmap, designs
interfaces/   D-Bus interfaces between the services and the desktop
plasma/       KDE Plasma defaults, layouts, Firefox policies
python/       AI assistant (Python)
security/     AppArmor profiles, network privacy settings
tests/        package, design and visual tests
tools/        build helpers
```

## Roadmap

| Phase | Content | Status |
|---|---|---|
| 1 | Design: architecture, threat model, technology choices, design system | ✅ |
| 2 | Build system, Debian packages, bootable ISO | ✅ |
| 3 | Desktop: branding, themes, layout, privacy defaults | ✅ |
| 4 | Download manager, network privacy (kill switch, IP changer, proxy chains, nftables), local AI (Ollama), Windows migration, Tor widget | 🔧 |
| 5 | Ctrl+E AI assistant with command safety checks | ⏳ |
| 6 | Hardening, installer with disk encryption, updates, Privacy Dashboard, recovery | ⏳ |
| 7 | Tests and user documentation | ⏳ |
| 8 | Release validation | ⏳ |

Details and acceptance criteria: [docs/roadmap.md](docs/roadmap.md);
day-to-day progress: [PROGRESS.md](PROGRESS.md).

## Documentation

| Document | Contents |
|---|---|
| [architecture.md](docs/architecture.md) | Layers, components, privileges, D-Bus contracts, build pipeline |
| [threat-model.md](docs/threat-model.md) | What NEXORA protects against, what it does not, and Tor's limits |
| [technology-choices.md](docs/technology-choices.md) | Decisions, rejected alternatives, verified versions, licences |
| [design-system.md](docs/design-system.md) | Colours, typography, layout, accessibility |
| [network-privacy.md](docs/network-privacy.md) | Kill switch, identity change, proxy chains, nftables |
| [local-ai-ollama.md](docs/local-ai-ollama.md) | Local AI with Ollama |

## About the project

NEXORA OS is a personal project by **[your name]**. The author set its goals,
features and rules (privacy, security, honesty about limits) and leads each
phase. It is developed with the help of Claude Code, Anthropic's AI coding
assistant; every part is checked by the automated builds and tests described
above.

## Licence

NEXORA's code and artwork: GPL-3.0-or-later. Third-party software keeps its
own licence. An ISO built with the default options contains Microsoft Visual
Studio Code, which has a proprietary licence, so ISO images are not published
here; `-VSCode none` builds a fully free ISO.

Debian, KDE, Firefox, Tor, Ollama and Claude are trademarks of their owners;
NEXORA OS is an independent project and is not affiliated with them.
