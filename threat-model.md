# NEXORA OS — Threat Model and Privacy Claims

Status: Phase 1 deliverable. Reviewed again at the end of Phases 5, 6 and 8.

Method: assets → adversaries → trust boundaries → threats per area (STRIDE
where useful) → mitigations → **residual risk stated plainly**. Section 7 lists
what NEXORA may and may not claim in UI text and marketing.

---

## 1. Scope and assumptions

**In scope:** a single-user or family laptop/desktop running NEXORA 1.0 from
the official ISO, used for everyday browsing, development and private work.

**Assumptions**

- Firmware and CPU are not malicious (NEXORA cannot defend against a
  compromised UEFI, Intel ME/AMD PSP, or hardware implants).
- The Debian archive signing keys and Debian's Secure Boot signing chain are
  trustworthy.
- The user chooses a reasonable disk-encryption passphrase.
- Users who face a capable, targeted adversary (journalists under state
  surveillance, activists) are directed to Tails or Whonix in the docs.

## 2. Assets

| Asset | Why it matters |
|---|---|
| User files (documents, photos, code) | Confidentiality and integrity |
| Credentials (KWallet, KeePassXC DB, SSH/GPG keys, browser sessions, AI API keys) | Account takeover |
| Assistant conversations and grants | May contain personal context; grants control what the AI can read |
| Network identity (IP, MAC, hostname, DNS queries) | Tracking and linking across networks |
| System integrity (packages, boot chain, policy files) | Persistence of an attacker |
| Windows partition during migration | Must stay unmodified |

## 3. Adversaries

| ID | Adversary | Capability | In scope? |
|---|---|---|---|
| A1 | Thief / finder of a powered-off laptop | Physical access, offline disk reading | Yes |
| A2 | Local network observer (public Wi-Fi, hotel, employer LAN) | Sniffing, spoofing, active scanning | Yes |
| A3 | ISP / DNS resolver operator | Sees destination IPs and plaintext DNS | Partly (documented limits) |
| A4 | Web trackers and websites | Fingerprinting, cookies, cross-site tracking | Partly (Tor Browser / Firefox hardening) |
| A5 | Malicious or compromised app running as the user | Reads everything the user can read | Partly (Wayland, Flatpak, AppArmor deny-lists) |
| A6 | Prompt injection via files, web pages or tool output fed to the AI | Tries to make the assistant leak data or run commands | Yes |
| A7 | Malicious AI provider or model | Sees what is sent to it; may return harmful suggestions | Yes |
| A8 | Supply chain: compromised third-party repo or NEXORA build | Ships malicious packages | Yes (process controls) |
| A9 | Evil maid (repeated physical access to a running/sleeping device) | Tampers `/boot`, installs hardware keylogger | Partly (documented) |
| A10 | Global passive adversary correlating Tor traffic | Watches both ends of Tor circuits | **No** — cannot be defended by an OS |
| A11 | Forensic examiner with a powered-on/suspended device | Cold boot, DMA, memory extraction | **No** (partial mitigations only) |

## 4. Trust boundaries

1. **Network ↔ host**: firewalld (default zone `public`, inbound denied),
   NetworkManager.
2. **User session ↔ root**: polkit + D-Bus policy; only `nexora-privd` and
   upstream system services cross it.
3. **App ↔ app (same user)**: weak by default; strengthened by Wayland,
   Flatpak/bubblewrap and AppArmor profiles for NEXORA daemons.
4. **Assistant service ↔ model provider**: everything sent is listed in the UI
   when the provider is remote.
5. **Model output ↔ execution**: human approval in the Assistant UI, execution
   in a visible terminal as the user.
6. **Boot chain ↔ disk**: Secure Boot covers shim → GRUB → kernel; LUKS2 covers
   root and home; `/boot` and initramfs are not covered (see T-BOOT-2).
7. **Migration ↔ Windows volume**: read-only mount + AppArmor deny-write.

## 5. Threats, mitigations and residual risk

### 5.1 Physical and storage

