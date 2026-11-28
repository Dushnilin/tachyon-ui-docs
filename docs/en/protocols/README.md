# Tachyon Protocol Catalog (`tachyon-ui v0.4.12` + `tachyon-core 0.6.9`)

[Complete Guide](../00-complete-guide.md) | [Protocol Encyclopedia](../09-protocols-encyclopedia.md) | [UI Reference](../11-ui-every-tab-and-button-guide.md)

Practical reference for all **27 protocols, transports, and routing meta-outbounds** in the `tachyon-ui v0.4.12` panel and `tachyon-core 0.6.9` engine: UI form fields, JSON schema mapping, presets, core constraints, firewall rules, subscription export formats, and diagnostics.

---

## Quick Protocol Selection Matrix

| Scenario / Goal | Recommended Protocol | L4 Transport | TLS & Own Domain | Why It Fits Best |
| :--- | :--- | :--- | :--- | :--- |
| **Primary all-round Tachyon channel** | [01. TewFai](./01-tewfai.md) | TCP + QUIC (UDP) + HTTP | REALITY (no domain needed) or TLS | Unified session with seamless carrier handover, traffic morphing, XOR-FEC, PQC |
| **Quick start without buying a domain** | [02. VLESS](./02-vless.md) (`REALITY`) / [09. ShadowTLS v3](./09-shadowtls.md) | TCP | REALITY / decoy TLS | Mimics a genuine TLS 1.3 handshake to a public website |
| **Lossy mobile links, packet loss** | [01. TewFai](./01-tewfai.md) (QUIC + FEC) / [03. Hysteria2](./03-hysteria2.md) / [04. TUIC v5](./04-tuic.md) | UDP (QUIC HTTP/3) | Mandatory TLS | Eliminates TCP head-of-line blocking; BBR/Brutal or FEC sustains throughput |
| **Corporate networks, CDN, strict proxies** | [10. NaiveProxy](./10-naive.md) / [12. TrustTunnel](./12-trusttunnel.md) / [02. VLESS](./02-vless.md) (`ws`/`xhttp`) | TCP (H2) / UDP (H3) | Real domain + certificate required | Standard HTTP/2 or HTTP/3 traffic framing |
| **Gaming, voice calls, lowest UDP ping** | [04. TUIC v5](./04-tuic.md) / [07. Shadowsocks](./07-shadowsocks.md) | UDP / TCP+UDP | TLS (TUIC) / No TLS (SS-2022) | Native UDP datagram relay and 0-RTT handshake |
| **L3 VPN for routers (OpenWrt / Keenetic)** | [22. AmneziaWG](./22-amneziawg.md) / [21. WireGuard](./21-wireguard.md) / [11. MASQUE](./11-masque.md) | UDP (L3 IP) | No TLS (WG/AWG) / TLS (MASQUE) | Minimal CPU overhead on embedded router SoCs |
| **Legacy & third-party client compatibility** | [05. Trojan](./05-trojan.md) / [06. VMess](./06-vmess.md) / [07. Shadowsocks](./07-shadowsocks.md) | TCP (+ UDP in SS) | TLS / REALITY (Trojan/VMess) | Supported by virtually any third-party client |
| **Failover and multi-hop cascades** | [27. Balancers and Chain](./27-balancers-and-chain.md) | Inherited from nodes | Inherited from nodes | 12 `urltest` strategies, manual `selector`, and multi-hop `chain` |

---

## Common Mechanisms in `tachyon-ui` and `tachyon-core 0.6.9`

### TLS, REALITY, and ACME Modes
- **Available TLS tab modes (`tls.mode`)** are filtered per protocol capability in the panel:
  - **No TLS (`[]`)**: `shadowsocks`, `shadowtls`, `snell`, `mieru`, `mtproto`, `wdtt`, `wireguard`, `amneziawg`, `socks`, `cloudflared`.
  - **Certificate TLS only (`['tls']`)**: `hysteria2`, `tuic`, `naive`, `masque`, `trusttunnel`, `openconnect`.
  - **TLS or REALITY (`['tls', 'reality']`)**: `anytls`, `fptn`.
  - **Plain or Certificate TLS (`['none', 'tls']`)**: `mixed`, `http`, `sudoku`.
  - **Full spectrum (`['none', 'tls', 'reality']`)**: `tewfai`, `vless`, `vmess`, `trojan`.
