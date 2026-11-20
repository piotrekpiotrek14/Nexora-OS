# NEXORA OS — Local AI with Ollama

Status: design, 2026-11-20, at the owner's request. Ollama replaces llama.cpp
`llama-server` as NEXORA's local model runtime (technology choice L5).
Implementation: Phase 4 part 4c (package, service, `ollama` command);
Phase 5 (Ctrl+E assistant provider). Threat model: T-AI-6, T-AI-8, T-AI-10, T-AI-11.

Facts checked on 2026-11-20 (docs.ollama.com/linux, /faq, /cloud and the
GitHub releases page):

| Fact | Value |
|---|---|
| In Debian trixie? | No (only the Python client `python3-ollama`) |
| Current release | v0.40.16, 2026-11-09 (v0.40.17 is a pre-release) |
| Linux amd64 archive | `ollama-linux-amd64.tar.zst`, 1.44 GB, SHA-256 published on the release page |
| Extra GPU archives | `ollama-linux-amd64-rocm.tar.zst` (AMD, 1010 MB); arm64 and Jetson variants |
| Default listen address | `127.0.0.1:11434` |
| Cloud features | Cloud models and web search send prompts to ollama.com; turned off with `OLLAMA_NO_CLOUD=1` (log shows `Ollama cloud disabled: true`) |
| Updates on Linux | No automatic updates documented (re-install to update) |

To verify during 4c: the archive's directory layout and licence file, whether
the release has a checksum file for automation, and proxy support for model
downloads (`HTTPS_PROXY`).

## 1. What the user gets

- **Ctrl+E assistant on this computer**: provider "Ollama (on this computer)".
  Prompts and answers do not leave the computer.
- The **`ollama` command** in a terminal (`ollama run`, `ollama pull`,
  `ollama list`) and the Ollama API on `127.0.0.1:11434` for other programs.
- **Model manager** in the assistant settings: installed models, download
  (with size, RAM needed and the model's licence shown first), delete.

## 2. Installation: small package, Ollama downloaded on request

The 1.44 GB archive is mostly NVIDIA CUDA libraries. Putting it in the ISO
would add well over 1.5 GB and mean NEXORA redistributes NVIDIA's CUDA
runtime under NVIDIA's licence. Models must be downloaded anyway, so local AI
needs a network connection at setup in any case. NEXORA therefore follows the
pattern of Debian's `torbrowser-launcher` (download from upstream, verify,
install):

- **`nexora-ollama`** (Architecture: all, a few kB, in the ISO) contains:
  - `release.toml`: pinned version, download URL on `github.com/ollama/ollama/releases`
    and SHA-256 for each architecture, plus the ROCm add-on;
  - `nexora-ollama-setup`: downloads the archive over HTTPS as an
    unprivileged user, **checks the SHA-256 before extracting anything**,
    extracts it into a staging directory, moves it to
    `/opt/nexora/ollama/<version>/` (root-owned, read-only), switches the
    `current` link atomically and restarts the service. A failed or
    interrupted download changes nothing; the previous version stays until
    the new one is verified;
  - the systemd unit, its AppArmor profile, and `/usr/bin/ollama`, which
    until setup only explains how to set up local AI.
- Setup starts from the Welcome app or the assistant settings (polkit,
  `auth_admin_keep`, through `nexora-privd`), or with `sudo nexora-ollama-setup`.
- **Updates**: a NEXORA package update changes the pinned version and
  checksum (security updates arrive through unattended-upgrades). If local
  AI is set up, the new version is installed automatically unless the
  connection is metered; then the tray asks first.
- **GPUs**: CPU always works. NVIDIA GPUs work with the CUDA libraries in the
  main archive once the NVIDIA driver is installed (Debian non-free). For AMD
  GPUs, setup offers the ROCm add-on (about 1 GB) when it detects one.
- **Removal**: removing the package removes Ollama; `apt purge` also deletes
  the downloaded models in `/var/lib/ollama` (the UI warns how much space that is).

## 3. Service

`nexora-ollama.service`, a system unit started on demand:

```ini
[Service]
ExecStart=/opt/nexora/ollama/current/bin/ollama serve
DynamicUser=yes
StateDirectory=ollama
Environment=HOME=/var/lib/ollama OLLAMA_MODELS=/var/lib/ollama/models
Environment=OLLAMA_HOST=127.0.0.1:11434 OLLAMA_NO_CLOUD=1
SupplementaryGroups=render video
ProtectHome=yes
ProtectSystem=strict
PrivateTmp=yes
NoNewPrivileges=yes
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6 AF_NETLINK
```

- **Own user, not the logged-in user**: running as an empty dynamic user
  with `ProtectHome=yes` means Ollama cannot read anyone's files. That is
  less privilege than running it in the user's session, and models are
  downloaded once for all users.
- **Not running when unused**: the assistant starts it when needed (a polkit
  rule lets the active local session start and stop this one unit without a
  password); "start at boot" is an option.
- Upstream's own unit (`User=ollama`, models in `/usr/share/ollama`) is not
  used; NEXORA's keeps state in `/var/lib/ollama`, as Debian policy expects.

## 4. Privacy and honest limits

- **Local by default**: with `OLLAMA_NO_CLOUD=1` Ollama's cloud models and
  web search are off, so prompts are not sent to ollama.com. Turning them on
  is a separate setting treated like any remote provider (T-AI-6): first-use
  notice, per-message "sent to ollama.com" label.
- **Network use** only when the user asks: downloading Ollama (github.com)
  and models (Ollama's registry). These requests show your IP address and
  which model you download.
- **No account**: NEXORA never signs in to ollama.com. Ollama creates a key
  pair in its own home on first start; it is used only for sign-in and
  publishing models, which NEXORA does not do.
- **No password on the local API**: any program or user on this computer can
  use `127.0.0.1:11434` (run models, list installed models). Ollama does not
  store conversations on the server side, so they cannot read your chats
  through it, but they can use your models and CPU/GPU (T-AI-10).
- **Speed**: on CPU only, small models (about 1–4 billion parameters) answer
  at a few words per second; the model manager shows size and memory needed
  before downloading.
- **Model licences** differ per model (shown before download, from the
  model's own metadata).

## 5. Ctrl+E assistant (Phase 5)

- Settings provider values become `none | ollama | anthropic`
  (`local` is renamed; nothing was released with the old name). The endpoint
  must be on this computer (`ollama_endpoint`, default `http://127.0.0.1:11434`).
- `providers/ollama.py`: Ollama's `/api/chat` with streaming and tool calls
  for models that support tools; client library (`python3-ollama` from
  Debian or `httpx`) chosen in Phase 5 after checking trixie's version.
- Tools go through the same `nexora-cmdguard` path as every provider:
  destructive commands never run without the user's explicit confirmation,
  whichever model asks.
- With no model installed the assistant stays in help mode (docs search).

## 6. Tests (4c acceptance)

- Setup script against a local fake release: wrong checksum → nothing
  installed; interrupted download → nothing installed; re-run is a no-op;
  failed update keeps the old version.
- Real setup in the builder container: pinned archive (cached) → checksum →
  `ollama serve` with the unit's environment → `/api/version` reports the
  pinned version, log shows `Ollama cloud disabled: true`, a small model is
  pulled and answers a prompt on CPU.
- `systemd-analyze security` for the unit; AppArmor profile compiles; in the
  VM the service runs as a dynamic user and cannot read `/home`.
