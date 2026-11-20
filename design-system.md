# NEXORA OS — Design System ("Nexora Dark")

Status: Phase 1 deliverable. Implemented in Phase 3 as a Plasma color scheme,
Plasma theme, look-and-feel package, shared QML module `Nexora`, and artwork.

Direction: dark, calm, premium. Cyan is the working accent (focus, selection,
primary actions); purple is the identity accent (brand, AI assistant). The
cyan→purple gradient appears only at brand moments (boot, login, assistant
orb), never as UI chrome. Performance and legibility win over effects.

---

## 1. What we take from the reference screenshots

The screenshots supplied with the brief are mood references. NEXORA reuses
ideas, not assets.

| Reference | Idea adopted | Where in NEXORA |
|---|---|---|
| Plasma setup with blue/purple folders, media widget, clock, "Status and Notifications" grid | Tile grid for system toggles; colour-tinted folders; floating widgets with soft borders | Control Center plasmoid; Papirus-Dark folder overrides; optional clock widget |
| COSMIC Files list view | Calm list rows, pill-shaped selection in sidebar, thin accent window outline | Dolphin defaults (details view option), sidebar selection style |
| Glass AI chat with "Last chats" sidebar | Conversation history sidebar, centred empty-state greeting, composer with chips | Assistant full window |
| Neon cyberpunk Plasma | **Rejected** as default: too saturated, poor contrast in places | — |
| Spacedrive grid + inspector + view-options popover | Right-side inspector with metadata; view-options popover with size/gap sliders; path bar at bottom | Dolphin Information panel on by default; Downloads/Migration detail panes |
| Local AI app with collapsible "Thought for N seconds" | Collapsible reasoning/tool-activity section | Assistant shows tool activity ("Searched ~/Documents") collapsed by default |
| CachyOS quick-settings tiles and pill top bar | Rounded tiles, sliders for brightness/volume, top bar pills | Control Center plasmoid, top bar |
| Purple neon Arch with ring meters | Ring meters for CPU/RAM | Optional system-monitor desktop widget (off by default) |
| Nautilus header bar | Integrated search and view switcher in the header | Dolphin toolbar layout |
| Orb-style voice assistant | Gradient ring as the assistant's "thinking" indicator; centred response card; pill composer | Ctrl+E overlay |
| Three-pane AI studio with settings sliders and connection status | Right settings drawer (provider, model, effort), status bar "Connected · local" | Assistant full window |

## 2. Colour tokens

All UI text pairs meet WCAG 2.2 AA (4.5:1 for body text). Ratios below are
approximate and are re-checked automatically by `tests/design/contrast_check.py`
(Phase 3).

| Token | Hex | Use |
|---|---|---|
| `bg.base` | `#0B0F17` | Desktop backdrop under windows, window background |
| `bg.surface1` | `#111723` | Sidebars, panels, top bar, dock |
| `bg.surface2` | `#161D2B` | Cards, popovers, menus |
| `bg.surface3` | `#1C2433` | Hover, pressed, raised controls |
| `border.subtle` | `#FFFFFF14` (8 %) | Hairlines, card outlines |
| `border.strong` | `#FFFFFF24` (14 %) | Input outlines, separators in dense views |
| `text.primary` | `#E6EAF2` | Body text (≈15.8:1 on `bg.base`) |
| `text.secondary` | `#9AA4B5` | Labels, metadata (≈7.1:1 on `bg.surface1`) |
| `text.disabled` | `#5E6878` | Disabled only (exempt from contrast rule) |
| `accent.cyan` | `#22D3EE` | Focus ring, selection, links, primary buttons (≈10.6:1 on `bg.base`) |
| `accent.cyan.on` | `#0B0F17` | Text on cyan buttons (≈10.6:1) |
| `accent.cyan.tint` | `#22D3EE29` (16 %) | Selected row background |
| `accent.purple` | `#A78BFA` | Assistant identity, secondary highlights, text-safe purple (≈7:1 on `bg.base`) |
| `accent.purple.fill` | `#7C3AED` | Purple buttons/badges with white text (≈5.7:1) |
| `brand.gradient` | `#22D3EE → #8B5CF6`, 135° | Logo, boot splash, SDDM accent line, assistant orb only |
| `status.ok` | `#34D399` | Connected, protected, up to date |
| `status.warn` | `#FBBF24` | Needs attention |
| `status.danger` | `#F87171` | Errors, destructive actions |
| `status.info` | = `accent.cyan` | Informational |

Status meaning is never carried by colour alone: every status pill has an icon
and a word ("On", "Off", "Needs restart").

**High-contrast variant** (`Nexora Dark HC`): `text.secondary → #C9D1DE`,
borders at 30 %, no translucency, focus ring 3 px.

## 3. Typography

