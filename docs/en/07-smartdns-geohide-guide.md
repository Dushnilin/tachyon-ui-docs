# 7. Self-Hosted SmartDNS / GeoHide, SNI Proxy, DoH/DoT & Setup Wizard

[Back to Index](../../README.md) | [Chapter 6 — Diagnostics & API](06-diagnostics-privacy-and-api.md) | [Русский](../ru/08-smartdns-geohide-guide.md)

Starting with **`tachyon-ui v0.4.5`** (`ui_version 20261126-v26`), the panel includes a built-in **SmartDNS (GeoHide-style selective DNS) + transparent SNI proxy** written in pure Rust (zero external daemons such as `dnsmasq`, `nginx`, or `sniproxy`, and zero new Cargo dependencies). In under a minute you can deploy your own **UDP/TCP `:53`**, **DNS-over-HTTPS (DoH RFC 8484)**, and **DNS-over-TLS (DoT `:853`)** server for devices that cannot run a VPN app: **PS5, Xbox, Nintendo Switch, SmartTVs (LG, Samsung, Apple TV)**, desktops, phones, and routers.

---

## 1. How It Works: SmartDNS + Transparent SNI Proxy

Unlike a full-tunnel VPN, **SmartDNS** requires no client software and leaves regular traffic direct:

```text
┌─────────────────────────────┐
│ Console / SmartTV / PC /    │   1. DNS query (:53 / DoH / DoT)
│ Phone / Router              ├───────────────────────────────────────────┐
│ DNS = Your VPS IP           │                                           ▼
└──────────────┬──────────────┘                              ┌──────────────────────────┐
               │                                             │   tachyon-ui SmartDNS    │
               │                                             │  Client IP allowlist     │
               │                                             └───┬──────────────────┬───┘
               │                                                 │                  │
               │  2a. Non-matched domain (e.g. local/bank)       │ DIRECT           │ PROXY (matched rule)
               │      Returns real upstream IP ◄─────────────────┤                  │ Returns VPS IPv4
               │      → traffic goes DIRECTLY to origin          │                  ▼ (proxy_ipv4)
               │                                                 │         + NOERROR NODATA for AAAA/HTTPS
               │  2b. Target service (PSN, YouTube, ChatGPT)     │
               │      Connects to VPS (:443 TLS / :80 HTTP) ◄────┴───────────────────────────────────────┘
               ▼
┌─────────────────────────────┐   3. Peeks ClientHello SNI (:443) or HTTP Host (:80) — NO TLS MITM!
│   Built-in SNI Proxy        │      Checks client IP authorization AND rule match before dialing
│   tachyon-ui (:443 / :80)   ├───────► outbound_mode = "direct" → direct VPS egress
└─────────────────────────────┘   or    outbound_mode = "socks5" → via socks5 (tachyon-core / relay)
```

### Key Design Properties
- **End-to-End TLS Preserved:** The SNI proxy only inspects the cleartext TLS `ClientHello` SNI extension on `:443` (or the HTTP `Host:` header on `:80`) and splices TCP streams verbatim. Clients validate the origin's genuine X.509 certificate.
- **Anti-Bypass for IPv6 & QUIC (`AAAA` / `HTTPS` / `SVCB`):** When `proxy_ipv6` is empty, proxied domains answer `NOERROR NODATA` for `AAAA`, `HTTPS`, and `SVCB` queries so consoles and browsers stay on the IPv4 SNI proxy instead of leaking over IPv6 or HTTP/3 Alt-Svc.
- **Zero DNS Loop:** The SNI proxy resolves real origin IPs directly over UDP against `upstream_dns` (default `1.1.1.1:53`, `8.8.8.8:53`), never via local `:53`, and rejects loopback or self `proxy_ipv4` destinations.
- **Flexible Egress (`direct` or `socks5`):** Traffic can exit directly from the VPS (`direct`) or be chained into a local SOCKS5 inbound such as `tachyon-core` (`127.0.0.1:10808`).

---

## 2. Server Requirements, Ports & TLS Certificates

