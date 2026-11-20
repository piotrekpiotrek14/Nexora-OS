# NEXORA OS — Technology Choices, Licensing and Hardware Support

Status: Phase 1 deliverable. Each decision is written as a short ADR: the
choice, what was rejected, and why. Version numbers marked **(verify)** are
expected values for Debian 13 "trixie" and are confirmed with
`apt-cache policy` in the Phase 2 builder container before anything depends on them.

---

## 1. Platform decisions

| # | Decision | Choice | Rejected | Rationale |
|---|---|---|---|---|
| D1 | Base distribution | **Debian 13 "trixie"** (current Debian Stable), `main contrib non-free-firmware` | Debian testing; Ubuntu LTS | Stable security support (~3 years + LTS), signed Secure Boot chain, live-build and Calamares packaged. `contrib` only for `torbrowser-launcher`. |
| D2 | Architecture | amd64 only for 1.0 | arm64 | Halves the test matrix; arm64 laptops need per-device work. |
| D3 | Desktop | **KDE Plasma 6, Wayland session default**, X11 session available | GNOME; custom shell | Requested; Qt/QML theming depth; Wayland gives input/screen isolation between apps. |
| D4 | Image build | **live-build** producing `iso-hybrid` (UEFI + BIOS) | mkosi; debian-cd | Debian's own live images use it; Calamares integration is proven. |
| D5 | Installer | **Calamares** with NEXORA settings forked from `calamares-settings-debian` | debian-installer | Graphical, supports LUKS2, btrfs subvolumes, branding and slideshow in QML. |
| D6 | Kernel & boot | Debian-signed `linux-image-amd64` (6.12 LTS), `shim-signed`, `grub-efi-amd64-signed` | Custom kernel; systemd-boot | Secure Boot works without NEXORA owning a Microsoft-signed shim. |
| D7 | Root filesystem | **btrfs** in LUKS2 with `@`/`@home` subvolumes; ext4 offered | ext4 default | Enables Timeshift snapshots = practical "system restore". **Open decision** for the user (see roadmap). |
| D8 | Swap | zram (`systemd-zram-generator`, verify package name) | Disk swap partition | No plaintext swap on disk; hibernation off by default (documented trade-off). |
| D9 | Firewall | **firewalld** (nftables backend) + `plasma-firewall` UI | ufw | Per-connection zones via NetworkManager ("public" vs "home"); D-Bus API with polkit for the dashboard. |
| D10 | MAC/confinement | **AppArmor** (Debian default) + extra profiles; Flatpak/bubblewrap for untrusted apps | SELinux; Firejail | AppArmor is already enabled in Debian; Firejail is setuid-root and adds attack surface. |
| D11 | Audio | PipeWire + WirePlumber + plasma-pa | PulseAudio | Debian 13 default for Plasma; portals for mic/camera consent. |
| D12 | Network | NetworkManager + plasma-nm; WireGuard native; `network-manager-openvpn` | systemd-networkd | Wi-Fi, VPN, captive portals, per-connection privacy settings. |
| D13 | DNS | systemd-resolved, optional strict DNS-over-TLS | DoH in browser only; dnscrypt-proxy | One system-wide place to configure; honest UI about what it protects. |
| D14 | File manager | **Dolphin** + KIO + Ark + kio-extras + ffmpegthumbs | Writing a new file manager | Dolphin already covers every item in requirement 2 (search via Baloo or KIO filenamesearch, archives via Ark, removable drives via Solid/UDisks2, permissions dialog, previews). A new one would be years of work and worse. NEXORA adds theme, defaults, service menus. |
| D15 | Download manager | **NEXORA-built** `nexora-downloadd` (Rust) + QML UI | KGet | KGet is lightly maintained; a small Rust daemon with a clear D-Bus API, resume validation and AppArmor confinement is feasible and testable. Browser downloads remain the browser's. |
| D16 | Updates UI | **Discover** (PackageKit) with offline updates + `unattended-upgrades` | Custom updater | Discover is mature; NEXORA only adds settings for automatic security updates. |
| D17 | Backups / restore | **Vorta + BorgBackup** (encrypted, deduplicated); **Timeshift** (btrfs snapshots) | Déjà Dup; custom | Both packaged in Debian; Borg encryption is well reviewed. |
| D18 | Password manager | **KeePassXC** (also used for secure notes); KWallet stays the Secret Service | Bitwarden client; custom notes app | Offline, audited KDBX4 format; avoids inventing crypto for "secure notes". |
| D19 | Tor | **`torbrowser-launcher`** (contrib) + optional `tor` (main, disabled) | Bundling Tor Browser in the ISO | Tor Browser updates often; the launcher downloads and verifies signed releases. Bundling would ship stale browsers. |
| D20 | VS Code | Microsoft `code` package from `packages.microsoft.com` with a pinned `signed-by` key | VSCodium | Requested by spec. **Licensing caveat in §5** — needs a user decision before any public ISO. |
| D21 | Default browser | Firefox ESR with enterprise policies (no telemetry/studies/Pocket/sponsored, HTTPS-Only, uBlock Origin from `webext-ublock-origin-firefox`) | Chromium | Debian-supported security updates; policy-driven hardening without forking. |
| D22 | Developer containers | **podman** (rootless) + distrobox | Docker | No root daemon; fits least privilege. |

