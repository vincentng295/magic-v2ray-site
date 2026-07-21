# Magic V2Ray — User Guide

> System-wide transparent proxy for rooted Android, powered by **Xray-core**.
> Module version **v1.28.1** · Xray-core **v26.10.31** · Author: HuskyDG (@vincentng295) · License: GPL-3.0

---

## Table of contents

1. [Introduction](#1-introduction)
2. [Requirements & installation](#2-requirements--installation)
3. [Quick start](#3-quick-start)
4. [Header & global controls](#4-header--global-controls)
5. [Dashboard](#5-dashboard)
6. [Traffic Settings](#6-traffic-settings)
7. [Network Settings](#7-network-settings)
8. [Exclude Apps](#8-exclude-apps)
9. [Routing Settings](#9-routing-settings)
10. [Custom Hosts](#10-custom-hosts)
11. [Xray Log](#11-xray-log)
12. [Network Latency](#12-network-latency)
13. [About](#13-about)
14. [Apply behavior: soft reload vs. full restart](#14-apply-behavior-soft-reload-vs-full-restart)
15. [Files & paths](#15-files--paths)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. Introduction

Magic V2Ray runs Xray-core as a **root daemon** and marks traffic at the **Netfilter** (iptables / policy-routing) layer. Because it does not use Android's `VpnService`, it:

- proxies **every app**, including system UIDs;
- proxies **hotspot / tethered clients** (optional);
- leaves Android's single VPN slot **free** for other VPN apps;
- is controlled entirely from a **WebUI** inside your root manager (KernelSU / APatch / Magisk + KsuWebUIStandalone).

### How it works (short version)

1. `service.sh` creates a TUN interface called `xraytun0` and hands its file descriptor to Xray (the `tun-in` inbound).
2. iptables `mangle` rules mark outgoing packets and policy routing sends marked packets into `xraytun0`.
3. Xray's own outbound sockets carry **fwmark 255**. Packets with that mark skip the redirect rules, so Xray's own traffic leaves through the real network instead of looping back into itself.
4. Xray applies your routing rules and sends traffic to `proxy`, `direct`, or `block`.
5. A local SOCKS inbound `socks-test-in` on `127.17.1.3:808` is used by the built-in test tools (latency, IP check).

Supported protocols for nodes: **VMess, VLESS, Trojan, Shadowsocks, WireGuard, Hysteria2, MASQUE, SOCKS, HTTP**, plus **Proxy Chains** and **Policy Groups**.

---

## 2. Requirements & installation

| Item | Details |
|---|---|
| Root | KernelSU, APatch, or Magisk (Magisk 27+ shows the Action button) |
| CPU architecture | `arm64-v8a` or `x86_64` only |
| WebUI | Built into KernelSU / APatch. **Magisk users need KsuWebUIStandalone** to open the WebUI |
| Bundled | `xray`, `curl`, `xhuskydg_helper`, `geoip.dat`, `geosite.dat` |

**Install:** flash the module zip from your root manager and reboot. Open the module's WebUI to configure it.

**Uninstall:** removing the module also removes the data directory `/data/adb/magic_v2ray` (profiles, settings, credentials).

---

## 3. Quick start

1. Open the WebUI. Open the **Dashboard**.
2. Paste a subscription link (`https://…`) or a node link (`vless://`, `vmess://`, `trojan://`, …) into the import box and tap **Process Connection Link**.
3. Expand the new category and **tap a node** to select it. Confirm the dialog.
4. Tap **Start Engine**. The status badge turns to `RUNNING` and shows the active node.
5. Tap the status badge to see your current exit IP.

> You can also toggle the engine without opening the WebUI via the root manager's **Action** button on the module (`action.sh`): it stops a running engine, restarts a crashed one, or starts a stopped one.

---

## 4. Header & global controls

The header is visible on every tab.

| Control | What it does |
|---|---|
| **Language selector** | `Auto (system)`, English, Tiếng Việt, 简体中文, 繁體中文, Русский. Auto follows the system language. |
| **Menu (☰)** | Switches between tabs: Dashboard, Traffic Settings, Network Settings, Exclude Apps, Routing Settings, Custom Hosts, Xray Log, Network Latency, About. |
| **Status badge** | Shows `Status: RUNNING / STOPPED / CRASHED` and the active node name (or *Direct Routing* / *Custom config*). **Tap it** to query your current exit IP through the running engine (via icanhazip.com). Tap again to re-check, e.g. after switching nodes. |
| **Start Engine** | Writes the active config, re-applies the mark rule for the live network interface, then starts (or restarts) Xray. |
| **Stop Engine** | Stops Xray and tears down the proxy path. |

If Xray crashes, a toast tells you to check the **Xray Log** tab.

---

## 5. Dashboard

The Dashboard is where you manage **nodes** (outbound servers) and choose which one is active.

### 5.1 Professional Mode

A toggle at the top: **Professional Mode** — *"Write config.json by hand instead of picking a node."*

When **on**:

- Node pickers are hidden and a **Custom config.json** editor appears.
- Your JSON is written **verbatim** to `config.v2.json`.
- The **Traffic Settings** and **Routing Settings** tabs **no longer shape `config.json`**. Only the **Network Settings** tab (interface mode, tether, LAN bypass, bypass interfaces) and Exclude Apps still apply, because they work at the OS level.

Editor buttons:

| Button | Action |
|---|---|
| **Save & Apply** | Validates, saves, and applies the config. |
| **Validate** | Checks the JSON without saving. |
| **Format JSON** | Pretty-prints the JSON. |
| **Load Template** | Replaces the editor content with the default template (asks for confirmation). |

**Rules your custom config must follow** (shown in the "Read this before editing" box):

| Requirement | Why |
|---|---|
| **fwmark 255** — every outbound that dials a socket needs `streamSettings.sockopt.mark = 255` | Marked packets skip the tun redirect rules. Without it the tunnel loops its own traffic and nothing connects. |
| **`tun-in` inbound is mandatory** — `protocol: "tun"`, `settings.name: "xraytun0"` | `service.sh` creates this interface and passes the descriptor to Xray. Without the inbound, the engine runs but carries nothing. |
| **`socks-test-in` inbound** on `127.17.1.3:808` | Used by latency and IP-check tools. Removing it breaks those tools only (not traffic). |
| **`direct` and `block` outbounds** — a `freedom` outbound tagged `direct` (mark 255, `domainStrategy: UseIP`) and a `blackhole` tagged `block` | Routing rules reference these tags. |
| **DNS rule first** — send port 53 to `direct` before other rules | Xray's own internal lookups have no `inboundTag`; otherwise they fall to `outbounds[0]`, get proxied, and can deadlock a tunnel that is still coming up. |

The validator reports **errors** (empty config, invalid JSON, no outbounds, no tun inbound, wrong tun name, no outbound with mark 255) and **warnings** (unusual tun tag, missing `socks-test-in` or wrong address, missing `direct`/`block`, empty rules, no port-53 rule).

### 5.2 Import Configuration Profile

Paste one or more of the following, then tap **Process Connection Link**:

- a **subscription URL** (`https://…`) — fetched with the bundled `curl`;
- node links: `vless://`, `vmess://`, `trojan://`, `ss://`, `wireguard://`, `hysteria2://`, `masque://`, `socks://`, `http://`;
- Base64 blobs that decode to a list of links;
- single-line Xray outbound JSON objects.

A subscription becomes a **category** named after it. Manually imported nodes go into **Manual** (or the category you choose).

### 5.3 The "+" menu

| Item | Purpose |
|---|---|
| **Add category** | Create an empty category to organize nodes. Names cannot contain `:`. |
| **Add policy group** | Create an automatic node selector (see [5.7](#57-policy-groups)). |
| **Add Proxy Chain** | Chain two nodes (see [5.6](#56-proxy-chains)). |
| **Add VMess / VLESS / Trojan / Shadowsocks / WireGuard / Hysteria2 / MASQUE / SOCKS / HTTP Proxy** | Open the node editor to create a node by hand. |

### 5.4 Connection Categories & nodes

The list starts with the built-in **Direct Routing** pseudo-node, followed by your categories.

**Direct Routing** (`FREEDOM | Core routing only, no tunnel`): runs Xray as a pure router. The `proxy` outbound behaves like `direct`, so all routing rules still apply but nothing is tunnelled. This is also the active "node" when you have not selected anything.

**Selecting a node:** tap its row. A confirmation appears. If the node is **plain VLESS** (no TLS/Reality and no VLESS Encryption) you get an explicit plaintext warning instead and must choose *Connect anyway*. If the engine is running, the new node is applied with a soft reload. The selected node shows a 📌.

**Category menu (⋮ on the category header):**

| Item | Action |
|---|---|
| **Reload** *(subscriptions only)* | Re-fetches the subscription URL and updates the nodes. |
| **Edit** | Edit category name, subscription URL, *Deduplicate configurations when fetching*, *Skip TLS certificate check* (unsafe), and the **User-Agent** used to fetch (default `v2rayNG/2.2.3`). |
| **Deduplicate** | Removes duplicate nodes in the category. |
| **Check HTTP** | Measures HTTP latency of **every node** in the category. |
| **Check IP** | Shows the **exit IP** of every node in the category. |
| **Delete** | Deletes the category and its nodes. |

**Node menu (⋮ on a node):**

| Item | Action |
|---|---|
| **Edit config** | Opens the node editor (or chain / policy-group editor). |
| **Move to category** | Move to an existing category or create a new one. |
| **Copy payload URI** | Copies the node's share link. |
| **Copy full config** | Copies the full Xray JSON generated for this node. |
| **Check HTTP** | Latency test for this node (shows `NNNms` in green, or `?` in red on failure). |
| **Check IP** | Shows this node's exit IP. |
| **Delete** | Removes the node. |

**How the tests work:** each test launches a short-lived, **isolated Xray instance** with a private SOCKS inbound and probes through it with `curl` (HTTP latency uses `http://gstatic.com/generate_204`; IP check uses `https://icanhazip.com`; 3 s limit per probe). Tests run in parallel with a concurrency limit and do not disturb the running engine.

### 5.5 Node editor

Fields shown depend on the protocol.

**Common:** Remarks, Address, Port, ID/Password, Category.

| Protocol | Extra fields |
|---|---|
| **VMess** | Alter ID, Encryption (`auto`, `none`, `zero`, `aes-128-gcm`, `chacha20-poly1305`) |
| **VLESS** | Flow (`xtls-rprx-vision`, `xtls-rprx-vision-udp443`), **VLESS Encryption** (post-quantum string such as `mlkem768x25519plus.native.600s…`; empty/`none` = off) |
| **Shadowsocks** | Encryption method (including `2022-blake3-*`) |
| **WireGuard** | Secret key, server public key, pre-shared key, reserved bytes, local address, MTU, and **Generate built-in WARP account** (registers a free Cloudflare WARP account and fills the fields) |
| **Hysteria2** | Obfs password, port hopping range, hop interval, bandwidth up/down, TLS SNI |
| **MASQUE** (CONNECT-IP) | HTTP version (h3/h2), SNI, TLS fingerprint (h2 only), Host, Path, extra headers (JSON), remote DNS in tunnel, pinned cert, verify-by-name, optional **Cloudflare WARP** P-256 device/edge keys and tunnel address |
| **SOCKS / HTTP** | Optional username / password |

**Transport** (VMess / VLESS / Trojan / Shadowsocks): `tcp` (header `none`/`http` with host/path), `kcp` (header, host, seed), `ws` (path, host), `httpupgrade` (host, path), `xhttp` (mode `auto`/`packet-up`/`stream-up`/`stream-one`, host, path, extra JSON), `h2` (host, path), `grpc` (mode `gun`/`multi`, authority, serviceName).

**FinalMask** (JSON): a full `FinalMaskObject` with keys `tcp` / `udp` / `quicParams`. Network-agnostic; also available for Hysteria2. Leave empty to disable.

**Security:** `none`, `tls`, or `reality`.

- **TLS:** SNI, fingerprint (`chrome`, `firefox`, `safari`, `edge`, `qq`, `random`), ALPN, pinned cert SHA256, verify peer cert by name, ECH config list. *Allow Insecure* is shown struck-through: it has been **removed by Xray-core** and is never written to the config, even if an imported link says `allowInsecure=true`.
- **Reality:** public key, short ID, SpiderX, post-quantum verify (`pqv`).

### 5.6 Proxy chains

*Client → Hop 1 → Hop 2 → Internet.* Hop 2's traffic is tunnelled **inside** Hop 1.

- **Hop 1 (outer / entry)** — the node your device connects to.
- **Hop 2 (inner / exit)** — the node that reaches the Internet; its payload is encapsulated in Hop 1's tunnel.

Rules: both hops are required, they must differ, and a chain cannot be used as a hop of another chain. Chains show as `⛓ CHAIN: PROTO1 → PROTO2`.

### 5.7 Policy groups

A policy group automatically picks a node from a set. Fields:

| Field | Meaning |
|---|---|
| **Name** | Must be unique. |
| **Save in category** | Where the group appears in the list. |
| **Policy group type** | `random`, `roundRobin`, `leastPing` (lowest ping), `leastLoad` (most stable). |
| **From category** | Source of candidate nodes (one category, or all). |
| **Remark regex filter** | Only nodes whose remark matches, e.g. `US|Singapore`. |

`leastPing` / `leastLoad` probe each node in the background (`generate_204`). If every node looks dead, traffic **falls back to the main node**. Policy groups can be selected as the active node and used as the `outboundTag` of routing rules (they become Xray `routing.balancers`).

---

## 6. Traffic Settings

**Advanced Engine Settings** control the Xray configuration that is generated for your node. Saving uses a **soft reload** (swaps the Xray process; iptables rules are untouched).

Buttons at the bottom:

- **Save & Apply Configurations**
- **Reset to Defaults** — restores *only* the fields owned by this tab (after confirmation). Network, Routing, Hosts, Exclude Apps, Professional Mode and language are **not** affected.

### 6.1 Routing & Core Protocol

| Setting | Default | Description |
|---|---|---|
| **Level Logging** | `info` | Xray log level: `none`, `debug`, `info`, `warning`, `error`. Output is shown in the **Xray Log** tab. |
| **Enable Sniffing** | on | Sniffs domains from the packet stream (HTTP, TLS, QUIC) so domain rules can match even when apps connect by IP. |
| **Enable RouteOnly** | off | Sniffed domains are used **only for routing decisions**; the connection still goes to the original destination IP. |
| **DNS query strategy** | `UseIPv4` | Which record types Xray's DNS module queries. `UseIP` = A + AAAA (Xray default). `UseIPv4` = A only. `UseIPv6` = AAAA only. `UseSystem` = checks whether the device has IPv4/IPv6 default gateways and limits queries accordingly. |
| **Pinned Peer Cert SHA256** | empty | Pins the SHA256 hash of the server certificate. |

### 6.2 Multiplexing (MUX)

Toggle (default **off**). When on, one connection carries multiple streams.

- **Max Concurrency Connections** — default `8` (1–1024).

Not applied to MASQUE nodes or in Direct Routing mode.

### 6.3 Firewall Fragment (Anti-DPI)

Toggle (default **off**). Splits the TLS ClientHello into small pieces to evade deep-packet inspection.

| Setting | Default |
|---|---|
| **Fragment Packets Target** | `tlshello` |
| **Fragment Length (min-max)** | `50-100` |
| **Fragment Interval (min-max)** | `10-20` |

### 6.4 Network Interfaces — MTU

**Maximum Transmission Unit (MTU)** — default `1350`. Used as the fallback MTU for tunnel-style outbounds (such as WireGuard) when a node does not define its own.

### 6.5 DNS Configuration

The master toggle (**Enable local DNS**) switches from the built-in simple DNS setup to your own structured DNS setup.

- **Off:** Xray uses the built-in default resolvers (DoH over Cloudflare: `https://1.1.1.1/dns-query` plus two Cloudflare Gateway DoH endpoints).
- **On:** the following options appear.

| Setting | Default | Description |
|---|---|---|
| **Resolve DNS via Proxy** | on | Wraps and routes all DNS resolver traffic through the proxy server. |
| **Hijack DNS** | off | Routes port-53 queries from `tun-in` / `socks-test-in` to Xray's **DNS module** (`dns-out`) instead of forwarding raw packets. Upstream queries then follow *Resolve DNS via Proxy*. |
| **Enable Fake DNS** | off | Maps domains to virtual IPs instantly using `198.18.0.0/15` (and `2001:2::/48` for IPv6). **Requires Hijack DNS**; the switch is disabled until it is on. Sniffing is extended with `fakedns` so the real domain is recovered for routing. |
| **DNS servers → Configure** | — | Opens the DNS servers editor (below). |

**DNS servers editor** — three lists, each editable per item or as plain text (one entry per line):

| List | Default | Role |
|---|---|---|
| **Direct DNS** | `quic+local://223.5.5.5` | Resolvers queried **without** the proxy. |
| **Domains using direct DNS** | `geosite:cn, geosite:private` | Domain rules answered by Direct DNS (e.g. `geosite:cn`, `domain:example.vn`). If empty, Direct DNS is not used. When the list contains only `geosite:cn` / `geosite:private`, answers are validated against `geoip:cn` / `geoip:private`. |
| **Proxy DNS** (foreign DNS) | the three built-in DoH entries | Resolvers for **all other domains**; queried through the proxy when *Resolve DNS via Proxy* is on. Several servers can be listed. If left empty, `1.1.1.1` is used so Xray never has an empty list. |

Changes in the editor are staged — press **Save & Apply Configurations** to apply them.

### 6.6 DNS Engine Options

These map to Xray's `dns.*` options.

| Setting | Default | Description |
|---|---|---|
| **Disable DNS cache** | off | Do not cache answers in Xray (system DNS is unaffected). |
| **Serve stale cache** | off | Return expired cache instantly and refresh it in the background. |
| **Stale cache lifetime (s)** | `0` | How long expired entries may still be served. `0` = never expires. |
| **Disable fallback** | off | Never query servers that did not match the domain list. |
| **Disable fallback if matched** | off | Skip fallback servers when a server's domain list already matched. |
| **Parallel query** | **on** | Query servers simultaneously to speed up resolving (Xray's own default is off). |
| **Use system hosts** | off | Append the device hosts file to the built-in DNS hosts. |

---

## 7. Network Settings

These switches change **how traffic is routed at the OS level** (iptables / policy routing). Saving here always performs a **full engine restart**.

### 7.1 Interfaces, tethering & IPv6

| Setting | Default | Description |
|---|---|---|
| **Apply Proxy On** | All Networks | `All Networks`, `WiFi / Ethernet Only` (mobile data bypasses the proxy), or `Mobile Data Only` (everything except mobile data bypasses). |
| **Upstream network interface** | `Auto` | The interface Xray itself connects out through. `Auto` follows the system's active network and re-pins on every network handover. Pick a specific interface (e.g. `wlan0`, `rmnet_data0`) to pin; if it is down, the module falls back to Auto. The list is refreshed each time you open the tab. |
| **Enable IPv6** | off | Allow IPv6 connectivity through the proxy. |
| ↳ **Advertise IPv6 to apps** | off | Adds a local-only IPv6 (ULA) address to the active interface so apps *believe* IPv6 works and use it. Traffic is still carried by Xray. Needed when the real network has no IPv6. Only available when *Enable IPv6* is on. |
| **Allow Tether from Proxy** | on | Routes tethered / hotspot clients through the proxy. Turn off to let them use the direct connection. |
| **Bypass VpnService** | on | Let `VpnService` handle packets first (`tun0`, `tun+`). Only the VPN app's own outgoing traffic is then handled by Xray. Your own `xraytun0` is not matched. |
| **Bypass Network Interface** | off | Traffic headed to the listed interfaces skips Xray entirely. When on, enter a comma-separated list, e.g. `tailscale0,wt0`. |

### 7.2 Bypass LAN

Choose which destination ranges **skip Xray and connect directly**.

- **Ticked** = bypass (direct).
- **Unticked** = send that range through Xray.
- **Bypass all / Bypass none** shortcuts are provided; a summary shows `N of M ranges bypassed`.
- **Loopback (`127.0.0.0/8`, `::1/128`) is always bypassed** and cannot be changed.

Default: **every range is bypassed.** Ranges listed:

| IPv4 | Description |
|---|---|
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Private (RFC 1918) |
| `100.64.0.0/10` | Carrier-grade NAT |
| `169.254.0.0/16` | Link-local |
| `192.0.0.0/24` | IETF protocol assignments |
| `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` | Documentation (TEST-NET-1/2/3) |
| `192.88.99.0/24` | 6to4 relay anycast (deprecated) |
| `224.0.0.0/4` | Multicast |
| `240.0.0.0/4` | Reserved (Class E) |
| `255.255.255.255/32` | Limited broadcast |

| IPv6 *(applies when IPv6 is enabled)* | Description |
|---|---|
| `::ffff:0:0/96` | IPv4-mapped |
| `64:ff9b::/96` | NAT64 well-known prefix |
| `100::/64` | Discard-only |
| `2001::/32` | Teredo |
| `2001:10::/28`, `2001:20::/28` | ORCHID / ORCHIDv2 |
| `2001:db8::/32` | Documentation |
| `2002::/16` | 6to4 |
| `fc00::/7` | Unique local (ULA) |
| `fe80::/10` | Link-local |
| `ff00::/8` | Multicast |

Press **Save & Restart Engine** to apply.

---

## 8. Exclude Apps

Apps you switch **on** here **bypass the proxy** and use the direct connection. Enforcement happens at the OS level by UID, so it applies regardless of routing rules. Applying a change **fully restarts the engine**.

### 8.1 Settings

| Setting | Default | Description |
|---|---|---|
| **Enable Exclude apps** | off | Master switch. Turn off to keep your list but proxy every app. |
| **Apply changes immediately** | off | Automatically restarts the engine about a second after each change. Turn off to review your edits and apply them yourself. |
| **Save & Restart Engine** | — | Appears with an *Unsaved changes* hint when edits are pending and auto-apply is off. |

Edits are held as a draft and only committed on apply, so an unrelated save elsewhere never persists half-edited exclusions.

### 8.2 App list

- **Search** by app name or package name.
- **Filters:** `User`, `System`, `All`, `Excluded`.
- **Select shown / Clear shown** — bulk-toggle the apps currently visible after filtering.
- A counter shows `N excluded`.
- The list is paged as you scroll. App names/icons come from the root manager's package API (`window.ksu`); the list is stored one package name per line in `/data/adb/magic_v2ray/excludelist.txt`.

### 8.3 Backend behavior

- Excluded UIDs are placed in a dedicated iptables chain `XRAY_EXCLUDE_APP`, which the mark chain jumps to **before** anything is marked.
- On a full start the chain is **flushed and rebuilt** from the list.
- A background `monitor_apps` (only while the feature is on) watches `/data/data` with `inotifyd`, so **newly installed or uninstalled excluded apps** are handled between restarts.
- Shared-UID packages are handled correctly: a UID's rule is removed only when the last excluded package using it is gone.
- Only **user 0** (`/data/data`) is covered.

---

## 9. Routing Settings

Defines **Xray routing rules**: which traffic goes to `proxy`, `direct`, or `block`. Rules are evaluated **top to bottom; the first match wins**.

> Not used while **Professional Mode** is on.

### 9.1 Domain strategy

| Value | Behavior |
|---|---|
| **AsIs** *(default)* | Never resolves domains for routing. Rules match on the domain (or the IP as requested). No extra DNS lookups. |
| **IPIfNonMatch** | Tries all rules with the domain first; if none match, resolves the domain with the built-in DNS and tries the IP rules again. |
| **IPOnDemand** | Resolves the domain as soon as a rule with an IP condition is reached. Matches IP rules earliest but adds a DNS lookup to more connections. |

### 9.2 Managing rules

- **＋** adds a rule. Tap a rule to edit; use the arrows to **move up/down**; use the toggle to **enable/disable**; delete removes it (with confirmation).
- A rule needs **at least one** of `domain`, `ip`, `port`, or `protocol`.
- **⋮ menu:**
  - **Import presets from JSON** — paste a JSON array or **fetch it from a URL**. Unlocked rules are replaced; **locked rules are kept**.
  - **Export presets JSON to clipboard**.
  - **Reset to default rules** — restores the built-in list (custom and locked rules are removed; domain strategy is unaffected).

### 9.3 Rule fields

| Field | Format | Example |
|---|---|---|
| **Remarks** | Free text label | `Proxy Google` |
| **Locked** | Keep this rule when importing presets | — |
| **domain** | Comma-separated Xray domain matchers | `geosite:google, domain:example.com` |
| **ip** | Comma-separated IPs / CIDRs / GeoIP | `geoip:private, 1.1.1.1` |
| **port** | Port or range | `443` or `1000-2000` |
| **protocol** | Sniffed protocols | `http,tls,bittorrent` |
| **network** | `tcp,udp` (both), `tcp`, or `udp` | `udp` |
| **outboundTag** | `proxy`, `direct`, `block`, **a saved node (by remark)**, or a **policy group** | `direct` |

If a rule points to a node or policy group that was deleted, the dropdown flags it as missing.

### 9.4 Built-in default rules

| # | Remark | Match | Outbound | Default |
|---|---|---|---|---|
| 1 | Block UDP 443 | port `443`, UDP | `block` | on |
| 2 | Proxy Google | `geosite:google` | `proxy` | on |
| 3 | Gateway Magic domain | `domain:gateway-magic.huskydg.dpdns.org` | `direct` | on |
| 4 | Gateway Magic IP | `198.18.0.1`, `fdfe:dcba:9876::1`, `2001:db8:7872:6179::1` | `direct` | on |
| 5 | Bypass LAN IP | `geoip:private` | `direct` | on |
| 6 | Bypass LAN domain | `geosite:private` | `direct` | on |
| 7 | Bypass China public DNS IPs | list of Chinese public resolver IPs | `direct` | off |
| 8 | Bypass China public DNS domains | `alidns.com`, `doh.pub`, `dot.pub`, `360.cn`, `onedns.net` | `direct` | off |
| 9 | Bypass China IP | `geoip:cn` | `direct` | off |
| 10 | Bypass China domain | `geosite:cn` | `direct` | off |

(Rule remarks 1, 2 and 5–10 are shipped in Chinese in the module; they are translated above.)

Rules 1–6 are active by default, so the **QUIC/UDP-443 block** forces apps to fall back to TCP, which proxies more reliably.

---

## 10. Custom Hosts

Static DNS overrides merged into Xray's `dns.hosts` **on top of the built-in defaults**. The data is a genuine **`/etc/hosts`-format file** (`IP  hostname [# comment]`), stored at `/data/adb/magic_v2ray/hosts`.

### 10.1 Using the tab

| Control | Action |
|---|---|
| **＋** | Add a host: a **domain** and one or more **IPs** (comma-separated). |
| **Search** | Filter by domain or IP. |
| Tap an entry | Edit it. |
| Delete | Remove the entry (with confirmation). |
| **Reload hosts file (↻)** | Re-reads the file from disk — useful if you edited it externally — and regenerates `config.json`. In Professional Mode the file is reloaded but `config.json` is left alone. |

A domain can have **several IPs**. The domain key accepts Xray prefixes such as `domain:`, `full:`, `keyword:`, `regexp:`, `geosite:`, `ext:`. A value can also be an **alias domain** instead of an IP (e.g. `domain:googleapis.cn → googleapis.com`).

### 10.2 Built-in defaults

The module ships pinned entries so DoH/DoT hostnames never need DNS to be resolved (avoiding resolver loops), including: `localhost`, the Magic gateway domain, `googleapis.cn → googleapis.com`, Alibaba DNS, Cisco/Umbrella, Cloudflare (`one.one.one.one`, `cloudflare-dns.com`, Cloudflare Gateway DoH hosts), `doh.pub` / `dot.pub`, Google DNS, Quad9, DNS.SB, and Yandex.

Your custom entries are merged on top of these. In the legacy text format, prefixing a line with `!` **drops a built-in entry**.

---

## 11. Xray Log

A live viewer for Xray's output (read from the module's log pipe through `xhuskydg_helper`).

### 11.1 Toolbar

| Control | Action |
|---|---|
| **Status dot / line count** | Green dot when live; shows how many lines are loaded. |
| **Auto-scroll (⇣)** | Keeps the view pinned to the newest line. |
| **Font size (− / +)** | Default 12. |
| **Word wrap** | Toggle wrapping of long lines. |
| **Copy** | Copy the log to the clipboard. |
| **Refresh** | Reload now. |
| **Clear view** | Clears the on-screen buffer (does not change the engine). |

### 11.2 Filters & refresh

- **Level chips:** `All`, `Debug`, `Info`, `Warn`, `Error`.
- **Auto-refresh:** on/off, interval `2s`, `5s` (default), `10s`, `30s`. Auto-refresh runs only while this tab is open.
- **Tail lines:** `100`, `200` (default), `500`, `1000`.

> The amount of detail depends on **Level Logging** in Traffic Settings. With `none` the log stays empty.

---

## 12. Network Latency

A live **network health monitor** with a chart, plus a **Mobile IP Hunter**.

### 12.1 How monitoring works

- The monitor runs **only while this tab is visible**: it starts when you open the tab and stops when you leave (or close/background the WebUI).
- Every ~2 seconds the backend measures the time-to-first-byte of `https://gstatic.com/generate_204` through the `socks-test-in` inbound.
- The UI sends a heartbeat; if it stops for ~15 seconds the backend probe **ends by itself**, so closing the WebUI never leaves a probe running.
- Because the probe goes through Xray, it reflects the **latency of your current proxy path**, and requires the engine to be running.

### 12.2 What you see

| Element | Description |
|---|---|
| **Current value** | Latest latency in ms, with a live status dot. |
| **Interface / IPv4 / IPv6** | Active network interface and its global addresses (shows *Not connected* if none). |
| **Avg / Min / Max / Loss** | Statistics over the last **60 samples**. |
| **Chart** | Latency graph. Green = response time (ms); red = timeout / no connection. |

### 12.3 Mobile IP Hunter

Tries to land on a mobile-data IP address within chosen prefixes by **restarting the mobile data stack** until the assigned IPv4 matches.

| Setting | Description |
|---|---|
| **Enable IP hunter** | On/off. Turning it on saves the prefixes and resets the mobile network. |
| **Prefix list** | IPv4 prefixes separated by `;`, e.g. `10.120;10.121;10.122`. Invalid entries are dropped. |

Behavior:

- Applies **only to mobile-data interfaces** (`rmnet*`, `ccmni*`, `pdp*`, `wwan*`).
- If the current IP does not start with any prefix, the telephony process (`com.android.phone`) is restarted — **not** airplane mode, so Wi-Fi and Bluetooth stay up.
- Retries use **exponential backoff** (3 s, 6 s, 12 s, …) and stop after **10 attempts**, at which point the hunter **disables itself** to avoid endless reconnects and battery drain.
- On a match it logs the matched IP and resets its counters.

---

## 13. About

Shows the module tagline, **version** (read from `module.prop`), author, links (source code on GitHub, latest release, issue tracker, website, Telegram group when configured), credits (**Xray-core**, **openxtun**, **curl-android**), and the license (**GNU GPL v3.0**). Links open outside the WebUI.

---

## 14. Apply behavior: soft reload vs. full restart

| Where you change something | What happens |
|---|---|
| Select a different node; edit a node; Traffic Settings; Routing rules; Custom Hosts | `config.json` is regenerated and Xray is **soft-reloaded** (the process is swapped; iptables / policy routing stay as they are). Only if the engine is running. |
| **Network Settings** save | **Full restart** (iptables and routing rules are rebuilt). |
| **Exclude Apps** apply | **Full restart.** |
| **Start Engine** button | Re-applies the mark rule for the live interface, then starts/restarts. |
| Professional Mode **Save & Apply** | Writes your JSON verbatim, then reloads. |

---

## 15. Files & paths

| Path | Purpose |
|---|---|
| `/data/adb/modules/magic_v2ray/` | Module directory (`service.sh`, `proxy_control.sh`, `action.sh`, `bin/`, `webroot/`). |
| `/data/adb/magic_v2ray/profiles.base64` | Saved categories and nodes. |
| `/data/adb/magic_v2ray/settings.base64` | All WebUI settings. |
| `/data/adb/magic_v2ray/active_config.txt` | The selected node. |
| `/data/adb/magic_v2ray/config.v2.json` | The generated (or custom) Xray config. |
| `/data/adb/magic_v2ray/hosts` | Custom Hosts file. |
| `/data/adb/magic_v2ray/excludelist.txt` | Excluded app package names. |
| `/data/adb/magic_v2ray/bypassIface.txt` | Bypass-interface list (file exists = feature on). |
| `/data/adb/magic_v2ray/ip_hunt.list` | IP Hunter prefixes (file exists = hunter on). |
| `/data/adb/magic_v2ray/proxy_control.log` | Control-script diagnostics. |
| `/dev/sysctl_stubs/…` | Runtime state: log pipe, latency results, control FIFO. Cleared on reboot. |

The data directory is root-owned (`0700`) because it holds server addresses, UUIDs, passwords and WireGuard keys. **Back up carefully and never share these files.**

---

## 16. Troubleshooting

| Symptom | Things to check |
|---|---|
| Engine starts but **nothing connects** | In Professional Mode, ensure a `tun-in` inbound named `xraytun0` and **mark 255** on every dialing outbound. Otherwise check the node with **Check HTTP / Check IP**. |
| Status shows **CRASHED** | Open **Xray Log** (set Level Logging to `info`/`debug`). Usually an invalid config or unsupported node setting. |
| **No node selected** | The built-in Direct Routing is used: the engine runs as a router only, nothing is tunnelled. |
| **DNS leaks or slow DNS** | Enable *local DNS* + *Hijack DNS* + *Resolve DNS via Proxy*; keep *Parallel query* on; consider *Serve stale cache*. |
| **Fake DNS switch is greyed out** | Enable **Hijack DNS** (and local DNS) first. |
| An app **must not** be proxied | Add it in **Exclude Apps** (turn the master switch on and apply). |
| **Another VPN app** conflicts | Keep **Bypass VpnService** on so the VPN app handles its packets first. |
| **Hotspot clients** bypass the proxy | Turn on **Allow Tether from Proxy** and restart. |
| **LAN devices / local servers** unreachable through proxy | Keep the relevant ranges ticked in **Bypass LAN**. |
| IPv6 apps fail on an IPv4-only network | Turn on **Enable IPv6** and **Advertise IPv6 to apps**. |
| **Latency tab empty** | The engine must be running; open the tab (the monitor only runs while it is visible). |
| Subscription **fetch fails** | Check connectivity (the bundled `curl` is used), try a different **User-Agent** in the category's Edit dialog. Use *Skip TLS certificate check* only for hosts with a self-signed certificate — it is unsafe. |
| Plain-VLESS warning | The node has no TLS/Reality/encryption: traffic is readable by anyone on the path. Only continue on trusted networks. |

---

*This guide describes Magic V2Ray v1.28.1 with Xray-core v26.10.31. Defaults are taken from the module's own source and may change in future versions.*