| Role | Font | Size (pt, Plasma scale) | Weight |
|---|---|---|---|
| UI default | Inter | 10 | 400 |
| Small / metadata | Inter | 9 | 400 |
| Section heading | Inter | 11 | 600 |
| Page title | Inter | 15 | 600 |
| Display (welcome, login clock) | Inter | 28–48 | 300 |
| Monospace (terminal, code, commands in assistant) | JetBrains Mono | 10 | 400 |
| Fallback / emoji / CJK | Noto Sans, Noto Color Emoji, Noto CJK | — | — |

Hinting slight, subpixel off (sharper on HiDPI, consistent in screenshots).

## 4. Shape, spacing, elevation, motion

- **Spacing:** 4 px grid; mapped to `Kirigami.Units.smallSpacing` (4) and
  `largeSpacing` (8/12/16 multiples).
- **Radii:** controls 6 px · cards/popovers/menus 10 px · panels, dock, overlay 14 px.
  Window corners follow the Breeze decoration (rounded top). Fully rounded
  windows need an out-of-tree KWin effect that is not in Debian and breaks on
  KWin updates — **not in 1.0**.
- **Elevation:** one soft shadow level for popovers (`0 8 24 #00000066`);
  windows use KWin/Breeze shadows.
- **Translucency:** panels, launcher, overlay at ~88 % opacity with KWin blur.
  "Reduce transparency" (Settings → Appearance) makes all of them opaque;
  automatically suggested on software rendering (llvmpipe).
- **Motion:** 120 ms (hover, press) · 180 ms (popover, page) · 240 ms (overlay
  open). Easing `OutCubic` in, `InCubic` out. All durations derive from
  `Kirigami.Units` so Plasma's animation-speed slider (including "instant")
  applies everywhere. The assistant orb animates only while a request is active.

## 5. Desktop layout (default "Nexora" layout)

```
┌────────────────────────────────────────────────────────────────────────────┐
│ ◆ NEXORA  ▢▢▢▢ desktops        Thu 19 Nov  14:32        ◉Tor ●mic ⌁ 🔊 ᛒ 🔋 🔔 │  top bar 32 px
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│                         (wallpaper; widgets optional)                      │
│                                                                            │
│                  ╭──────────────────────────────────────╮                  │
│                  │  ◎  Ask NEXORA anything…        ⌘E   │  ← Ctrl+E overlay│
│                  ╰──────────────────────────────────────╯                  │
│                                                                            │
│           ╭──────────────────────────────────────────────────╮             │
│           │  ▣  ▣  ▣  ▣  ▣  ▣  ▣  │  ▣  ▣                    │  dock 56 px  │
│           ╰──────────────────────────────────────────────────╯  floating   │
└────────────────────────────────────────────────────────────────────────────┘
```

- **Top bar** (Plasma panel, 32 px, translucent): Kickoff launcher with NEXORA
  glyph; virtual-desktop pager as pills; centred clock (click → calendar);
  system tray: Tor status, privacy indicators (mic/camera/screen-share in use),
  network, volume, Bluetooth, battery, notifications, and the **Control Center**
  plasmoid (tiles: Wi-Fi, Bluetooth, VPN, Night Light, Do Not Disturb,
  Airplane mode; sliders: brightness, volume; buttons: lock, power).
- **Dock** (Plasma panel, floating, "fit content" width, dodge windows):
  icons-only task manager with pinned Dolphin, Firefox ESR, Tor Browser,
  Konsole, VS Code, Assistant, Settings; running indicators as a small cyan bar.
- **Alternative layout** "Classic" (single bottom taskbar) selectable in the
  Welcome app and in Global Theme settings. Both layouts are fully editable in
  Plasma's edit mode.
- **Desktop widgets** shipped but off by default: clock, system monitor rings,
  media controls, privacy status card.

## 6. Key surfaces

### Boot and login

| Surface | Design |
|---|---|
| GRUB | Dark `bg.base`, NEXORA wordmark top-centre, menu in `text.secondary`, selection bar `accent.cyan.tint`; timeout 3 s (hidden menu when single OS) |
| Plymouth | Logo centred, thin gradient ring spinner; LUKS prompt as a pill input with lock icon and keyboard-layout hint |
| SDDM | Blurred wallpaper, large clock, user avatar, password pill, buttons: session, keyboard layout, **accessibility** (on-screen keyboard, large text), power |
| Splash (KSplash) | Logo fade-in, 1 s max |

### Ctrl+E assistant overlay

- 720 px wide, centred in the upper third, `bg.surface2` at 92 %, 14 px radius.
- Left: orb (brand gradient ring, static when idle). Centre: input. Right:
  provider badge (`Local` in cyan / `Remote: <name>` in amber) and expand button.
