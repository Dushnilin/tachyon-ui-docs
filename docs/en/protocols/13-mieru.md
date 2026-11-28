# 13. Mieru v3

> **Summary Card**
> - **Role in `tachyon-core`**: `Inbound` (server) and `Outbound` (client) — **Mieru v3** protocol
> - **L4 Transport**: `TCP` or `UDP` (plain string field `transport`: `"TCP"` or `"UDP"`). A single `Inbound` in `tachyon-core` supports binding up to **256 ports** (`listen_ports`, `server_ports`, `port_bindings`)
> - **TLS / REALITY / ACME**: no external TLS is used (`tlsModes: []`); stream encryption and replay protection are built directly into the Mieru protocol
> - **Domain Required**: no — operates directly over an IP address (`198.51.100.10`) without domains or certificates

---

## Purpose & How It Works in `tachyon-core`

**Simple Analogy.** If `Shadowsocks` is a minimalist encrypted pipe ("one request, one stream"), **Mieru v3** is a multi-lane armored conveyor belt with tunable stream aggregation. It does not rely on TLS handshakes (meaning no domain, no certificates, and zero TLS fingerprint), encrypts every byte with its own cryptographic layer, and dynamically controls how many underlying sockets are opened via **multiplexing levels** (`MULTIPLEXING_HIGH` .. `MULTIPLEXING_OFF`) over either `TCP` or `UDP`.

**Engine Mechanics in `tachyon-core`:**
- Unlike VLESS/VMess/Trojan where `transport` is an object configuring `ws`/`grpc`/`xhttp`, in **Mieru** the **`transport` field is a plain uppercase string `"TCP"` or `"UDP"`**. Passing a JSON object instead of a string triggers a config parse error.
- Server-side multiplexing intensity is controlled by the `multiplexing` enum string:
  - `MULTIPLEXING_HIGH` — maximum aggregation: many user streams share minimal underlying connections (reduces handshake overhead and socket churn).
  - `MULTIPLEXING_MIDDLE` — balanced aggregation (optimal for `UDP`).
  - `MULTIPLEXING_LOW` — light aggregation.
  - `MULTIPLEXING_OFF` — disables multiplexing (one dedicated transport channel per connection).
- In `tachyon-core`, a Mieru v3 inbound can bind across a range or list of up to 256 ports (`listen_ports` / `server_ports` / `port_bindings`) and transports both TCP streams and UDP datagrams internally over either `TCP` or `UDP`.
- Authentication is mandatory on both sides: `tachyon-ui` generates `users[].{name, password}` on `Inbound` and `username` + `password` on `Outbound`.

---

## Panel Form Screenshot

![Mieru v3 configuration form in tachyon-ui](../../images/protocols/en/proto-13-mieru.png)

---

## Step-by-Step Configuration in `tachyon-ui`

### Inbound Setup

1. Open **Inbounds** and click **Add Inbound**.
2. Select **Mieru (`mieru`)** as the protocol.
3. Choose one of the built-in panel presets (`PROTOCOL_PRESETS.mieru`):
   - **Mieru TCP (High Mux)** — port `8451` (TCP), transport `TCP`, multiplexing `MULTIPLEXING_HIGH`.
   - **Mieru UDP (Low Latency)** — port `8460` (UDP), transport `UDP`, multiplexing `MULTIPLEXING_MIDDLE`.
4. Adjust **Mieru Transport Protocol** (`inb-mieru-transport`: `TCP` or `UDP`) and **Multiplexing Level** (`inb-mieru-mux`) as needed.
5. Open the **Clients** tab and add users (`name` and `password`), ensuring both fields are non-empty.
6. Click **Save** and **Apply Changes**.

### Outbound Setup

1. Open **Outbounds** and choose type **`mieru`**.
2. Enter the server address (`198.51.100.10`) and port (`8451` or `8460`).
3. Fill in the username (`out-socks-user` -> `username`) and password (`out-socks-pass` -> `password`).
4. In **Advanced (raw JSON merge)** (`out-settings`), set `"transport": "TCP"` or `"transport": "UDP"` (matching the server exactly!), and optionally `"multiplexing"` and `"server_ports"`.

---

## Field, Toggle & Parameter Reference

### Inbound Node

| Panel Field (`ID`) | `tachyon-core` JSON Key | Allowed Values / Default | Purpose & Configuration Notes |
|---|---|---|---|
| **Mieru Transport Protocol** (`inb-mieru-transport`) | `transport` | `"TCP"` (default) \| `"UDP"` | Must be a plain string (not an object). Pick `"TCP"` if UDP is unstable or throttled on your ISP, or `"UDP"` for lower latency. |
| **Multiplexing Level** (`inb-mieru-mux`) | `multiplexing` | `"MULTIPLEXING_HIGH"` (default), `"MULTIPLEXING_MIDDLE"`, `"MULTIPLEXING_LOW"`, `"MULTIPLEXING_OFF"` | Balances socket count against multiplexing overhead. Use `HIGH` for TCP and `MIDDLE` or `LOW` for UDP. |
| **Clients** (Clients tab) | `users[].{name, password}` | Non-empty `name` and `password` | Authorized accounts. When zero active clients exist, the panel inserts `__disabled__`. |
| **Multi-Port Binding** (optional JSON) | `listen_ports`, `port_bindings` | Up to 256 ports | Supported by `tachyon-core` runtime when extending the config manually. |