### Network Ports
| Protocol | Default Port | Purpose | Firewall Requirement |
| :--- | :--- | :--- | :--- |
| **DNS (UDP + TCP)** | `53` | Plain DNS for PS5, Xbox, Switch, SmartTV, Windows, routers | Allow `53/udp` & `53/tcp` |
| **SNI Proxy HTTPS** | `443` | Transparent TLS SNI forwarding | Allow `443/tcp` |
| **SNI Proxy HTTP** | `80` | Transparent HTTP `Host` forwarding | Allow `80/tcp` |
| **DNS-over-TLS (DoT)** | `853` | Encrypted DNS for Android Private DNS and routers | Allow `853/tcp` |
| **DNS-over-HTTPS (DoH)** | Panel port (e.g. `2053`) | RFC 8484 endpoints (`/dns-query` & `/dns-query/<token>`) | Panel port already open |

> [!NOTE]
> When **Automated Firewall Management** (`auto_firewall_enabled`) is enabled in **Settings**, `tachyon-ui` reconciles `UFW` / `firewalld` / `iptables` rules automatically for active SmartDNS ports (`dns_port`, `sni_https_port`, `sni_http_port`, `dot_port`). If your VPS provider enforces an external cloud security group / firewall (AWS, Oracle Cloud, Hetzner Cloud Firewall, etc.), allow those ports there as well.

### Automatic `systemd-resolved` Coexistence (Ubuntu / Debian)
On Ubuntu and Debian servers, `systemd-resolved` typically binds `127.0.0.53:53`, causing `0.0.0.0:53` to fail with `EADDRINUSE`. When `bind_addr` is empty (`auto` mode) and binding `0.0.0.0:53` hits `EADDRINUSE`, `tachyon-ui` automatically falls back to the host's external non-loopback IPv4 interface (e.g. `48.102.165.41:53`) without breaking system DNS resolution, and surfaces an informational warning in the status banner.

### Bare IP vs. Domain + Let's Encrypt
- **Bare IP server (no domain):**
  - Classic **DNS `:53` + SNI Proxy `:443/:80`** works out of the box for consoles (PS5, Xbox, Switch), SmartTVs, desktops, and routers.
  - System **Android Private DNS (DoT)**, browsers, and **iOS** require a trusted CA certificate matching a hostname.
- **Domain + Let's Encrypt (recommended for DoH/DoT):**
  - Bind a domain under **Settings → Domain & TLS** (ACME HTTP-01 / DNS-01) — see [Chapter 2](02-stealth-web-domain-tls.md).
  - Check port `:443` ownership before enabling SNI `:443`: if `tachyon-core` (e.g. `VLESS REALITY`), `Unified 443 Front`, or a web site already binds `443`, move that inbound (for example to `8443`) or run SmartDNS on a dedicated node. Step 1 of the Setup Wizard probes all ports automatically.

---

## 3. One-Minute Deployment via the 4-Step Setup Wizard (`#smartdns`)

Open **SmartDNS / GeoHide** (`#smartdns`) in the sidebar and click **Quick Setup Wizard** (`openSmartDnsWizard()`):

1. **Step 1 — Server Probe (`GET /api/smart-dns/probe`):**
   - Detects the server's public IPv4, your client IP, listener availability on ports `53`, `443`, `80`, and `853`, `systemd-resolved` state, valid TLS certificate status, and local `socks_candidates`.
2. **Step 2 — Scenario & Service Presets:**
   - Choose one of 4 presets (and customize individual categories if desired):
     - **`console_tv`** (*Consoles & SmartTV*) — enables `consoles`, `streaming`, `smarttv` + DNS `:53` + SNI `:443/:80` + DoH.
     - **`ai_work`** (*AI & Productivity*) — enables `ai`, `social` + DNS `:53` + SNI `:443/:80` + DoH.
     - **`full_geohide`** (*Full GeoHide*) — enables all 6 presets + AdBlock + DNS `:53` + SNI `:443/:80` + DoH.
     - **`adblock_only`** (*Clean DNS + AdBlock*) — disables SNI proxy (`sni_proxy_enabled = false`), acting purely as a fast ad-blocking resolver.
   - Select egress mode: **`direct`** or **`socks5`** (e.g. `127.0.0.1:10808`).
