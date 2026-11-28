# 10. NaiveProxy (HTTPS/2 & HTTP/3 QUIC)

> **Summary Card**
> - **Role in `tachyon-core`**: `Inbound` (server) and `Outbound` (client)
> - **L4 Transport**: `TCP` (HTTP/2 `CONNECT` over TLS) and/or `UDP` (HTTP/3 QUIC with ALPN `h3` when `naive_quic: true`). Client UDP flows are tunneled over built-in `UoT v1/v2` (`udp_over_tcp`)
> - **TLS / REALITY / ACME**: certificate-backed `TLS` (`tls.enabled: true`) is **strictly mandatory** (`tlsModes: ['tls']`). REALITY is not supported. **`tls.insecure: true` and `tls.disable_sni: true` are strictly forbidden** on `Outbound`
> - **Domain Required**: **yes, strictly mandatory** — a real domain with a trusted CA/ACME TLS certificate whose SAN matches `server_name`

---

## Purpose & How It Works in `tachyon-core`

**Simple Analogy.** Most TLS-wrapped proxies merely place their own framing inside TLS, leaving HTTP header anomalies, response lengths, and error behavior exposed to traffic analysis. **NaiveProxy** operates as a genuine HTTP/2 or HTTP/3 web server backed on the client side by Chromium's network stack: traffic travels inside standard `CONNECT` tunnels with randomized HTTP/2 frame padding, making it virtually indistinguishable from ordinary HTTPS browsing.

**Engine Mechanics in `tachyon-core`:**
- In default mode (HTTP/2 over TLS), the server terminates HTTPS connections, validates `Basic` authentication (`users[].{username, password}`), and opens `CONNECT` tunnels while adding random padding to HTTP/2 frames to obscure payload sizes.
- When **HTTP/3 QUIC** (`inb-naive-quic` = `true`) is enabled, the panel emits `"network": "udp"` and `alpn: ["h3"]`, switching the transport to QUIC over UDP.
- **Strict `naive` Security Invariant**: the `tachyon-core` validator **explicitly forbids** `tls.insecure: true` and `tls.disable_sni: true` on `naive` outbounds (`insecure is not supported on naive outbound`). Consequently:
  1. `tachyon-ui` hides the self-signed certificate selector (`inb-tls-insecure-wrap`) for `naive`.
  2. Subscription generation forces `insecure = false` (never emitting `insecure=1` in URIs or `insecure: true` in JSON).
  3. Subscription export automatically resolves `sni` to a domain present in the SAN of the server's active certificate so clients pass strict PKI verification.
- If the client does not trust the server's certificate (for instance, a self-signed certificate not installed in the system root store), NaiveProxy rejects the connection immediately during the TLS handshake.

---

## Panel Form Screenshot

![NaiveProxy configuration form in tachyon-ui](../../images/protocols/en/proto-10-naive.png)

---

## Step-by-Step Configuration in `tachyon-ui`

### Prerequisite: Domain & Trusted Certificate

