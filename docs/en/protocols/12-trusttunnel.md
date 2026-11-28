# 12. TrustTunnel (HTTP/2 & HTTP/3 QUIC)

> **Summary Card**
> - **Role in `tachyon-core`**: `Inbound` (server) and `Outbound` (client)
> - **L4 Transport**: `TCP` (HTTP/2 `CONNECT` over TLS — default) or `UDP` (HTTP/3 QUIC with ALPN `h3` when `quic: true`). Carries TCP streams, UDP datagrams (via the internal `_udp2` pseudo-host and UoT), and ICMP echo
> - **TLS / REALITY / ACME**: certificate-backed `TLS` (`tlsModes: ['tls']`) is required in the panel UI; in `tachyon-core`, `tls` is strictly mandatory on `Inbound` when `quic: true` and always mandatory (`tls.enabled: true`) on `Outbound`. REALITY is not supported
> - **Domain Required**: a domain with your own TLS certificate (via ACME in `tachyon-ui`) is recommended; on `Outbound`, standard `server_name` and TLS options apply for testing

---

## Purpose & How It Works in `tachyon-core`

**Simple Analogy.** Think of **TrustTunnel** as a dual-stack HTTPS gateway that speaks either classic **HTTP/2** (over TCP) or low-latency **HTTP/3 QUIC** (over UDP). Inside a clean HTTPS connection with standard HTTP Basic authentication, it carries web TCP streams, gaming and voice UDP datagrams, and even `ping` (ICMP echo) — looking on the wire like ordinary traffic to a modern HTTPS endpoint.

**Engine Mechanics in `tachyon-core`:**
- In default mode (`quic: false`), TrustTunnel runs over **TCP + TLS + HTTP/2 `CONNECT`**, providing maximum compatibility in networks where UDP or QUIC is blocked.
- When **`quic: true`** (`inb-tt-quic` in the UI) is enabled, `tachyon-core` serves **HTTP/3 QUIC over UDP** (ALPN `["h3"]`), cutting connection setup latency and eliminating TCP head-of-line blocking. On `Outbound`, `quic_congestion_control` can be tuned via JSON.
- Authentication uses **HTTP Basic** (`username` + `password`). On `Inbound`, `tachyon-core` performs constant-time password verification to resist timing side-channels, and `users[]` cannot be empty. On `Outbound`, `username` and `password` must always be supplied **as a pair** (setting one without the other is rejected).
- UDP datagrams are multiplexed via the internal `_udp2` pseudo-host and UoT, alongside native ICMP echo forwarding.

---

## Panel Form Screenshot

![TrustTunnel configuration form in tachyon-ui](../../images/protocols/en/proto-12-trusttunnel.png)

---

## Step-by-Step Configuration in `tachyon-ui`

### Inbound Setup

1. Open **Inbounds** and click **Add Inbound**.
2. Select **TrustTunnel (`trusttunnel`)** as the protocol.
3. Choose one of the built-in panel presets (`PROTOCOL_PRESETS.trusttunnel`):
   - **TrustTunnel HTTP/2** — port `8465` (TCP), `tls`, SNI `www.microsoft.com`, ALPN `h2,http/1.1`, `quic: false`.
   - **TrustTunnel HTTP/3 QUIC** — port `8466` (UDP), `tls`, SNI `www.cloudflare.com`, ALPN `h3`, `quic: true`.
4. Set the **HTTP/3 QUIC Mode** toggle (`inb-tt-quic`):
   - Off (`false`) for TCP / HTTP/2.
   - On (`true`) for UDP / HTTP/3 QUIC.
5. On the **TLS** tab, set **Server Name (SNI)** (ideally your domain `panel.example.com` matching `data/certs/server.crt` / `server.key`) and verify ALPN matches the transport (`h2,http/1.1` for HTTP/2, or `h3` for HTTP/3).
6. Add at least one active user (`username` and `password`) on the **Clients** tab, click **Save**, and **Apply Changes**.

### Outbound Setup

1. Open **Outbounds** and choose type **`trusttunnel`**.
2. Enter the server (`panel.example.com` or `198.51.100.10`), port (`8465` / `8466`), and password (`out-password`).
3. **Required**: supply **`username`** in **Advanced (raw JSON merge)** (`out-settings`, e.g., `{"username": "alice"}`), and add `"quic": true` (plus optional `"quic_congestion_control": "bbr"`) when connecting over HTTP/3.
4. Under **TLS**, keep **Enable TLS** active and set `server_name` / `alpn`.

---

## Field, Toggle & Parameter Reference

### Inbound Node

| Panel Field (`ID`) | `tachyon-core` JSON Key | Allowed Values / Default | Purpose & Configuration Notes |
|---|---|---|---|
| **HTTP/3 QUIC Mode** (`inb-tt-quic`) | `quic` | `false` (HTTP/2 TCP, default) / `true` (HTTP/3 UDP) | Switches between HTTP/2 over TLS (TCP) and HTTP/3 over QUIC (UDP). When `true`, `tachyon-ui` emits `quic: true` and updates firewall rules accordingly. |
| **Clients** (Clients tab) | `users[].{username, password}` | Non-empty array of `{ username, password }` | HTTP Basic credentials. When zero active clients exist, the panel inserts `__disabled__`. |
| **TLS Mode** (`inb-tls-mode`) | `tls.enabled: true` | `tls` only | Certificate-backed TLS is required in the UI; in `tachyon-core`, `tls` is strictly mandatory when `quic: true`. |
| **SNI / ALPN / Certificate** (`inb-tls-*`) | `tls.{server_name, alpn, certificate_path, key_path}` | `h2,http/1.1` for HTTP/2; `h3` for QUIC | Using a real domain and a trusted certificate issued via `tachyon-ui` (ACME) is recommended. |