3. **Step 3 — Access Control & Transports:**
   - Keep **Authorize my current IP immediately** checked and optionally add extra IPs or CIDRs.
   - Toggle desired protocols (`DNS :53`, `SNI Proxy :443/:80`, `DoH`, `DoT :853`).
4. **Step 4 — Ready (`POST /api/smart-dns/wizard-apply`):**
   - Saves config, updates firewall rules, starts listeners, and displays copy-ready cards:
     - **Primary DNS IPv4 for Consoles / SmartTV** (`effective_proxy_ipv4`),
     - **DoH URL** (`https://<host>:<port>/dns-query/<token>`),
     - **DoT Host** (`<domain>`),
     - **1-Click IP Activation (DDNS) Link** (`/dns-bind/<token>`).

---

## 4. Built-In Presets, Custom Rules & AdBlock

### Built-In Categories (`enabled_presets`)
| Preset ID | Category | Sample Domains (including all subdomains) |
| :--- | :--- | :--- |
| `consoles` | **Consoles (PlayStation, Xbox, Nintendo)** | `playstation.com`, `playstation.net`, `sonyentertainmentnetwork.com`, `psn.com`, `xbox.com`, `xboxlive.com`, `xboxservices.com`, `gamepass.com`, `nintendo.com`, `nintendo.net`, `nintendowifi.net` |
| `ai` | **AI & Assistants** | `openai.com`, `chatgpt.com`, `oaistatic.com`, `oaiusercontent.com`, `sora.com`, `anthropic.com`, `claude.ai`, `claude.com`, `gemini.google.com`, `aistudio.google.com`, `perplexity.ai`, `copilot.microsoft.com`, `cursor.com`, `midjourney.com`, `grok.com`, `x.ai` |
| `streaming` | **Streaming & Music** | `youtube.com`, `googlevideo.com`, `ytimg.com`, `youtu.be`, `netflix.com`, `nflxvideo.net`, `spotify.com`, `scdn.co`, `twitch.tv`, `ttvnw.net`, `disneyplus.com`, `soundcloud.com` |
| `games` | **PC Game Platforms** | `steampowered.com`, `steamcommunity.com`, `steamstatic.com`, `ea.com`, `origin.com`, `epicgames.com`, `ubisoft.com`, `riotgames.com`, `valorant.com`, `roblox.com`, `battle.net`, `blizzard.com`, `supercell.com` |
| `social` | **Social & Messaging** | `discord.com`, `discord.gg`, `discordapp.com`, `discord.media`, `instagram.com`, `cdninstagram.com`, `facebook.com`, `fbcdn.net`, `threads.net`, `x.com`, `twitter.com`, `tiktok.com`, `linkedin.com`, `patreon.com` |
| `smarttv` | **SmartTV & Cinema Apps** | `lgappstv.com`, `lgtvcommon.com`, `samsungcloud.com`, `samsungqbe.com`, `tv.apple.com`, `itunes.apple.com`, `roku.com`, `plex.tv`, `kinopub.net` |

### Rule Evaluation Order
1. **`bypass_domains` (`DIRECT`):** Highest priority (useful to keep bulk CDN downloads direct while proxying store/auth domains).
2. **AdBlock (`adblock_enabled` + `custom_block_domains` → `BLOCK`):** Returns `0.0.0.0` (`A`) / `::` (`AAAA`) for 16 built-in tracker/ad domains plus any custom entries.
3. **`enabled_presets` + `custom_proxy_domains` (`PROXY`):** Answers `proxy_ipv4` for `A`, `NOERROR NODATA` for `AAAA`/`HTTPS`/`SVCB`, and permits SNI proxy connections on `:443/:80`.
4. **Everything else (`DIRECT`):** Forwarded to `upstream_dns` and cached in memory.

---

## 5. Security: No Open Resolver / No Open Proxy