| ID | Threat | Mitigation | Residual risk |
|---|---|---|---|
| T-STOR-1 | Thief reads disk (A1) | LUKS2 (argon2id, AES-XTS-512) on root+home, offered by default in installer; zram instead of disk swap; `/tmp` is tmpfs (Debian 13 default) | If user declines encryption, no protection. Weak passphrase is brute-forceable. |
| T-STOR-2 | Data recovered from swap/hibernation image | No disk swap by default (zram) → hibernation disabled by default | Users who enable hibernation must use encrypted swap; documented. |
| T-STOR-3 | Data in suspended RAM (A11) | Lock screen on suspend; doc recommends shutdown when crossing borders | Suspend keeps LUKS key in RAM. Cold-boot/DMA attacks are out of scope. |
| T-BOOT-1 | Booting a modified bootloader/kernel | Secure Boot with Debian-signed shim/GRUB/kernel; kernel lockdown (integrity) is automatic under SB in Debian kernels | SB disabled by user → no protection. |
| T-BOOT-2 | Evil maid modifies unencrypted `/boot` initramfs to capture passphrase (A9) | Secure Boot verifies kernel but **not** initramfs. Mitigation in 1.0 is documentation + optional encrypted `/boot` (GRUB-decrypted, PBKDF2 only, slower boot) | Real residual risk. TPM2-measured boot / signed UKI is a post-1.0 item. |
| T-BOOT-3 | DKMS modules (NVIDIA) break Secure Boot | MOK enrollment flow documented; not installed by default | User friction. |

### 5.2 Network

| ID | Threat | Mitigation | Residual risk |
|---|---|---|---|
| T-NET-1 | Inbound attacks on public Wi-Fi (A2) | firewalld default zone `public`: inbound denied except DHCPv6-client; no SSH server installed; no listening NEXORA services (all D-Bus or 127.0.0.1) | Vulnerable client software (browsers) remains attack surface. |
| T-NET-2 | Device tracking across Wi-Fi networks via MAC | NM: `wifi.scan-rand-mac-address=yes`, `wifi.cloned-mac-address=stable` (different MAC per network, stable on the same network); per-connection "random" option | Same network can still recognise a returning device (by design of `stable`). Ethernet MAC preserved by default (802.1X compatibility). |
| T-NET-3 | Hostname leaked via DHCP and mDNS | `dhcp-send-hostname=false` default; generic default hostname `nexora`; Avahi disabled by default (toggle in Privacy Dashboard, needed for some printer discovery) | Users who rename the host or enable Avahi re-expose it on the LAN. |
| T-NET-4 | DNS queries visible to network/ISP (A2, A3) | systemd-resolved; optional "Encrypted DNS" (DNS-over-TLS, strict) with user-chosen resolver | Off by default to keep captive portals working. Encrypted DNS hides queries from the local network, **not** from the chosen resolver; TLS SNI and destination IPs still reveal sites. |
| T-NET-5 | IPv6 address tracking | `ip6-privacy=2` (temporary addresses), `addr-gen-mode=stable-privacy` | Network operator still sees addresses. |
| T-NET-6 | VPN leaks when VPN drops | WireGuard/OpenVPN via NM; doc explains NM has no built-in kill switch; optional firewalld policy "VPN only" (Phase 6, evaluated for feasibility) | Without kill switch, traffic falls back to the normal route. |

### 5.3 Tor and anonymity (must be read before writing any UI text)

| ID | Limitation | What NEXORA does |
|---|---|---|
| T-TOR-1 | **Only Tor Browser traffic goes through Tor.** Every other app (updates, VS Code, Discover, the AI assistant, Firefox ESR, ping, git) uses the normal connection. | Tor status widget says "Tor Browser: connected" — never "You are anonymous" or "System protected". Explainer page states this in the first sentence. |
| T-TOR-2 | **DNS leaks** happen when apps other than Tor Browser resolve names outside Tor. Tor Browser itself resolves through Tor. | Documented; optional `tor.service` + `torsocks`/SOCKS5h instructions for advanced users, with warnings about app-level leaks. |
| T-TOR-3 | **Fingerprinting**: changing Tor Browser's window size, fonts, add-ons or settings makes the user stand out. | NEXORA does not modify Tor Browser or its profile. Docs: don't install add-ons, don't maximise, keep default security level unless needed. |
| T-TOR-4 | **Traffic correlation** by an adversary who watches both the user's connection and the destination (A10). | Out of scope for any OS. Documented. |
| T-TOR-5 | **Hardware/OS identifiers** (MAC, machine-id, disk serials, hostname) can link identities if leaked by non-Tor apps or by the user. Tor Browser does not expose them to websites. | MAC randomisation, generic hostname, no DHCP hostname; doc lists remaining identifiers. |
| T-TOR-6 | **Behavioural linking**: logging into personal accounts inside Tor Browser deanonymises the user. | Docs. |
| T-TOR-7 | Tor use is visible to the ISP (A3) unless bridges are used. | Docs link to Tor Browser's built-in bridge configuration. |
| T-TOR-8 | Supply chain of Tor Browser itself. | Installed by `torbrowser-launcher`, which verifies the Tor Project's signatures; NEXORA does not repackage Tor Browser. |

### 5.4 AI assistant