## 2. Language and framework decisions

| # | Area | Choice | Rejected | Rationale |
|---|---|---|---|---|
| L1 | System services | **Rust**, MSRV pinned to trixie's `rustc` (1.85, verify), edition 2021 | Go; C++ | Requested; memory safety for the root helper and network-facing daemons. Building with Debian's toolchain avoids rustup in the pipeline. |
| L2 | Rust crates | `zbus` 5 (D-Bus, pure Rust), `tokio`, `serde`/`toml`, `tracing` + `tracing-journald`, `thiserror`/`anyhow`, `reqwest` (rustls), `rusqlite` (system libsqlite3), `sha2`, `nix`, `clap`; tests: `proptest`, `tempfile`, `wiremock` | dbus-rs (libdbus); openssl bindings | Pure-Rust D-Bus avoids libdbus threading pitfalls. Crates are vendored (`cargo vendor`) and built `--locked --offline`. |
| L3 | AI service | **Python 3.13** (trixie), asyncio | Rust for everything | Requested; fastest path to provider SDKs and evolving LLM tooling. |
| L4 | Claude provider | Official **`anthropic` Python SDK**, streaming, model default `claude-opus-5-5` (user can choose `claude-sonnet-5-5` / `claude-haiku-5-5`), adaptive thinking (always on for Opus 5.5; depth via `output_config.effort`, NEXORA default `low` for chat), custom tools with `strict: true` and `tool_choice: auto`, handle `stop_reason == "refusal"`, server-side fallbacks enabled (`fallbacks: "default"`). Not in Debian → vendored with hash-pinned wheels into `/usr/lib/nexora/assistant/` at package build time. | Raw HTTP | SDK handles streaming, retries, typed errors. Opus 5.5 rejects sampling parameters (`temperature`/`top_p`) and forced `tool_choice`, so the UI exposes "effort" for Claude, not temperature. |
| L5 | Local inference | **Ollama** (changed 2026-11-20 at the owner's request; design: [local-ai-ollama.md](local-ai-ollama.md)): pinned upstream release downloaded and SHA-256-checked by `nexora-ollama-setup` on request, run as a sandboxed system service with its own dynamic user on 127.0.0.1:11434, cloud features off (`OLLAMA_NO_CLOUD=1`) | llama.cpp `llama-server` (the original choice: not in trixie, NEXORA would have to build and update it itself, and has no model manager) | Model download/management and GPU support come with it; running it as its own user with `ProtectHome=yes` keeps it away from users' files. Neither Ollama nor models are in the ISO (1.44 GB archive mostly NVIDIA CUDA libraries; licence per model). |
| L6 | Python deps | From Debian where possible: `python3-httpx`, `python3-pydantic`, `python3-secretstorage`, D-Bus via `python3-dbus-fast` (verify; fallback `python3-jeepney`), `python3-pytest` | pip at runtime | No network at install time; security updates via Debian. |
| L7 | UI | **QML + Kirigami (KF6)**, hosted by thin **PySide6** launchers (verify `python3-pyside6.*` packages; fallback: a 40-line C++ `QQmlApplicationEngine` host) | cxx-qt (Rust ↔ Qt) | cxx-qt is not in Debian and would complicate reproducible packaging. QML talks to daemons over QtDBus; business logic stays in Rust/Python daemons. |
| L8 | Plasma widgets | Pure QML plasmoids (Control Center, Tor status) | C++ applets | No ABI coupling to Plasma libraries; note that some imports used by control widgets (e.g. plasma-pa's QML module) are private and may change at a Debian major upgrade — retested each upgrade. |
| L9 | Overlay window | Assistant overlay as a layer-shell surface via `layer-shell-qt` (verify QML support in trixie); fallback: normal window + KWin window rule (centered, keep-above, skip taskbar) | — | Correct Wayland behaviour for a launcher-like overlay. |
| L10 | Global shortcut | `.desktop` with `X-KDE-Shortcuts=Ctrl+E` + default in `/etc/xdg/kglobalshortcutsrc` (verify mechanism on Plasma 6.3) | khotkeys (removed in Plasma 6) | Works on Wayland through KWin/kglobalaccel. |
| L11 | Packaging | One native Debian source package (`debian/`, format `3.0 (native)`, debhelper 13) → multiple binary packages; `dh-cargo`-style build with vendored crates; `pybuild` | Per-component repos; Snap/Flatpak for NEXORA apps | One version, one changelog, one CI pipeline. |
| L12 | Tests | `cargo test`, `pytest`, `qmltestrunner` with `QT_QPA_PLATFORM=offscreen`, D-Bus integration on a private `dbus-daemon`, `lintian`, `piuparts`, QEMU/OVMF serial boot smoke test, AppArmor denial scan | — | Every layer testable in a container. |

## 3. Package versions in trixie — verified in Phase 2

Full, regenerated list: [verified-versions.md](verified-versions.md) (`nexora verify`).

| Component | Verified | Component | Verified |
|---|---|---|---|
| Linux kernel | 6.12 LTS | systemd | 257 |
| Plasma | 6.3.6 | Qt | 6.8.2 |
| KDE Frameworks | 6.13 | Calamares | 3.3.14 |
| Python | 3.13.5 | rustc / cargo | 1.85.1 |
| GRUB (signed) | 2.12 | cryptsetup | 2.7.5 |
| AppArmor | 4.1.0 | PipeWire | 1.4.2 |
| NetworkManager | 1.52.1 | firewalld | 2.3.1 |
| Firefox ESR | **153 ESR** (not 140) | tor | **0.4.9** (not 0.4.8) |

**Phase 2 verification outcomes that change Phase 1 assumptions**

- **L5 llama.cpp is not packaged in trixie** → planned to build `llama-server` ourselves; superseded on 2026-11-20 by Ollama (also not in trixie; installed from a pinned, checksum-verified upstream release).
- **`anthropic` Python SDK is not packaged** → vendored with hash-pinned wheels, as planned in L4.
- **`grub-btrfs` is not packaged** → snapshot restore goes through Timeshift from the running system or the live ISO; booting snapshots from GRUB is not in 1.0.
- Confirmed available: PySide6 incl. QtQuick/QtDBus (L7), `python3-dbus-fast` 2.44 (L6), `qml6-module-org-kde-layershell` for the Ctrl+E overlay (L9), `systemd-zram-generator` (D8), `kde-config-flatpak`, `keepassxc-full`, `kde-spectacle`, `webext-ublock-origin-firefox`, `torbrowser-launcher` (contrib).
- All Rust crates NEXORA plans to use are packaged in Debian except `wiremock` → a fully offline build against Debian's crate registry is possible for Phase 4 daemons (decision then).
- NetworkManager 1.52 supports every privacy default as a global `[connection]` default, including `dhcp-send-hostname` (now shipped by `nexora-base`).

## 4. Reproducibility

- Builder image pinned by digest; apt sources point at `snapshot.debian.org`
  with a fixed timestamp recorded in `build/snapshot.env`.
- `SOURCE_DATE_EPOCH` from the last commit; live-build `--debian-installer none`,
  deterministic `mksquashfs` options (sorted, fixed timestamps).
- Rust: `Cargo.lock` committed, crates vendored, `--remap-path-prefix`.
- Python: no network at build; vendored wheels pinned by SHA-256.
- Release gate: two independent builds compared with `diffoscope`; differences
  are either fixed or listed in release notes.

## 5. Licensing

| Item | Licence | Notes |
|---|---|---|
| NEXORA code (Rust, Python, QML, scripts) | **GPL-3.0-or-later** | Compatible with Qt (LGPL/GPL), KDE frameworks (LGPL), Papirus (GPL-3.0). SPDX headers, REUSE-compliant. |
| NEXORA artwork (wallpapers, logos, themes) | **CC-BY-SA-4.0** | Original work only. The reference screenshots supplied for this project are design inspiration; none of their wallpapers, icons or logos are copied. |
| Fonts | Inter, JetBrains Mono, Noto — SIL OFL 1.1 | Debian packages `fonts-inter`, `fonts-jetbrains-mono`, `fonts-noto-*`. |
| Icon base | Papirus-Dark — GPL-3.0 | NEXORA overrides folder colours and adds app icons. |
| Firmware | `non-free-firmware` — redistributable, not free software | Included so Wi-Fi/GPU/audio work on common laptops; listed in release notes. |
| **VS Code** | **Microsoft proprietary licence** (Microsoft-built binaries) | The licence restricts redistribution of the software, and telemetry is on by default. Options: **(a)** ship the Microsoft repo + key and install `code` at first boot after the user accepts Microsoft's terms (safe for public ISOs); **(b)** preinstall `code` in the ISO — acceptable for private/internal builds, requires legal review before public distribution; **(c)** VSCodium (MIT, no telemetry, Open VSX marketplace instead of Microsoft's). Build flag `NEXORA_VSCODE=firstboot|preinstall|vscodium`. Default proposed: `firstboot`. |
| Tor Browser | Mozilla/Tor licences; "Tor" trademark | Not redistributed by NEXORA — downloaded by `torbrowser-launcher`. NEXORA does not use the Tor onion logo in its own branding. |
| Debian | Trademark policy | "Based on Debian" is fine; no Debian swirl in NEXORA branding; `ID=nexora`, `ID_LIKE=debian`. |
| AI models | Per model (e.g. Apache-2.0, or custom community licences) | Shown in the model picker before download; NEXORA ships no weights. |
| Anthropic API | Anthropic commercial terms | Users bring their own API key; NEXORA has no relationship with the provider by default. |

## 6. Hardware compatibility (stated limits)

| Area | Expected status | Notes |
|---|---|---|
| Intel/AMD integrated GPUs | Works (Mesa) | Wayland default. |
| NVIDIA | nouveau by default (basic); proprietary driver optional | Proprietary driver via DKMS needs MOK enrollment with Secure Boot; Wayland quality depends on driver version in trixie. |
| Wi-Fi | Intel, Qualcomm/Atheros, MediaTek, most Realtek: works with `firmware-*` | Broadcom often needs `broadcom-sta-dkms` (non-free, DKMS → MOK). |
| Audio | Most laptops incl. SOF (Intel Smart Sound) via `firmware-sof-signed` | Some very new codecs may need newer kernels (trixie-backports). |
| Bluetooth | BlueZ + bluedevil | Firmware dependent. |
| Fingerprint readers | `fprintd` for supported devices only | Many readers unsupported on Linux. |
| Very new hardware | May need `trixie-backports` kernel | Backports kernels are still Debian-signed. |
| Secure Boot | Works with Debian's signed chain | Some OEM firmware ships without the Microsoft 3rd-party UEFI CA enabled; user must enable it in firmware settings. |
| Minimum spec | 2 cores, 4 GB RAM, 32 GB disk | Local AI: 16 GB RAM recommended for 4B-parameter models on CPU; 8 GB works with ~1–2B models. |
| VMs | Hyper-V Gen2, QEMU/KVM (OVMF), VirtualBox 7 | Hyper-V Gen2 Secure Boot requires the "Microsoft UEFI Certificate Authority" template. |