- **ACME certificates**: Built-in server runtime ACME is disabled in `tachyon-core 0.6.9` (the schema validator accepts `tls.acme` / `tls.certificate_provider` fields, but starting the server runtime with them returns an error). In production, `tachyon-ui` manages Let's Encrypt issuance and renewal itself and passes ready `certificate_path` (`data/certs/server.crt`) and `key_path` (`data/certs/server.key`) files to the engine.
- **Reverse proxy**: The `external_port` and `external_tls` Inbound fields override the port and TLS mode in exported client subscriptions when traffic enters through an external NGINX or Caddy proxy on port `443`.

### "Transport & Mux" Tab
For **VLESS**, **VMess**, and **Trojan**, shared L7 transports (`tcp`, `ws`, `grpc`, `httpupgrade`, `xhttp`, `http`) and the `h2mux` multiplexer (+ `brutal` bandwidth pacing) are available. For **Shadowsocks**, the `Mux` tab (`h2mux` + `brutal`) is available. On `vless`, if any assigned client uses `xtls-rprx-vision`, multiplexing is disabled automatically.

### Subscription Export Formats (`GET /sub/{sub_id}?format=...`)
| `format` Parameter | Panel Output | Compatibility Notes |
| :--- | :--- | :--- |
| *(empty)* / `raw` / `uri` | Base64-encoded or line-separated URI list | Universal links: `tewfai://`, `vless://`, `hy2://`, `tuic://`, `trojan://`, `vmess://`, etc. |
| `json` / `singbox` / `tachyon` / `outbounds` | `tachyon-core` client config JSON | Full support for all native fields (TewFai, XHTTP, HTTPUpgrade, PQC, balancers, `endpoints[]`) |
| `clash` / `clashmeta` / `meta` / `mihomo` / `yaml` | Clash Meta YAML profile | Only format-supported types (`vless`, `trojan`, `anytls`, `vmess`, `hysteria2`, `tuic`, `shadowsocks`, `mieru`, `snell`, `socks`/`http`, `wireguard`); `tewfai`, `xhttp`, `httpupgrade`, `chain`, `naive`, `masque`, `trusttunnel`, `fptn`, `wdtt` are omitted in YAML |
| `sip008` | SIP008 JSON (`version: 1`) | Only for `shadowsocks` |
| `openwrt` / `router` / `html` | Router JSON / Web Portal | Automated routing profile for OpenWrt gateways and client portal page |

---

## Complete Catalog of Protocol Guides (`01..27`)

