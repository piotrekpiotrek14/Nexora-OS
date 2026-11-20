# NEXORA OS — Development Roadmap (Phases 2–8)

Status: Phase 1 deliverable.

## Change from the original phase order: build the ISO from Phase 2 onward

The brief puts "produce a bootable ISO" in Phase 8. That is risky: problems
with live-build, Secure Boot, firmware or Calamares would surface last, after
months of work. NEXORA instead builds a **walking skeleton**: Phase 2 produces
a bootable (plain) ISO, and every later phase adds to an image that is booted
in a VM at the end of each phase. Phase 8 becomes release hardening and full
validation rather than the first boot.

## Build and test environment on the current machine

Checked on the development host (Windows 11 Pro):

| Tool | Present | Use |
|---|---|---|
| Docker (Linux containers) | Yes (29.x) | All builds: `.deb` packages and the ISO (live-build needs a `--privileged` container for loop mounts and chroots) |
| Hyper-V | Yes | GUI validation: Generation 2 VM, Secure Boot with "Microsoft UEFI Certificate Authority" template, 4 vCPU, 8 GB RAM, 64 GB disk |
| WSL2 | No | Not required; can be added with `wsl --install` if container builds hit limits |
| QEMU | No (on host) | Runs inside the builder container for headless serial-console smoke tests (no KVM acceleration on Windows → slow but workable for boot checks) |
| Rust toolchain | No (on host) | Not needed on host; Debian's `rustc` inside the container is the build toolchain |
| Git | No | Install before Phase 2 (`winget install Git.Git`) so the repository has history |

## Phase overview

| Phase | Theme | Rough size (one engineer) | Exit criterion |
|---|---|---|---|
| 2 | Repo + build system + bootable skeleton ISO | 1–2 weeks | Plain NEXORA-named Plasma ISO boots in Hyper-V with Secure Boot on |
| 3 | Desktop: branding, theme, layout, integration | 2–4 weeks | ISO shows full NEXORA look from GRUB to desktop |
| 4 | Files, downloads, audio, Wi-Fi, migration, Tor | 4–6 weeks | Downloads + Migration apps work end-to-end in VM |
| 5 | Ctrl+E assistant + permissions + command guard | 3–5 weeks | Assistant works with local and remote provider; security tests pass |
| 6 | Hardening, privileged helper, installer, updates, recovery | 3–5 weeks | Encrypted install from ISO, Privacy Dashboard live, auto-updates configurable |
| 7 | Tests, docs, troubleshooting | 2–3 weeks | CI green; user guide + troubleshooting complete |
| 8 | Release validation | 1–2 weeks | Validation matrix passed; reproducibility checked; signed checksums |

Estimates assume code is generated quickly; most calendar time goes to
building, booting and debugging in VMs.

---

## Phase 2 — Repository and foundational Debian build

**Deliverables (files)**

- `README.md`, `PROGRESS.md`, `REUSE.toml`, `LICENSES/GPL-3.0-or-later.txt`, `LICENSES/CC-BY-SA-4.0.txt`, `.editorconfig`, `.gitignore`, `Makefile`
- `build/Containerfile.trixie`, `build/snapshot.env`, `build/build-debs.sh`, `build/build-iso.sh`, `build/smoke-test.sh`
- `distro/live-build/auto/{config,build,clean}`
- `distro/live-build/config/package-lists/{base,desktop,firmware,apps,dev,live}.list.chroot`
- `distro/live-build/config/archives/{microsoft-vscode.list.chroot,microsoft-vscode.key.chroot}` (behind `NEXORA_VSCODE`)
- `distro/live-build/config/hooks/normal/0100-nexora-os-release.hook.chroot` (temporary until `nexora-base` exists)
- `distro/live-build/config/includes.chroot/etc/...` minimal (hostname, NM conf)
- `debian/{control,rules,changelog,copyright,source/format}` with `nexora-base` and `nexora-desktop` (metapackage) only
- `interfaces/*.xml` — first drafts of the four D-Bus interfaces
- `crates/Cargo.toml` workspace + `nexora-common` (logging, config, errors) with tests
- `python/nexora_assistant/pyproject.toml` skeleton + `tests/`

**Acceptance criteria**

1. `make debs` and `make iso` succeed inside Docker from a clean checkout.
2. `apt-cache policy` output for the §3 version table of `technology-choices.md` captured in `docs/verified-versions.md`.
3. ISO boots in Hyper-V Gen2 with Secure Boot **on** to a Plasma Wayland live session; `mokutil --sb-state` → enabled; `/etc/os-release` shows NEXORA.
4. ISO also boots in BIOS mode (QEMU SeaBIOS) to multi-user target.
5. `cargo test` and `pytest` run (trivially) in CI container.

## Phase 3 — Desktop environment and system integration

**Deliverables**

