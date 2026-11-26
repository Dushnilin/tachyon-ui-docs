# Tachyon-UI

**English** | [Русский](README.ru.md)

A lightweight, self-contained control panel for `tachyon-core` written in pure Rust: a single static binary with zero C dependencies or external runtimes, an embedded single-page UI (`src/ui.html`) with 16 built-in themes (13 dark and 3 light) plus a custom theme studio, a dual-protocol HTTP + HTTPS web server, built-in ACME automation, public website masking (`SiteEngine`), a `Unified 443 Front` profile, 4-level stealth subscriptions with Covert Knock, safe automated firewall reconciliation, a built-in Telegram bot + Mini App, and full parity with `tachyon-core` balancers, chains, routing, and runtime diagnostics.

---

## Key Features (version `v0.4.5`)

| Subsystem | Capabilities |
| :--- | :--- |
| **Zero-C Pure Rust Stack, HTTP Cache & 16 Themes** | Single static binary (`linux-amd64`, `linux-arm64` `musl`, and `windows-amd64`) with embedded responsive SPA (`src/ui.html`). TLS powered 100% by pure Rust (`rustls`, `tokio-rustls`, `oxitls-rustcrypto-provider`, no `openssl-sys` or `ring`). HTTP `gzip` compression (honoring `Accept-Encoding` / `gzip;q=0`), precompressed `UI_HTML` cache (`~225 KB` vs `~1.05 MB`), `ETag`/`304 Not Modified`, and non-blocking read-API delivery. **16 built-in themes** (13 dark + 3 light: `Material Light`, `Apple Light`, `Gruvbox Light`) and custom theme editor. |
| **Dual HTTP + HTTPS & SiteEngine** | Simultaneous HTTP and HTTPS on a single port via first-byte inspection (`0x16`), dedicated `:80` + `:443`, hot port/cert rebinding, and clear active HTTPS URL / port conflict diagnostics in the TLS summary. 6 built-in multi-page site templates, custom `.zip` hosting (up to 15 MiB), and active Honeypot. |
| **Domains, ACME & Unified 443 Front** | **5 binding profiles** (`dual_single_port`, `native_80_443`, `reality_self_steal` on `127.0.0.1:8443`, `reverse_proxy`, and **`unified_443_front` — NGINX + Core on a single port `443` TCP/UDP with hidden L7 paths, REALITY SNI mux, and `:18080` loopback masquerade for Hysteria2/TUIC**). Built-in ACME HTTP-01 (temporary `:80` listener without external Caddy/Nginx), Cloudflare DNS-01, SAN Self-Signed with 30-day auto-renewal, and X.509 metadata caching (`read_cert_info_cached`) with `(mtime, size)` invalidation. |
| **Firewall Automation & Panel Isolation** | Safe `UFW` / `firewalld` (`firewall-cmd`) / `iptables` management tagged with `tachyon-ui:managed`, automatic rule cleanup when inbounds are deleted/changed, strict protection for manual admin rules and SSH, plus an `auto_firewall_enabled` toggle. Strict `web_base_path` prefix, neutral `nginx/1.26.2` headers, `sid` cookie, and 9-check Self-Audit (`0–100`). |
| **4-Level Stealth Subscriptions & Covert Knock** | Polymorphic paths (`sub`, `static_feed`, `cdn_v2`, `rss_updates`), Zero-Oracle 404, scanner filter, Portal Guard (`?k=`), HMAC knock (`If-None-Match` / `X-Client-Digest` / `?v=`), subscription profiles, portal branding, separate multi-subscription accounting (`sub_online_ips`, `InboundUserNameResolver`), in-place `sub_id` rename (`old_sub_id`), HWID/Anti-Share, dynamic query filters (`?format=`, `?country=`, `?mode=`), and export for `sing-box`/`tachyon`, `clash`/`mihomo`, `base64`/`raw`, `sip008`, `wg`/`awg`, `openwrt` + 1-click installer. |
| **`tachyon-core` Balancers** | All 12 `urltest` modes, full `balancer` object (`pool`, `pool_tolerance`, `sticky_hash`, `quota_mib`, `tiers`), `selector` (`auto_failover`, live member switching), and multi-hop `chain` (`strip_evasion`, `strip`, `rewrite`). |
| **Protocols, Routing & Automation** | VLESS/REALITY, VMess, Trojan, Hysteria/Hy2, TUIC, AnyTLS, Naive, ShadowTLS, WireGuard/AWG (Port Hopping, automatic free `wg_client_ip` allocation), MASQUE (`keep_alive_period: 20s`, `idle_timeout: 50s` for stable router HTTP/3 tunnels), Tewfai, FPTN, TrustTunnel, OpenConnect, Mieru, Snell, MTProto (`telegram-ws`/`kws`), WDTT, Sudoku, Cloudflared, SSH, Tor, Bridge, Desync; country groups with zero-reload metadata edits & batch country actions; `.trs` lists, `time_window`, `udp_nat_mode`, `smart_queue`, SNI scanner, Auto-Healer, Unlock Probe, Cloudflare DNS Failover, Zero-Traffic Watchdog, and Smart Relay Failover. |
| **SmartDNS / GeoHide & SNI Proxy** | Built-in pure-Rust DNS server (UDP/TCP `:53` with automatic `systemd-resolved` fallback), DoH (RFC 8484 `/dns-query`), DoT (`:853`), and transparent SNI proxy (`:443` / `:80`) over `direct` or `socks5` for PS5, Xbox, Nintendo Switch, SmartTVs, desktops, and routers. 6 curated presets, built-in AdBlock, Open Resolver / Open Proxy protection (`allow_public=false`, `/dns-bind/<token>` DDNS link), live query log, and a 4-step Setup Wizard. |
| **Cluster, Telegram Bot & Diagnostics** | Multi-node sync and cascades, built-in shop bot with plans, promo codes, vouchers (`/redeem`), and Mini App, 5-tab live core diagnostics modal, bottom **Live Connections Dock** (`Ctrl+\``), Privacy Mask (`Alt+P`), Command Palette (`Ctrl+K`), quick view switching (`Alt+1..8`), and 10-revision snapshot rollback ring. |

