# NEXORA OS — Progress Record

Updated: 2026-11-20 · Current phase: **3 — verified except the booted-ISO checks (running)**

Status legend: ✅ done/verified · 🟡 in progress · ⬜ not started · ⛔ blocked.
**Rule agreed with the project owner: no phase starts until the previous phase
builds and is verified.**

## Phase status

| Phase | Status | Notes |
|---|---|---|
| 1 — Architecture, threat model, choices, roadmap | ✅ | Documents |
| 2 — Repo + Debian build skeleton | ✅ (one item rides on the Phase 3 ISO) | ISO built and boots to a login prompt; its "Welcome to Debian" identity bug is fixed and confirmed by the Phase 3 ISO boot test |
| 3 — Desktop & integration (part 1: identity, themes, defaults) | 🟡 | Everything verified in containers; Phase 3 ISO build + boot tests running |
| 4 — Files, downloads, audio, Wi-Fi, migration, Tor | ⬜ | Starts when Phase 3 closes |
| 5 — Ctrl+E assistant | ⬜ | |
| 6 — Hardening, installer, updates, recovery | ⬜ | |
| 7 — Tests & docs | ⬜ | |
| 8 — Release validation | ⬜ | |

Deferred from Phase 3 to a later part: Control Center and Tor status plasmoids,
shared `ui/Nexora` QML module (they need the Phase 4/5 services to show real data).

## How verification works (`build/nexora.ps1 <step>`)

| Step | What it proves |
|---|---|
| `verify` | Every package name NEXORA uses exists in Debian 13; key versions; live-build options; Phase 1 assumptions → [docs/verified-versions.md](docs/verified-versions.md) |
| `lint` | `bash -n`/`sh -n` + ShellCheck on all scripts and live-build hooks; XML, JSON, TOML; layout scripts parse as JavaScript |
| `test` | Rust (24), Python (19), design/contrast (9) unit tests |
| `debs` | All packages build offline; lintian clean |
| `test-packages` | Each package installed, exercised and removed in a clean trixie container; file-conflict check against all of Debian 13 |
| `visual` | Real trixie software renders the NEXORA login screen, GRUB menu and Firefox policy page → `out/visual/*.png` |
| `iso` / `smoke` | ISO builds; QEMU boots it to a login prompt and systemd must say "Welcome to … NEXORA OS" |
| `boot-screens` | ISO boots through UEFI + GRUB with emulated graphics; timed screenshots of menu, splash, desktop → `out/boot/` |

## Phase 2 — verification record

| Check | Status |
|---|---|
| Package names/versions vs. trixie, lint, Rust/Python tests, `.deb` build, lintian, `nexora-base` install test | ✅ |
| ISO build (3.0 GB) | ✅ |
| QEMU boot to `nexora login:` with 0 failed units | ✅ |
| System identifies as NEXORA OS at boot | ❌ in the Phase 2 ISO ("Welcome to Debian"): live-build replaced `/etc/os-release` with a copy of Debian's file. Fixed by `distro/live-build/config/hooks/normal/9000-nexora-os-release.hook.chroot`; the boot test now requires the NEXORA greeting. Confirmation: Phase 3 ISO boot test |

## Phase 3 — verification record

| Check | Status |
|---|---|
| Colour tokens → generated KDE/Konsole schemes; every promised contrast pair and all 128 KDE foreground/background pairs ≥ 4.5:1 (7:1 high-contrast text) | ✅ |
| Artwork renders at package build (88 files: wallpapers ×4 sizes, Plymouth frames, GRUB assets and fonts, isolinux splash, icons) | ✅ |
| `nexora-artwork`: Plymouth default via diversion, 36-frame spinner, GRUB theme on `/boot` with all font names matching, SDDM theme, live boot assets, permissions, clean removal | ✅ |
| `nexora-plasma-defaults`: kdeglobals, two look-and-feel packages, all 23 Plasma scripting names exist in trixie's plasmashell, Konsole/SDDM/clipboard defaults, Firefox policy names in firefox-esr's schema, purge | ✅ |
| No file conflicts with Debian 13 (133 files; one shared file covered by a declared `Conflicts`) | ✅ |
| Login screen renders in the real SDDM greeter, no QML errors | ✅ `out/visual/sddm-login.png` |
| GRUB menu renders with the NEXORA theme | ✅ `out/visual/grub-menu.png` |
| Firefox ESR shows all 11 NEXORA policies active | ✅ `out/visual/firefox-policies.png` |
| Phase 3 ISO build | 🟡 running |
| Boot test: NEXORA greeting + login prompt | 🟡 after ISO |
| Boot screens: GRUB → Plymouth splash → live desktop | 🟡 after ISO |
| Hyper-V Gen 2 with Secure Boot (needs your administrator shell) | ⬜ |

### Bugs found and fixed by verification (Phases 2–3)

| Found by | Bug |
|---|---|
| Windows PowerShell 5.1 run | `nexora.ps1` crashed on docker's normal stderr output |
| `debs` | Files from the Windows folder were mode 0777 → world-writable files in packages and the ISO |
| `debs` | Package build ran the test suite with network access |
| lintian | Unversioned dependency on an Essential package; files in `/etc/skel`; hard-coded `update-grub` path |
| `test-packages` | `/usr/lib/os-release` briefly missing during install (unsafe diversion) |
| `test-packages` | GRUB SemiBold and Regular fonts had the same internal name |
| `test-file-conflicts` | Firefox policy file clashed with `debian-edu-config`, then with `firefox-esr-mobile-config` |
| Real Firefox (`about:policies`) | `/etc/firefox-esr/policies/` is **ignored** by Debian's Firefox; moved to `/etc/firefox/policies/` |
| `visual` | SDDM theme: property name reserved by QML (`onAccent`); missing `qt6-svg-plugins` dependency |
| Plymouth log | Boot splash engine `two-step` lives in `plymouth-themes`, which nothing installed → every ISO would have shown a text splash |
| Screenshot review | GRUB countdown text hidden under an oversized bar; login clock showed seconds; empty session picker |
| Boot test | Live system said "Welcome to Debian" (see Phase 2) |
| Own tests | verify script ignored troff escapes; `set -e` traps in 4 scripts; an over-strict API check; a dependency parser that dropped names |

## To finish Phase 3

1. ISO build (running) → `nexora smoke` → `nexora boot-screens`, review screenshots.
2. Hyper-V Gen 2 Secure Boot check — administrator PowerShell:
   `.\build\hyperv\New-NexoraTestVM.ps1 -IsoPath .\out\iso\nexora-0.3.0-amd64.iso`
3. Install Git and initialise the repository (still not done).

## Open decisions (owner input needed)

OD1 VS Code distribution · OD2 apt repo hosting & signing keys · OD3 btrfs vs ext4 ·
OD4 Ctrl+E conflicts · OD5 first-run AI provider · OD6 hosting/CI ·
OD7 package maintainer name/e-mail (placeholder `maintainers@nexora.invalid`) ·
OD8 D-Bus namespace `org.nexora.*` assumes control of `nexora.org`.