- `artwork/` sources: logo, wordmark, 3 original wallpapers (4K + 1080p), Plymouth theme, GRUB theme, SDDM theme, app icons, folder recolour script
- `plasma/look-and-feel/org.nexora.desktop/` (defaults, layout script `layout.js` for top bar + dock, splash)
- `plasma/colors/NexoraDark.colors`, `NexoraDarkHC.colors`; Plasma theme `plasma/desktoptheme/nexora/`
- `plasma/plasmoids/org.nexora.controlcenter/`, `plasma/plasmoids/org.nexora.torstatus/` (status logic in Phase 4)
- `ui/Nexora/` shared QML module (tokens, Card, StatusPill, RiskBadge, Stepper)
- `/etc/xdg` defaults: kdeglobals, kwinrc (blur, animation speed), plasmarc, klipperrc (privacy), dolphinrc, konsolerc + profile, kscreenlockerrc
- `/etc/skel`: VS Code telemetry off, Firefox ESR policies (system-wide `policies.json`)
- `debian/` adds `nexora-artwork`, `nexora-plasma-defaults`
- `tests/design/contrast_check.py`

**Acceptance criteria**: full visual chain GRUB → Plymouth → SDDM → Plasma
matches `design-system.md`; contrast check passes; layout survives logout/login;
"Classic" layout switch works; idle RAM within budget.

## Phase 4 — File manager, downloads, audio, Wi-Fi, Windows migration, Tor