---

## Quick Start

### One-Command Installer (Ubuntu / Debian / RHEL / Alpine Linux)

```bash
bash <(curl -Ls https://raw.githubusercontent.com/Dushnilin/tachyon-ui/main/install.sh)
```

The installer sets up the `tachyon-ui` binary, a `systemd` service, and the `tachyon-panel` CLI, opens the panel port in the firewall, and prints the panel URL and initial credentials (change them immediately in Settings after first sign-in).

### Docker Deployment

```bash
docker build -t tachyon-ui .
docker run -d --name tachyon-ui --restart unless-stopped \
  -p 2053:2053 -p 443:443 \
  -v /opt/tachyon-ui/data:/data \
  tachyon-ui
```

See [Chapter 1](docs/en/01-installation-and-cli.md) for environment flags (`ADMIN_PASS`, `PANEL_PORT`, `PUBLIC_IP`, `SERVER_COUNTRY`, `OPEN_WEB_PORTS`), static `x86_64`/`aarch64` `musl` cross-compilation, automated firewall management, and CLI tools (`tachyon-panel info`, `reset-pass`, `reset-path`, `open-web-ports`, `tachyon-ui --reset-creds`).

---

## Interface Overview

![Tachyon-UI Dashboard](docs/images/en/02-dashboard.png)

| Site, Domain, HTTP/HTTPS & Defense Wizard | Balancer Builder (12 `urltest` Modes, `tiers`, `chain`) |
| :---: | :---: |
| ![Site & Domain Wizard](docs/images/en/10-wizard-site.png) | ![Balancer Builder](docs/images/en/06-balancer-builder.png) |

---

## Documentation Index

1. [Installation, Docker, Source Build & CLI](docs/en/01-installation-and-cli.md)
2. [Public Website, Dual HTTP+HTTPS, Domain Profiles, ACME & Self-Audit](docs/en/02-stealth-web-domain-tls.md)
3. [Subscriptions, Web Portal, Vouchers & Tachyon Covert Knock](docs/en/03-subscriptions-and-covert-knock.md)
4. [Balancers (12 `urltest` Modes), `chain` Outbounds & Routing](docs/en/04-balancers-chain-and-routes.md)
5. [Protocols, Multi-Node Clusters, Cascades & Telegram Bot](docs/en/05-protocols-nodes-and-tgbot.md)
6. [Core Diagnostics, Privacy Mask (`Alt+P`) & REST API Reference](docs/en/06-diagnostics-privacy-and-api.md)
7. [Self-Hosted SmartDNS / GeoHide, SNI Proxy, DoH/DoT & Setup Wizard](docs/en/07-smartdns-geohide-guide.md)