- Answers stream into a card below the input; tool activity shows as a
  collapsed line ("Read disk usage · 1 permission used").
- **Command proposal card:** monospace command block; one-line explanation;
  risk badge (Read-only · Modifies your files · Administrator · Irreversible ·
  Blocked) with icon + colour; buttons `Copy` and `Run in terminal`. For
  Administrator/Irreversible the run button opens a confirmation step that asks
  the user to type `run`.
- `Esc` closes; `Ctrl+E` toggles; `Ctrl+Enter` sends with expanded mode.

### Assistant full window (three panes)

History sidebar (search, pinned, by date) · conversation · right drawer:
provider, model, effort (Claude) or temperature (local models), granted
permissions with revoke buttons, "What was sent" log for remote providers.
Status bar: `● Connected · Local · Qwen3-4B` or `● Remote · Claude Opus 5.5`.

### Privacy Dashboard

Card grid, each card = one fact with a status pill and a "Change" action:
Disk encryption · Secure Boot · Firewall · Automatic updates · AppArmor ·
Telemetry (list of components, all "Off") · Network identity (MAC mode, DHCP
hostname, DNS) · Tor (Tor Browser installed/running, system Tor service) ·
App isolation (Flatpak permissions) · Devices in use (mic/camera) · Clipboard ·
Assistant (provider, grants). No overall score.

### Downloads

List rows: file icon, name, source host, progress bar (cyan), rate, ETA,
pause/resume/cancel; filter chips (Active · Completed · Failed); right
inspector: URL, destination, size, SHA-256 field ("Verify"), "Open folder".

### Windows migration wizard

Stepper: Source → Profile → What to bring → Review → Copy → Done. Review step
shows size, free space, conflicts, skipped cloud-only files, and the sentence
"Your Windows drive is opened read-only and will not be changed."

## 6a. Implementation decisions (Phase 3)

| Topic | Decision | Why |
|---|---|---|
| Source of truth | `design/tokens.json` → `tools/nexora_design.py` generates the KDE colour schemes, `/etc/xdg/kdeglobals` and the Konsole scheme at package build time | One place to change a colour; every promised contrast pair (and all 128 KDE foreground/background combinations) is checked by `tests/design/test_design.py` |
| Plasma style | Plasma's default (Breeze) style, coloured by the Nexora Dark scheme — no custom SVG Plasma theme in 1.0 | Breeze follows the colour scheme, supports translucency/blur and floating panels; a custom SVG theme is dozens of files to keep in step with every Plasma release |
| Wordmark | Drawn as strokes, not text | Renders identically everywhere (GRUB, Plymouth, SDDM) without font dependencies |
| Generated images | PNGs and GRUB fonts are rendered from SVG at package build (`artwork/tools/render.py`) | No binary blobs in the repository; reproducible |
| Boot menu | One GRUB theme used by installed systems (`/boot/grub/themes/nexora`) and the live ISO | The live ISO takes its assets from the built `nexora-artwork` package |
| Plymouth default | Diversion of `/usr/share/plymouth/plymouthd.defaults` | Avoids editing Plymouth's conffile; `/etc/plymouth/plymouthd.conf` still overrides |
| Dock width | Plasma "fit content" when available, otherwise a fixed 40 % width | Plasma 6.3's scripting API is checked against `plasmashell` by the package test |

## 7. Iconography

- Base: **Papirus-Dark**; folders recoloured to a cyan→indigo tint set at build
  time (generated SVGs, GPL-3.0 like the source).
- NEXORA app icons (Assistant, Downloads, Migrate, Privacy, Settings overrides):
  original SVGs on a 48 px grid, rounded-square base `bg.surface2` with a glyph
  in brand gradient. Sources in `artwork/icons/`.
- Symbolic icons follow Breeze sizing (16/22/24 px).

## 8. Accessibility checklist (release gate)

- [ ] All text pairs ≥ 4.5:1 (automated check).
- [ ] Every action reachable by keyboard; visible 2 px cyan focus ring.
- [ ] Orca reads all NEXORA app controls (QML `Accessible.name` set).
- [ ] Animations honour Plasma animation speed, including "instant".
- [ ] High-contrast variant passes the same checks.
- [ ] SDDM offers on-screen keyboard and large text before login.
- [ ] Status never conveyed by colour alone.

## 9. Performance budget

| Item | Budget |
|---|---|
| Idle RAM after login (Plasma + NEXORA services idle) | ≤ 1.2 GB on 8 GB machine |
| NEXORA daemons idle | 0 MB (D-Bus activated, exit when idle), except tray widgets |
| Ctrl+E to visible overlay | ≤ 150 ms warm, ≤ 600 ms cold |
| Blur | Disabled automatically on software rendering |
| Wallpaper | Static image by default (no animated wallpapers) |