**Order of work (updated 2026-11-20 with the owner's additions)**

| Part | Content | Design |
|---|---|---|
| 4a | Download service `nexora-downloadd` (Rust) + package | interfaces/org.nexora.Downloads1.xml |
| 4b | Network privacy: nftables, kill switch (VPN-only / Tor-only), identity change ("IP changer"), proxy chains (proxychains-ng) — added by the owner | [network-privacy.md](network-privacy.md) |
| 4c | Local AI runtime: Ollama (`nexora-ollama`: pinned, checksum-verified setup, sandboxed service, `ollama` command) — added by the owner | [local-ai-ollama.md](local-ai-ollama.md) |
| 4d | Downloads window + Dolphin service menus | below |
| 4e | Windows migration (`nexora-migrate`) | below |
| 4f | Tor status and Control Center widgets, shared `ui/Nexora` QML module | below |
| 4g | ISO with all of the above, boot and VM checks | — |

**Deliverables**

- Dolphin: service menus (`Open in VS Code`, `Verify SHA-256`, `Open terminal here`), Information panel on, preview plugins installed, Baloo indexing **content off by default** (filename only) for privacy
- `crates/nexora-downloadd/` + `ui/downloads/` + D-Bus service files + systemd user unit + AppArmor profile; unit tests (range resume, `If-Range` invalidation, filename sanitisation, disk-full handling) against a local `wiremock` server
- Audio: WirePlumber/plasma-pa defaults; Control Center volume/mic tiles wired
- Wi-Fi: NM privacy conf (`security/networkmanager/90-nexora-privacy.conf`); Control Center Wi-Fi tile; captive-portal behaviour documented
- `crates/nexora-migrate/` (library + daemon) + `ui/migrate/`; tests with NTFS and exFAT images created in the container (`mkntfs`, `mkfs.exfat`), BitLocker read-only path verified or explicitly marked unsupported
- Tor: `torbrowser-launcher` in package list, dock entry, Tor status plasmoid (reads Tor Browser's control port when available, otherwise "running/not running"), explainer page `docs/user/tor.md` linked from the widget

**Acceptance criteria**: pause/resume survives daemon restart and reboot;
migration from a Windows 11 evaluation VM disk copies all selected categories,
verifies hashes, and the source volume's NTFS checksum (`sha256sum` of the raw
partition) is identical before and after; Tor widget never says "anonymous".

## Phase 5 — Ctrl+E AI assistant

**Deliverables**

- `python/nexora_assistant/`: service entry point, provider interface, `providers/anthropic.py` (official SDK, streaming, refusal handling, fallbacks), `providers/ollama.py` (Ollama on 127.0.0.1, see local-ai-ollama.md), permission store (read-only to service), tools (`get_system_info`, `get_disk_usage`, `search_files`, `read_file`, `propose_command`, `open_settings_page`, `search_nexora_docs`), history store with retention, offline "help mode" (docs search without any model)
- `crates/nexora-cmdguard/`: shell-word parser, risk classifier with tiers, `exec` subcommand; property tests and a corpus of ≥ 300 labelled commands
- `ui/assistant/`: overlay + full window, proposal card, permission prompts, provider setup (API key → KWallet), Ollama model manager (installed models, download with size/RAM/licence shown first, delete)
- Ctrl+E registration, single-instance toggle, D-Bus owner verification
- AppArmor profiles for assistantd, UI, cmdguard (Ollama's profile ships with 4c)
- Security tests: prompt-injection corpus (files that instruct the model to run/send things) must produce zero executions and zero unapproved file reads; grant file not writable by service (AppArmor denial observed)

**Acceptance criteria**: works with no provider (help mode), local model, and
Claude; every command path requires a click; destructive tier requires typed
confirmation; history clear removes all rows; nothing is sent remotely when
local provider is selected (verified with a network capture).

## Phase 6 — Security hardening, installer, updates, recovery

**Deliverables**

- `security/`: sysctl (`kernel.kptr_restrict=2`, `kernel.dmesg_restrict=1`, `kernel.yama.ptrace_scope=1`, `kernel.unprivileged_bpf_disabled=1`, `net.core.bpf_jit_harden=2`, `fs.protected_*`), coredump policy, firewalld default zone + NM zone mapping, resolved conf (DoT toggle), Avahi default-off, polkit actions/rules
- `crates/nexora-privd/` + `ui/privacy/` (Privacy Dashboard) + D-Bus policy + systemd unit with sandboxing; tests run privd against a private system bus with a mock polkit
- `distro/calamares/`: settings.conf, module configs (partition with LUKS2 argon2id default-on, btrfs `@`/`@home`, separate `/boot`, no swap partition), users (sudo, no root password, strong password check), branding + QML slideshow, removal of live packages after install
- Updates: `unattended-upgrades` defaults, privd templates, reboot-needed notification, Discover offline updates
- Recovery: Timeshift preconfigured for btrfs; Vorta installed; `nexora-collect-logs`; `docs/user/recovery.md` (GRUB recovery mode, booting the live ISO, `cryptsetup open`, chroot repair, Timeshift restore)
- Secure Boot validation notes + MOK guide for DKMS

**Acceptance criteria**: "Erase disk" install with encryption in Hyper-V
boots with Secure Boot on and asks for the LUKS passphrase in Plymouth;
manual partitioning works; Privacy Dashboard reflects real state and every
toggle round-trips; AppArmor shows NEXORA profiles in enforce mode with no
denials during a scripted session; `lynis audit system` report attached
(informational).

## Phase 7 — Tests, documentation, troubleshooting

**Deliverables**

- CI definition (GitHub Actions or self-hosted runner) running all `make test` targets + lintian + piuparts on every change; nightly ISO build + QEMU smoke test
- `docs/user/`: getting started, Ctrl+E assistant, privacy & Tor, migration from Windows, backups, updates, recovery
- `docs/troubleshooting/`: no Wi-Fi, no sound, Secure Boot errors, NVIDIA, Calamares failures, AppArmor denials, assistant provider errors, reading logs
- `docs/developer/`: build from source, add a package, D-Bus API reference generated from `interfaces/`

**Acceptance criteria**: a new contributor builds the ISO from docs alone;
each troubleshooting page reproduced once in a VM.

## Phase 8 — Release validation

**Validation matrix**

| Scenario | Hyper-V Gen2 SB on | QEMU OVMF SB off | QEMU BIOS |
|---|---|---|---|
| Live boot to desktop | ✔ required | ✔ | ✔ |
| Erase disk + LUKS2 install | ✔ | ✔ | ✔ |
| Install alongside Windows 11 (eval VM) | ✔ | — | — |
| Manual partitioning | — | ✔ | — |
| Migration from Windows partition | ✔ | — | — |
| Assistant: local + remote | ✔ | — | — |
| Egress capture matches threat-model §6 | ✔ | — | — |

**Release steps**: two independent builds + `diffoscope`; `SHA256SUMS` +
detached GPG signature; release notes listing non-free firmware, VS Code
licence handling, known hardware limits and open risks.

---

## Open decisions needed from the project owner

| # | Decision | Default proposed | Needed by |
|---|---|---|---|
| OD1 | VS Code distribution: `firstboot` / `preinstall` / `vscodium` | `firstboot` for public ISOs, `preinstall` allowed for private builds | Phase 2 (package lists) |
| OD2 | NEXORA apt repository hosting and signing-key custody | Static hosting + offline primary key, signing subkey on build machine | Before first public ISO |
| OD3 | Default filesystem: btrfs (snapshots) vs ext4 | btrfs | Phase 6 |
| OD4 | Ctrl+E as a *global* shortcut steals Ctrl+E from apps (VS Code "Go to File", LibreOffice "Center", Firefox search) | Keep Ctrl+E per spec; offer Meta+Space as one-click alternative in Welcome app | Phase 5 |
| OD5 | First-run AI: no provider vs prompt to download a local model | No provider; Welcome app offers local model download | Phase 5 |
| OD6 | Project hosting and CI (GitHub, GitLab, self-hosted) | GitHub + Actions for code; ISO builds on a self-hosted runner (privileged containers) | Phase 7 |
