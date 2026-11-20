# NEXORA OS — Network Privacy: Kill Switch, Identity Change, Proxy Chains, nftables

Status: design (added 2026-11-20 at the owner's request). Implementation:
Phase 4, part 4b. Threat-model references: T-NET-1…6, T-TOR-1…7.

## 1. nftables

| Layer | Role |
|---|---|
| **nftables** (kernel packet filter, `nft`) | The only packet filter in NEXORA. Installed explicitly (`nftables` package). |
| **firewalld** (nftables backend) | Base inbound firewall and per-network zones, managed from the Privacy Dashboard and System Settings. Keeps its own nftables tables. |
| **NEXORA kill switch** | A separate nftables table, `inet nexora_killswitch`, owned by NEXORA. It drops outbound traffic that would leak outside the chosen tunnel. |

In nftables a packet passes through every base chain registered on a hook,
and a drop in any of them is final. So the kill-switch table can only make
the firewall stricter, never weaker, and it works the same whether firewalld
is running or not. Debian's own `nftables.service` (`/etc/nftables.conf`)
stays disabled so two managers never fight over the same rules.

## 2. Kill switch

Modes (one at a time):

| Mode | Outbound traffic allowed | Everything else |
|---|---|---|
| **Off** (default) | All | — |
| **VPN only** | Loopback; VPN tunnel interfaces (`wg*`, `tun*`); the configured VPN server endpoints on the physical interface; DHCP/DHCPv6 and the IPv6 neighbour discovery needed to get online; optionally the local network | Rejected with "administratively prohibited", so applications fail fast instead of hanging |
| **Tor only** | Loopback; traffic from the system Tor daemon (`debian-tor` user); DHCP/IPv6 neighbour discovery; optionally the local network | Rejected |

Design points:

- **Applied at boot, before the network** (`nexora-killswitch.service`,
  `Before=network-pre.target`) when enabled, so nothing leaks during start-up
  or while the VPN is still connecting.
- **Configuration** in `/etc/nexora/killswitch.conf`, changed only through
  `nexora-privd` (`org.nexora.System1.SetKillSwitch`, polkit `auth_admin_keep`).
  The VPN endpoints are read from the active NetworkManager VPN/WireGuard
  connection, never from free text.
- **Tor-only mode and Tor Browser:** Tor Browser normally runs its own Tor
  process as the user, which this mode would block. In Tor-only mode the
  launcher starts Tor Browser against the system Tor (`TOR_SKIP_LAUNCH=1`,
  SOCKS port 9050), as Tails does.
- **DNS:** in both modes, plain DNS to the local router is blocked unless it
  goes through the tunnel. That is the point: no DNS leaks. Name resolution
  works through the VPN's DNS or through Tor (`DNSPort` / SOCKS with remote DNS).
- **Status:** shown in the tray and Privacy Dashboard ("Kill switch: VPN only —
  blocking", or "… — VPN connected").

Honest limits (shown in the UI):

- The kill switch prevents traffic from leaving outside the tunnel. It does
  not make you anonymous: the VPN provider sees your traffic, and Tor's own
  limits (T-TOR-1…7) still apply.
- Traffic the kernel itself generates before the rules load (very early boot)
  and other machines on your network are outside its control.
- With "allow local network" on, devices on your LAN (printers, NAS) stay
  reachable, and so does anyone else on that network.

## 3. Identity change ("IP changer")

One "Change identity" action, three independent parts the user can choose:

| Part | What actually changes | What does not |
|---|---|---|
| **Local network** | New random MAC address for the current Wi-Fi/Ethernet connection, then reconnect → new DHCP lease and usually a new local IP | Your public IP (the router's) normally stays the same |
| **Tor** | Asks the system Tor for new circuits (`SIGNAL NEWNYM`) → new connections leave from different exit relays/IPs | Connections already open keep their circuit; Tor rate-limits this to about once per 10 s |
| **VPN** | Reconnects to the next VPN profile you configured (round-robin) → new public IP from the VPN's pool | Needs at least two VPN profiles or a provider that rotates addresses |

Implemented in `nexora-privd` (`ChangeIdentity(parts as)`): NetworkManager
over its D-Bus API, Tor through its control port (cookie readable only by
root and `debian-tor`). Tor Browser's own "New identity" button is unchanged
and also clears browser state; NEXORA's action does not touch browser data.

Honest limits: websites still recognise you by cookies, logins and browser
fingerprint; changing IP does not change those. A new MAC is visible to the
network as a "new device", which some networks (hotel captive portals) treat
as a new login.

## 4. Proxy chains

- Package: **proxychains-ng** (`proxychains4`) from Debian.
- NEXORA configuration `/etc/nexora/proxychains.conf` (the package's own
  `/etc/proxychains4.conf` is left untouched): `strict_chain`, `proxy_dns`
  (DNS goes through the chain, no local DNS leak), default chain = the system
  Tor SOCKS port `127.0.0.1:9050`. Users add their own SOCKS5/HTTP proxies in
  `~/.config/nexora/proxychains.conf` (overrides the system file).
- `nexora-proxy <program> [args]` wrapper, "Run through proxy chain" in the
  application launcher's context menu and in Dolphin for scripts, and a
  setting page to edit the chain and test it (shows the exit IP seen through
  the chain).

Honest limits (shown in the UI and docs):

- Works by preloading a library into the program: it covers dynamically
  linked programs that use the C library's network calls. It does **not**
  cover statically linked programs (many Go/Rust tools), Flatpak/Snap apps,
  setuid programs, or UDP traffic. Such traffic either goes direct or fails;
  combine with the kill switch (Tor-only mode) to make it fail instead of leak.
- Every proxy in the chain can see the traffic that is not end-to-end
  encrypted; an HTTP proxy sees plain HTTP. Chaining proxies does not make
  untrusted proxies trustworthy.

## 5. Privacy claims (additions to threat-model §7)

Allowed: "The kill switch blocks traffic outside your VPN/Tor while it is on."
"Change identity gives you a new local MAC/IP, new Tor circuits and/or a new
VPN server." "The proxy chain sends this program's TCP traffic and DNS through
the proxies you configured."

Forbidden: "untraceable", "hides your IP everywhere", "anonymous mode",
"kill switch makes you anonymous".

## 6. Tests (part of 4b's acceptance)

- nftables ruleset generated for each mode, checked with `nft -c` (syntax)
  and loaded in a network namespace in the builder container; probes confirm
  allowed paths pass and others are rejected (no real network needed).
- Kill switch survives reboot (boot test in VM) and blocks traffic when the
  VPN interface goes down (VM test with a WireGuard pair in two namespaces).
- `nexora-proxy curl` through a local test SOCKS server returns the proxy's
  address; DNS is resolved by the proxy (no UDP/53 leaving the namespace).
- Identity change: MAC differs before/after in a VM; Tor NEWNYM accepted by a
  test tor instance.