1. Point your domain's `A`/`AAAA` record (e.g., `panel.example.com`) to the server IP `198.51.100.10`.
2. Issue a trusted TLS certificate (using the panel's built-in ACME manager or an external tool writing to `data/certs/server.crt` and `data/certs/server.key`) with your domain in the certificate's SAN.

### Inbound Setup

1. Open **Inbounds** and click **Add Inbound**.
2. Select **NaiveProxy (`naive`)** as the protocol.
3. Choose one of the panel presets (`PROTOCOL_PRESETS.naive`):
   - **NaiveProxy HTTPS/2** — port `8450`, TLS `tls`, SNI `www.microsoft.com` (automatically replaced by the panel's `web_domain` when set), ALPN `h2,http/1.1`, `quic: false`.
   - **NaiveProxy HTTP/3 QUIC** — port `8459`, TLS `tls`, SNI `www.cloudflare.com` (or `web_domain`), ALPN `h3`, `quic: true`.
4. Toggle **HTTP/3 QUIC Mode (UDP)** (`inb-naive-quic`): keep off for TCP/HTTP/2, or turn on for UDP/HTTP/3.
5. On the **TLS** tab:
   - Enter your real domain in **Server Name (SNI)** (`inb-tls-sni`) — it **must** match the certificate SAN.
   - Verify the certificate and private key paths (`inb-tls-cert`, `inb-tls-key`). Notice that `allow_insecure` is intentionally hidden in the UI for `naive`.
6. Add clients (`username` and `password`) on the **Clients** tab, click **Save**, and **Apply Changes**.

### Outbound Setup

1. Open **Outbounds** and choose type **`naive`**.
2. Enter the server (`panel.example.com` or `198.51.100.10`), port (`8450` / `8459`), username (`out-socks-user`), and password (`out-socks-pass`).
3. Under **TLS & Transport**, keep **Enable TLS** enabled, **do not** enable insecure/self-signed verification, and set **Server Name (SNI)** to the domain matching the server certificate's SAN.

---

## Field, Toggle & Parameter Reference

### Inbound Node

| Panel Field (`ID`) | `tachyon-core` JSON Key | Allowed Values / Default | Purpose & Configuration Notes |
|---|---|---|---|
| **HTTP/3 QUIC Mode (UDP)** (`inb-naive-quic`) | `network: "udp"` (when `true`) | `false` (HTTPS/2 TCP) / `true` (HTTP/3 UDP) | Adds `"network": "udp"` and `alpn: ["h3"]` to the inbound config. In exported subscriptions, switches the URI scheme to `naive+quic://` and adds `"quic": true` to JSON outbounds. |
| **Clients** (Clients tab) | `users[].{username, password}` | Non-empty `username` and `password` | HTTP Basic credentials for `CONNECT` requests. When zero active clients exist, the panel inserts `__disabled__`. |
| **TLS Mode** (`inb-tls-mode`) | `tls.enabled: true` | `tls` only | Mandatory certificate-backed TLS. REALITY and `none` are prohibited. |
| **Server Name (SNI)** (`inb-tls-sni`) | `tls.server_name` | Real domain matching certificate SAN (`panel.example.com`) | Critical setting: if `server_name` does not match the trusted certificate SAN, `naive` clients refuse to connect. |
| **ALPN** (`inb-tls-alpn`) | `tls.alpn` | `["h2", "http/1.1"]` for HTTP/2; `["h3"]` for QUIC | Advertises supported HTTP versions during TLS negotiation. |
| **Certificate & Key** (`inb-tls-cert`, `inb-tls-key`) | `tls.certificate_path`, `tls.key_path` | Defaults to `data/certs/server.crt` / `data/certs/server.key` | Public certificate (preferably fullchain from a trusted CA / Let's Encrypt) and private key. |

### Outbound Node

| Panel Field (`ID`) | `tachyon-core` JSON Key | Allowed Values / Default | Purpose & Configuration Notes |
|---|---|---|---|
| **Server / Port** (`out-server`, `out-port`) | `server`, `server_port` | Domain or IP and port (`8450` / `8459`) | Using the certificate domain directly in `server` (or setting it in `tls.server_name`) is recommended. |
| **Username / Password** (`out-socks-user`, `out-socks-pass`) | `username`, `password` | Strings | HTTP Basic credentials matching a server user. |
| **QUIC Mode** (`out-settings`) | `quic`, `quic_congestion_control` | `bool`, `bbr` / `cubic` / `reno` / `bbr2` | Enables HTTP/3 QUIC transport on the outbound. |
| **TLS** (`out-tls*`) | `tls.{enabled, server_name, alpn}` | `tls.enabled: true`, **`insecure` and `disable_sni` forbidden** | Setting `insecure: true` or `disable_sni: true` on a `naive` outbound fails `tachyon-core` validation. |

---

## Networking, TLS & Firewall

- **Firewall Ports (`inbound_firewall_rules`)**:
  - When `naive_quic: false`, **`TCP`** is opened on `inb.port` (preset `8450`).
  - When `naive_quic: true`, both **`TCP` and `UDP`** are opened on `inb.port` (preset `8459`).
- **ACME & Certificates**: `tachyon-core` does not run an embedded ACME client at runtime (schema validation accepts `acme` fields, but runtime execution is disabled). Manage certificate issuance and renewal through `tachyon-ui` or an external ACME client (such as Caddy or certbot).
- **Reverse Proxy**: if port `443` is owned by a web server (Caddy/NGINX), Caddy can terminate TLS and forward to NaiveProxy via `forward_proxy`, or you can use L4 SNI routing (`stream`) while setting `external_port: 443` in the inbound for accurate client links.

---

## Subscription Export, Clients & `tachyon-core` JSON Example

- **URI** (`?format=raw` or Base64):
  - HTTP/2: `naive+https://alice:password@panel.example.com:8450?sni=panel.example.com#NaiveProxy`
  - HTTP/3 QUIC: `naive+quic://alice:password@panel.example.com:8459?sni=panel.example.com#NaiveProxy-QUIC`
  *(The `insecure=1` query parameter is never emitted.)*
- **Client JSON** (`?format=singbox`, `?format=tachyon`, `?format=outbounds`): exported as `"type": "naive"`, `"username"`, `"password"`, `"quic": true` (when QUIC is enabled), `"tls": { "enabled": true, "server_name": "panel.example.com", ... }` (strictly without `insecure`).
- **Clash Meta YAML** and **SIP008**: **not exported** — use `?format=singbox`, `?format=tachyon`, or URI links.
- **Compatible Clients**: `tachyon-core` (including OpenWrt routers) and upstream Chromium-based `naive` clients.

### Valid `tachyon-core` JSON Example

```json
{
  "inbounds": [
    {
      "type": "naive",
      "tag": "naive-in",
      "listen": "::",
      "listen_port": 8450,
      "users": [
        {
          "username": "alice",
          "password": "example-naive-password"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "panel.example.com",
        "alpn": ["h2", "http/1.1"],
        "certificate_path": "data/certs/server.crt",
        "key_path": "data/certs/server.key"
      }
    }
  ],
  "outbounds": [
    {
      "type": "naive",
      "tag": "naive-out",
      "server": "panel.example.com",
      "server_port": 8450,
      "username": "alice",
      "password": "example-naive-password",
      "tls": {
        "enabled": true,
        "server_name": "panel.example.com",
        "alpn": ["h2", "http/1.1"]
      }
    }
  ]
}
```

---

## Diagnostics & Troubleshooting

| Symptom | Root Cause | Resolution |
|---|---|---|
| `tachyon-core` validation error: `insecure is not supported on naive outbound` | Manual config or imported profile contains `tls.insecure: true` or `tls.disable_sni: true` | Remove `insecure` and `disable_sni` from the `naive` outbound `tls` block and configure a valid CA certificate. |
| Client aborts with certificate error (`ERR_CERT_AUTHORITY_INVALID` / `x509`) | Server uses a self-signed or expired certificate, or `sni` does not match the certificate SAN | Issue a trusted Let's Encrypt / ZeroSSL certificate for your domain, verify `data/certs/server.crt` contains the full chain, and set `tls.server_name` to that domain. |
| Client fails to connect with `HTTP/3 QUIC` enabled | `UDP` is blocked on the chosen port (e.g., `8459`), or the ISP drops QUIC/UDP | Open UDP on the inbound port in your VPS firewall, or switch to the **NaiveProxy HTTPS/2** (TCP) preset. |
| `clash` subscription omits the `naive` node | Clash Meta YAML export does not support `naive` | Fetch the subscription using `?format=singbox`, `?format=tachyon`, or `naive+https://` URIs. |

---

## See Also

- [Protocol Catalog](./README.md)
- [Complete End-to-End Guide](../00-complete-guide.md)
- [Protocol Encyclopedia](../09-protocols-encyclopedia.md)
- [UI Every Tab & Button Guide](../11-ui-every-tab-and-button-guide.md)
- [09. ShadowTLS v3](./09-shadowtls.md)
- [11. MASQUE (HTTP/3 CONNECT-IP)](./11-masque.md)
- [12. TrustTunnel](./12-trusttunnel.md)