### Outbound Node

| Panel Field (`ID` / `out-settings`) | `tachyon-core` JSON Key | Allowed Values / Default | Purpose & Configuration Notes |
|---|---|---|---|
| **Server / Port** (`out-server`, `out-port`) | `server`, `server_port` | Host and port (`8465` / `8466`) | Remote TrustTunnel server endpoint. |
| **Password & Username** (`out-password` + `out-settings`) | `password`, `username` | Mandatory pair of non-empty strings | On `Outbound`, `username` and `password` are only accepted together. Provide `username` inside `out-settings`. |
| **HTTP/3 QUIC** (`out-settings`) | `quic`, `quic_congestion_control` | `bool`; optional CC algorithm (`bbr`, `cubic`, etc.) | Enables HTTP/3 instead of HTTP/2 on the client outbound. |
| **TLS** (`out-tls*`) | `tls.{enabled, server_name, alpn, insecure}` | `tls.enabled: true` strictly mandatory | Must match the server's TLS settings. |

---

## Networking, TLS & Firewall

- **Firewall Ports (`inbound_firewall_rules`)**:
  - When `trusttunnel_quic: false`, **`TCP`** is opened on `inb.port` (preset `8465`).
  - When `trusttunnel_quic: true`, both **`TCP` and `UDP`** are opened on `inb.port` (preset `8466`).
- **ACME**: `tachyon-core` does not launch an embedded ACME client; `tachyon-ui` manages certificate issuance/renewal and passes `certificate_path` / `key_path` to the core.
- **Reverse Proxy**: in HTTP/2 mode (`quic: false`), an L4 SNI proxy (`stream`) on port `443` works cleanly with `external_port: 443` set on the inbound. If routing behind an L7 HTTP proxy, verify it supports `CONNECT` tunneling and preserves the `Authorization` header.

---

## Subscription Export, Clients & `tachyon-core` JSON Example

- **URI** (`?format=raw` or Base64):
  - HTTP/2: `trusttunnel://alice:password@panel.example.com:8465?sni=panel.example.com#TrustTunnel-H2`
  - HTTP/3: `trusttunnel://alice:password@panel.example.com:8466?quic=1&sni=panel.example.com#TrustTunnel-H3`
  *(If `allow_insecure: true`, `&insecure=1` is appended.)*
- **Client JSON** (`?format=singbox`, `?format=tachyon`, `?format=outbounds`): exported as `"type": "trusttunnel"`, `"username"`, `"password"`, `"quic": true` (when QUIC is enabled), and `"tls"`.
- **Clash Meta YAML** and **SIP008**: **not exported** — use `?format=singbox`, `?format=tachyon`, or URIs.
- **Compatible Clients**: `tachyon-core` (including OpenWrt routers) and clients supporting TrustTunnel HTTP/2 / HTTP/3.

### Valid `tachyon-core` JSON Example

```json
{
  "inbounds": [
    {
      "type": "trusttunnel",
      "tag": "tt-h2-in",
      "listen": "::",
      "listen_port": 8465,
      "users": [
        {
          "username": "alice",
          "password": "example-tt-password"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "panel.example.com",
        "alpn": ["h2", "http/1.1"],
        "certificate_path": "data/certs/server.crt",
        "key_path": "data/certs/server.key"
      }
    },
    {
      "type": "trusttunnel",
      "tag": "tt-h3-in",
      "listen": "::",
      "listen_port": 8466,
      "quic": true,
      "users": [
        {
          "username": "alice",
          "password": "example-tt-password"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "panel.example.com",
        "alpn": ["h3"],
        "certificate_path": "data/certs/server.crt",
        "key_path": "data/certs/server.key"
      }
    }
  ],
  "outbounds": [
    {
      "type": "trusttunnel",
      "tag": "tt-h3-out",
      "server": "panel.example.com",
      "server_port": 8466,
      "username": "alice",
      "password": "example-tt-password",
      "quic": true,
      "tls": {
        "enabled": true,
        "server_name": "panel.example.com",
        "alpn": ["h3"]
      }
    }
  ]
}
```

---

## Diagnostics & Troubleshooting

| Symptom | Root Cause | Resolution |
|---|---|---|
| `Outbound` validation fails on save/start: incomplete credentials | `password` provided without `username` (or vice versa) in a manual `Outbound` | Supply both fields as a pair (`username` in `out-settings`, `password` in the form). |
| Client cannot connect in HTTP/3 (`quic: true`) mode | UDP port `8466` is closed on the server, or client ISP throttles/blocks QUIC/UDP | Open UDP on the inbound port, or switch to the **TrustTunnel HTTP/2** (TCP) preset. |
| Client fails TLS check without a custom domain | Self-signed certificate is not allowed on the client | Issue a trusted certificate for a domain (recommended), or enable `allow_insecure` only in test environments. |
| Core startup `tls` error with `quic: true` | Manual JSON omits the `tls` object when `quic: true` is set | A `tls` block with `enabled: true` is strictly required for HTTP/3 QUIC. |

---

## See Also

- [Protocol Catalog](./README.md)
- [Complete End-to-End Guide](../00-complete-guide.md)
- [Protocol Encyclopedia](../09-protocols-encyclopedia.md)
- [UI Every Tab & Button Guide](../11-ui-every-tab-and-button-guide.md)
- [10. NaiveProxy](./10-naive.md)
- [11. MASQUE (HTTP/3 CONNECT-IP)](./11-masque.md)
- [13. Mieru v3](./13-mieru.md)