| # | Protocol / Group | Panel Role | L4 Transport | TLS / REALITY | Status in `tachyon-core 0.6.9` | Guide |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| `01` | **TewFai** | Inbound + Outbound | TCP + QUIC (UDP) + HTTP | `none` / `tls` / `reality` | Full support | [01-tewfai.md](./01-tewfai.md) |
| `02` | **VLESS** | Inbound + Outbound | TCP (+ HTTP carriers; `quic` Outbound only) | `none` / `tls` / `reality` | Full support | [02-vless.md](./02-vless.md) |
| `03` | **Hysteria2** | Inbound + Outbound | UDP (QUIC ALPN `h3`) | `tls` only | Full support | [03-hysteria2.md](./03-hysteria2.md) |
| `04` | **TUIC v5** | Inbound + Outbound | UDP (QUIC ALPN `h3`) | `tls` only | Full support | [04-tuic.md](./04-tuic.md) |
| `05` | **Trojan** | Inbound + Outbound | TCP (+ HTTP carriers; `quic` Outbound) | `none` / `tls` / `reality` | Full support | [05-trojan.md](./05-trojan.md) |
| `06` | **VMess** | Inbound + Outbound | TCP (+ HTTP carriers; `quic` Outbound) | `none` / `tls` / `reality` | Full support (Inbound AEAD `alterId: 0`) | [06-vmess.md](./06-vmess.md) |
| `07` | **Shadowsocks** | Inbound + Outbound | TCP + UDP (UoT v1/v2) | Hidden in UI | Full support (AEAD & SS-2022) | [07-shadowsocks.md](./07-shadowsocks.md) |
| `08` | **AnyTLS** | Inbound + Outbound | TCP | `tls` / `reality` | Full support | [08-anytls.md](./08-anytls.md) |
| `09` | **ShadowTLS v3** | Inbound + Outbound | TCP | Delegated external TLS handshake | Full support (`v3`, `v1` rejected) | [09-shadowtls.md](./09-shadowtls.md) |
| `10` | **NaiveProxy** | Inbound + Outbound | TCP (H2) / UDP (H3 QUIC) | `tls` only (`insecure` forbidden) | Full support | [10-naive.md](./10-naive.md) |
| `11` | **MASQUE** | Inbound + Outbound | UDP (H3 `CONNECT-IP`) / H2 | `tls` only | Full support | [11-masque.md](./11-masque.md) |
| `12` | **TrustTunnel** | Inbound + Outbound | TCP (H2) / UDP (H3 QUIC) | `tls` only | Full support | [12-trusttunnel.md](./12-trusttunnel.md) |
| `13` | **Mieru** | Inbound + Outbound | TCP or UDP | Built-in encryption | Full support (Mieru v3) | [13-mieru.md](./13-mieru.md) |
| `14` | **Snell** | Inbound + Outbound | TCP (+ UDP relay) | No TLS | Inbound v5/v6; Outbound v4/v5/v6 | [14-snell.md](./14-snell.md) |
| `15` | **MTProto** | Inbound + Outbound | TCP (+ `telegram-ws` / `kws`) | No TLS (or TLS for `kws`) | Full support (`32 hex` secret) | [15-mtproto.md](./15-mtproto.md) |
| `16` | **FPTN** | Inbound + Outbound | TCP (HTTPS + WS) | `tls` / `reality` | Full support (`RealityMode2`) | [16-fptn.md](./16-fptn.md) |
| `17` | **WDTT** | Inbound + Outbound | UDP (RTP / DTLS / WG / CSQTT) | Built-in encryption | Full support | [17-wdtt.md](./17-wdtt.md) |
| `18` | **Sudoku** | Inbound + Outbound (form/schema) | TCP (+ HTTP Mask) | `none` / `tls` | Schema check & URI export (no runtime in `0.6.9`) | [18-sudoku.md](./18-sudoku.md) |
| `19` | **OpenConnect** | Inbound + Outbound | TCP (CSTP) + UDP (DTLS/ESP) | `tls` only | Full support (6 dialects) | [19-openconnect.md](./19-openconnect.md) |
| `20` | **Cloudflared** | Inbound only | Outbound QUIC / H2 to Edge | Cloudflare Token | Requires build with `with_cloudflared` feature | [20-cloudflared.md](./20-cloudflared.md) |
| `21` | **WireGuard** | Endpoint (Inbound + Outbound UI) | UDP (L3 IP) | No TLS (`Noise_IKpsk2`) | Full support in `endpoints[]` | [21-wireguard.md](./21-wireguard.md) |
| `22` | **AmneziaWG** | Endpoint (Inbound + Outbound UI) | UDP (L3 IP + Junk/CPS) | No TLS (`Noise_IKpsk2`) | Full support (AWG 2.0/3.x in `endpoints[]`) | [22-amneziawg.md](./22-amneziawg.md) |
| `23` | **Mixed / SOCKS / HTTP** | Inbound + Outbound | TCP (+ UDP for SOCKS5/Mixed) | `none` / `tls` (`mixed`, `http`) | Full support | [23-mixed-socks-http.md](./23-mixed-socks-http.md) |
| `24` | **Desync** | Outbound only | TCP / UDP (direct egress) | No TLS | Full support (adaptive mode & `udp_fake`) | [24-desync.md](./24-desync.md) |
| `25` | **Hysteria (v1)** | Outbound (schema) | UDP (`udp`/`wechat-video`/`faketcp`) | `tls` required | Schema check (no runtime in `0.6.9`, use [03-hysteria2.md](./03-hysteria2.md)) | [25-hysteria1.md](./25-hysteria1.md) |
| `26` | **SSH / Tor / Bridge** | Outbound only (schema) | TCP (`ssh`) / process (`tor`) / L2-L3 (`bridge`) | No TLS | Schema check (no runtime in `0.6.9`) | [26-ssh-tor-bridge.md](./26-ssh-tor-bridge.md) |
| `27` | **Balancers and Chain** | Outbound groups | Inherited from nodes | Inherited from nodes | Full support (`urltest`, `selector`, `chain`) | [27-balancers-and-chain.md](./27-balancers-and-chain.md) |

---

## See Also
- [Protocol Encyclopedia in Plain Language](../09-protocols-encyclopedia.md)
- [Complete End-to-End Guide](../00-complete-guide.md)
- [UI Reference: Every Tab and Button](../11-ui-every-tab-and-button-guide.md)