### Outbound Node

| Panel Field (`ID`) | `tachyon-core` JSON Key | Allowed Values / Default | Purpose & Configuration Notes |
|---|---|---|---|
| **Server / Port** (`out-server`, `out-port`) | `server`, `server_port` | IP/domain and port (`8451` / `8460`) | Remote Mieru server address and port. |
| **Username / Password** (`out-socks-user`, `out-socks-pass`) | `username`, `password` | Non-empty strings | Must match an entry in the server's `users[]` (`username` corresponds to `name` on the inbound). |
| **Transport / Multiplexing** (`out-settings`) | `transport`, `multiplexing`, `server_ports` | `"TCP"` \| `"UDP"`, mux level | Client `transport` must match the server's `transport`. |

---

## Networking, TLS & Firewall

- **Firewall Ports (`inbound_firewall_rules`)**:
  - When `mieru_transport == "UDP"`, opens **`UDP`** on `inb.port` (preset `8460`).
  - Otherwise (`"TCP"`), opens **`TCP`** on `inb.port` (preset `8451`).
- **TLS & Reverse Proxy**: Mieru uses its own binary cryptographic transport without outer TLS. Because of this, it **cannot** run behind an HTTP/TLS-terminating reverse proxy (NGINX/Caddy) — traffic must reach the server port directly (or via pure L4 port forwarding).

---

## Subscription Export, Clients & `tachyon-core` JSON Example

- **URI** (`?format=raw` or Base64):
  `mieru://alice:password@198.51.100.10:8451?transport=TCP#Mieru-TCP`
  *(or `?transport=UDP` for UDP inbounds).*
- **Client JSON** (`?format=singbox`, `?format=tachyon`, `?format=outbounds`): exported as `"type": "mieru"`, `"username"`, `"password"`, `"transport": "TCP"` (or `"UDP"`).
- **Clash Meta YAML** (`?format=clash` / `mihomo`): exported with `"type": mieru`, `"username"`, `"password"`, `"transport"` (`TCP`/`UDP`), `"udp": true`.
- **SIP008**: not supported.
- **Compatible Clients**: `tachyon-core` (including OpenWrt routers), `Mihomo / Clash Meta` clients, and native Mieru v3 clients.

### Valid `tachyon-core` JSON Example

```json
{
  "inbounds": [
    {
      "type": "mieru",
      "tag": "mieru-in",
      "listen": "::",
      "listen_port": 8451,
      "transport": "TCP",
      "multiplexing": "MULTIPLEXING_HIGH",
      "users": [
        {
          "name": "alice",
          "password": "example-mieru-password"
        }
      ]
    }
  ],
  "outbounds": [
    {
      "type": "mieru",
      "tag": "mieru-out",
      "server": "198.51.100.10",
      "server_port": 8451,
      "username": "alice",
      "password": "example-mieru-password",
      "transport": "TCP",
      "multiplexing": "MULTIPLEXING_HIGH"
    }
  ]
}
```

---

## Diagnostics & Troubleshooting

| Symptom | Root Cause | Resolution |
|---|---|---|
| Config parse/validation error: expected string for `transport` | Manual JSON passed an object like `"transport": { "type": "tcp" }` in VLESS style | Replace it with an uppercase string: `"transport": "TCP"` or `"transport": "UDP"`. |
| Client fails to connect or packets time out silently | Server uses `UDP` while client connects over `TCP` (or vice versa), or the wrong L4 protocol is open in the firewall | Match `transport` between server and client/subscription and verify UDP is open for `UDP` mode or TCP for `TCP` mode. |
| Authentication error on client startup | Empty `username` or `password` on `Outbound`, or user disabled on server | Ensure both `username` and `password` are set and the account is active in `users[]`. |
| Packet loss or micro-stutters in games under `UDP` + `MULTIPLEXING_HIGH` | Too many datagram streams packed onto a single transport channel | Lower the multiplexing level to `MULTIPLEXING_MIDDLE` or `MULTIPLEXING_LOW`. |

---

## See Also

- [Protocol Catalog](./README.md)
- [Complete End-to-End Guide](../00-complete-guide.md)
- [Protocol Encyclopedia](../09-protocols-encyclopedia.md)
- [UI Every Tab & Button Guide](../11-ui-every-tab-and-button-guide.md)
- [12. TrustTunnel](./12-trusttunnel.md)
- [14. Snell v4/v5/v6](./14-snell.md)