| ID | Threat | Mitigation | Residual risk |
|---|---|---|---|
| T-AI-1 | Assistant executes a harmful command | No execution tool exists for the model. Commands are *proposals*; execution needs a user click in the UI, typed confirmation for admin/destructive tiers, runs as the user in a visible Konsole; blocked patterns cannot be run from the assistant | A user can still approve a harmful command; classifier can be evaded by unusual syntax (it fails closed to "destructive" when it cannot parse). |
| T-AI-2 | Prompt injection in a file/web page tells the model to exfiltrate data (A6) | Tool outputs are wrapped as untrusted data; no tool can send data anywhere except back to the configured provider; file *content* reads need per-file confirmation; remote provider shows exactly what is attached | If the provider is remote, any content the user approves is sent to that provider. |
| T-AI-3 | Assistant reads sensitive files | Capability grants (`files.search` scoped to chosen folders; `files.read` per file); AppArmor profile denies `~/.ssh`, `~/.gnupg`, `~/.local/share/kwalletd`, browser profiles, `*.kdbx` even if granted | Files the user explicitly grants can be read and, with a remote provider, uploaded. |
| T-AI-4 | Assistant service grants itself permissions | Grants written only by the UI; AppArmor denies the service write access to the grant file | A root compromise or an unconfined same-user process can still edit grants. |
| T-AI-5 | API key theft | Key stored in KWallet (Secret Service), never in config/env/logs | Any process of the user that can talk to an unlocked KWallet can request secrets (KWallet's per-app prompts mitigate partially). |
| T-AI-6 | Remote provider retains/uses data (A7) | Remote provider off by default; first-use dialog names the provider and links its data policy; local model option; per-message indicator "sent to <provider>". Ollama's own cloud models and web search (which send prompts to ollama.com) are off (`OLLAMA_NO_CLOUD=1`); turning them on is treated like any remote provider | Out of NEXORA's control once sent. |
| T-AI-7 | Hallucinated system advice | Answers cite NEXORA docs when available; proposals show risk tier; UI copy: "Check before running" | Inherent to LLMs. |
| T-AI-8 | Malicious model file | Models come from Ollama's registry (content-addressed by SHA-256) only when the user downloads them; model files are data parsed by Ollama, which runs as its own dynamic user with `ProtectHome=yes`, `ProtectSystem=strict` and an AppArmor profile, so a parser bug does not reach users' files | Parser bugs in Ollama's engine; a compromised service can still use the network (needed for model downloads). |
| T-AI-10 | Another local program or user uses the Ollama API (`127.0.0.1:11434` has no authentication) | Listens on loopback only; Ollama keeps no conversations on the server side; documented in the UI | Any local process can run or list models and use CPU/GPU time. |
| T-AI-11 | Tampered Ollama download (supply chain) | `nexora-ollama-setup` installs only the release pinned in the NEXORA package and checks its SHA-256 before extracting; files root-owned and read-only | NEXORA trusts upstream's build (not reproducible by NEXORA); the pinned checksum only proves it is the file the maintainers reviewed. |
| T-AI-9 | Ctrl+E overlay spoofed by another app | Wayland: other clients cannot inject input into or read the overlay; proposals are only accepted from the service over the session bus | A same-user process could impersonate `org.nexora.Assistant1` before it starts (D-Bus name squatting); mitigated by D-Bus activation and checking the owner's executable via `GetConnectionUnixProcessID` + `/proc/<pid>/exe`. |

### 5.5 Privileged helper and system integrity

| ID | Threat | Mitigation | Residual risk |
|---|---|---|---|
| T-PRIV-1 | Local user escalates via `nexora-privd` | Typed enum arguments only, no paths/commands from callers; polkit per method; atomic writes from fixed templates; systemd sandbox; fuzzed argument parsing; unit tests for every method | Bugs in zbus/polkit/systemd. |
| T-PRIV-2 | Confused deputy via polkit `auth_admin_keep` | Keep window = polkit default (5 min); actions narrowly scoped | Within the window, the same session can repeat the action without re-auth. |
| T-SUP-1 | Compromised third-party repo (Microsoft) (A8) | `signed-by=` per-repo keyring; apt pinning: Microsoft repo may only provide `code`; no other third-party repos by default | Microsoft repo compromise could still ship a malicious `code`. |
| T-SUP-2 | Compromised NEXORA build | Builds in pinned containers from snapshot.debian.org, `Cargo.lock` + vendored crates, reproducibility check (build twice, diffoscope) before release, signed SHA256SUMS | Signing key custody is an open decision (roadmap). |
| T-SUP-3 | Malicious Flatpak/third-party app | Flathub off until user enables; Flatpak permissions visible in Privacy Dashboard (KDE Flatpak KCM) | Flatpak sandboxes are only as strict as each app's manifest. |
| T-UPD-1 | Unpatched vulnerabilities | Automatic security updates on by default | Kernel/session updates need a reboot; user can postpone. |

### 5.6 Windows migration

| ID | Threat | Mitigation | Residual risk |
|---|---|---|---|
| T-MIG-1 | Windows volume modified/corrupted | UDisks2 mount with `ro`; never mount rw, never remove `hiberfil.sys`; AppArmor denies writes under `/media/**` for `nexora-migrated` | None known for data; reading a dirty NTFS volume may show stale data (documented). |
| T-MIG-2 | BitLocker recovery key exposure | Key typed into the wizard, passed over the session bus to UDisks2, never stored or logged | Key is in process memory during unlock. |
| T-MIG-3 | Importing secrets unintentionally | Browser passwords, cookies and history are explicitly not imported | — |

### 5.7 Desktop features

| ID | Threat | Mitigation | Residual risk |
|---|---|---|---|
| T-DESK-1 | Clipboard history keeps passwords | Klipper: history not saved across sessions, size 20; KeePassXC marks password copies (`x-kde-passwordManagerHint`) so Klipper skips them, and clears its own copies after 10 s | Apps that don't set the hint will land in history until session end. |
| T-DESK-2 | Camera/mic used without the user noticing | PipeWire + portals; Plasma tray shows mic-in-use; Privacy Dashboard lists apps currently using mic/camera | Hardware camera LED is outside OS control on many laptops. |
| T-DESK-3 | Screenshots/screen sharing by apps | Wayland: capture only via portal with user consent | — |
| T-DESK-4 | Download writes autostart entry or shell rc file | AppArmor persistence deny-list for `nexora-downloadd` and `nexora-migrated` | Browser downloads are governed by the browser, not by NEXORA. |

## 6. Network egress inventory (default install, nothing configured)

Every outbound connection a fresh NEXORA install makes without user action.
This list must stay accurate; Phase 8 verifies it by capturing traffic in the VM.

| Destination | Purpose | When | Can be disabled |
|---|---|---|---|
| `deb.debian.org`, `security.debian.org` | apt metadata + security updates | daily (apt timers) | Yes (Privacy Dashboard → Updates) |
| `packages.microsoft.com` | VS Code updates (if VS Code repo enabled) | daily with apt | Yes |
| NTP pool (`debian.pool.ntp.org` via systemd-timesyncd) | Clock sync | boot + periodic | Yes |
| NetworkManager connectivity check | **Not enabled** in Debian by default; NEXORA keeps it off | — | — |
| `torproject.org` | Tor Browser download/verify | Only when user launches Tor Browser installer/launcher | n/a |
| Flathub, ODRS reviews (Discover) | App store | Only after the user enables Flathub / opens Discover | Yes |
| `store.kde.org` (KNewStuff) | "Get new wallpapers/themes" | Only when the user opens those dialogs | n/a |
| AI provider | Assistant | Only when configured by the user | Yes |
| Model catalogue host | Local model download | Only when the user downloads a model | n/a |

Not present: telemetry, crash reporting uploads (DrKonqi upload disabled;
reports saved locally), popularity-contest, KDE UserFeedback (left at its
default "off"), Firefox telemetry/studies (disabled by enterprise policy),
VS Code telemetry (set to `off` in `/etc/skel` settings; the user can change it).

## 7. Privacy claims policy

**Allowed claims (true, with the stated limits nearby):**

- "NEXORA contains no telemetry. Components that could send usage data are
  turned off; see the list in Privacy Dashboard."
- "Your disk is encrypted with LUKS2 (if selected during install). This
  protects data when the computer is off."
- "Tor Browser sends its own traffic through the Tor network. Other apps use
  your normal connection."
- "Wi-Fi MAC address is randomised per network."
- "The assistant never runs commands by itself. You approve each one."
- "With a local model, conversations stay on this computer."

**Forbidden claims:**

- "Anonymous OS", "untraceable", "military-grade", "100% private", "hack-proof".
- Any statement implying system-wide Tor routing.
- "Encrypted DNS makes browsing private."
- "Secure Boot protects against evil-maid attacks."
- A numerical "privacy score".

## 8. Open risks tracked for later phases

1. Measured boot / signed initramfs (UKI + TPM2) to close T-BOOT-2 — post-1.0 evaluation.
2. VPN kill switch design (T-NET-6) — feasibility in Phase 6.
3. D-Bus name squatting checks (T-AI-9) — implemented and tested in Phase 5.
4. NEXORA signing key custody and repository hosting (T-SUP-2) — decision needed before first public ISO.
5. AppArmor profile breakage on Debian point releases — profiles run in `complain` mode in nightly builds and `enforce` in releases; ISO smoke tests check for denials.