- **`allow_public = false` by default:** Unauthorized clients receive DNS `REFUSED` (`RCODE 5`) on `:53`/`:853`, `HTTP 403` on tokenless `/dns-query`, and immediate connection drop on SNI `:443/:80`.
- **Strict domain allowlisting on SNI proxy:** Even authorized IPs can only tunnel hostnames that match an active `PROXY` rule (`RuleDecision::Proxy`) — `:443/:80` never acts as an open proxy.
- **Three ways to authorize your devices:**
  1. **Static `allowed_ips`:** Single IPs or IPv4/IPv6 CIDRs.
  2. **Dynamic Activation Link (`/dns-bind/<token>`):** Open in a browser or trigger from a router cron job to authorize the caller's IP for `dynamic_ip_ttl_hours` (default `24` hours).
  3. **Auto-bind via DoH token (`/dns-query/<token>`):** A valid DoH request automatically authorizes the caller's IP on the fly, unlocking `:53` and `:443/:80` for other devices behind the same home NAT (consoles, SmartTVs).
- **Constant-time verification:** `bind_token` is verified with `constant_time_eq`; invalid tokens return decoy `SiteEngine` 404. Rotate tokens anytime via **Rotate Token** (`POST /api/smart-dns/rotate-token`).

---

## 6. Device Setup & Testing

### Consoles & SmartTVs
- **PS5 / PS4:** Network → Settings → Set Up Internet Connection → Advanced Settings → DNS Settings = **Manual** → Primary DNS `<VPS_IP>`, Secondary DNS `1.1.1.1`.
- **Xbox Series X|S / One:** Settings → General → Network settings → Advanced settings → DNS settings = **Manual** → Primary IPv4 DNS `<VPS_IP>`, Secondary `1.1.1.1`.
- **Nintendo Switch:** Internet Settings → Change Settings → DNS Settings = **Manual** → Primary DNS `<VPS_IP>`, Secondary `1.1.1.1`.
- **SmartTV (LG webOS / Samsung Tizen / Apple TV):** Switch network IP/DNS to Manual and set DNS to `<VPS_IP>`.

### Command-Line Verification
```bash
# 1. Proxied domain should return your VPS IP:
nslookup chatgpt.com <VPS_IP>
dig +short @<VPS_IP> playstation.com A

# 2. AAAA should return empty NOERROR NODATA (prevents IPv6 bypass):
dig @<VPS_IP> chatgpt.com AAAA

# 3. Non-matched domain should return its genuine upstream IP (DIRECT):
nslookup example.com <VPS_IP>

# 4. Transparent SNI proxy check from an authorized IP:
curl -I --resolve chatgpt.com:443:<VPS_IP> https://chatgpt.com/   # Expected: HTTP 200/301
curl -I --resolve example.com:443:<VPS_IP> https://example.com/   # Expected: connection reset/closed (not an open proxy)
```

> [!TIP]
> If `nslookup` returns `198.18.x.x` (`FakeIP`), your **router or local PC VPN/TUN client** is transparently intercepting port `53`. Temporarily disable DNS interception/FakeIP or exempt the test device to verify SmartDNS directly.

---

## 7. SmartDNS REST API & Public Endpoints

| Endpoint | Access | Description |
| :--- | :--- | :--- |
| `GET /api/smart-dns` | Admin | Returns `smart_dns` config, `presets`, live `status` (`stats`, `dynamic_ips`, `recent_logs`), and ready `doh_url` / `dot_host` / `bind_url` |
| `PUT /api/smart-dns` | Admin | Saves `SmartDnsConfig`, runs `reconcile_firewall`, and reloads listeners |
| `GET /api/smart-dns/probe` | Admin | Probes host IPv4, ports `53/443/80/853`, `systemd-resolved`, TLS cert validity, and `socks_candidates` |
| `POST /api/smart-dns/wizard-apply` | Admin | Applies scenario presets, egress mode, protocols, and optional client IP authorization |
| `POST /api/smart-dns/bind-my-ip` | Admin | Adds/removes current client IP |
| `DELETE /api/smart-dns/dynamic-ip/:ip` | Admin | Removes a dynamic client IP |
| `POST /api/smart-dns/rotate-token` | Admin | Rotates `bind_token` immediately |
| `GET /dns-bind/<token>` | Public (token) | Binds caller IP for `dynamic_ip_ttl_hours`; supports `?format=json` or HTML confirmation |
| `GET/POST /dns-query` | Public (authorized IP) | RFC 8484 DoH (`application/dns-message`); returns `403` if unauthorized |
| `GET/POST /dns-query/<token>` | Public (token) | RFC 8484 DoH with automatic caller IP authorization (decoy 404 on bad token) |
