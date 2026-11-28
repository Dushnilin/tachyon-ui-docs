# 19. OpenConnect (AnyConnect / GlobalProtect / FortiGate / Pulse)

| Parameter | Value |
| :--- | :--- |
| **Role in `tachyon-ui`** | `Inbound` + `Outbound` (`inbounds[]` and `outbounds[]` are fully functional; `endpoints[]` is supported for `check` only) |
| **L4 Transport** | `TCP` (TLS / CSTP) + `UDP` (DTLS for `anyconnect` outbound; ESP when `ip: true`) |
| **TLS / REALITY / ACME** | Certificate **`tls` is strictly mandatory** (REALITY is not supported) |
| **Status in `tachyon-core 0.6.9`** | Full Inbound and Outbound support in schema (`check`) and runtime (`run`) |

---

## Purpose and How It Works in `tachyon-core`

**OpenConnect** is an open implementation of enterprise SSL VPN protocols compatible with Cisco AnyConnect, Palo Alto GlobalProtect, Fortinet FortiGate, Juniper Pulse Secure, and related clients. To external observers and transit gateways, the session appears as a legitimate corporate SSL VPN over HTTPS.

How the protocol operates in `tachyon-core 0.6.9`:

1. The client connects over standard `HTTPS` (`TLS`) on the configured port, negotiating a compatible dialect (`flavor`): `anyconnect`, `gp`, `pulse`, or `fortinet` (the core also accepts `nc` and `f5`).
2. After verifying credentials and the session `cookie`, the server establishes an L3 tunnel via the `smoltcp` userspace network stack (`tachyon-ui` always sets `"ip": true` on inbound).
3. IP packets travel inside the TLS stream (`CSTP`) or — when UDP is reachable — over an accelerated `DTLS` channel (for `anyconnect`) or `ESP` (for compatible dialects with `ip: true`).

---

## Panel Form Screenshot

![OpenConnect protocol configuration form in tachyon-ui](../../images/protocols/en/proto-19-openconnect.png)

---

## Step-by-Step Setup in `tachyon-ui`

### Configuring an Inbound

1. Open **Inbounds → Add** and select the **`openconnect`** protocol.
2. Pick one of the panel's built-in presets:
   - **`Cisco AnyConnect SSL VPN`** — port `8468`, TLS, SNI `www.microsoft.com`, `flavor: anyconnect`, `mtu: 1400`.
   - **`Palo Alto GlobalProtect`** — port `8468`, TLS, SNI `www.cloudflare.com`, `flavor: gp`, `mtu: 1400`.
   - **`Fortinet FortiGate`** — port `8468`, TLS, SNI `www.microsoft.com`, `flavor: fortinet`, `mtu: 1400`.
3. In the protocol parameters, select the **Dialect (`flavor`)** (`anyconnect`, `gp`, `fortinet`, `pulse`) and **MTU** (`1200..1480`, default `1400`).
4. On the **TLS** tab, verify the `SNI` domain (`panel.example.com`) and ensure valid `certificate_path` and `key_path` files are configured (for example, issued via ACME by `tachyon-ui`).
5. Add a client (**Name** and **Password**): the panel automatically uses the first active client's password as the tunnel authentication `cookie`. Save the inbound.

### Configuring an Outbound

1. Go to **Outbounds → Add** and select **`openconnect`**.
2. Specify **Server**, **Port**, **Password**, and **TLS** settings (`tls.enabled: true` is mandatory).
3. In additional JSON settings (`out-settings`), optionally set `flavor`, `username`, `cookie`, `ip: true`, `dtls`, `esp`, and `server_cert_pin`. Note: `flavor: "pulse"` on outbound requires both `username` and `password`, and enabling `esp: true` requires `ip: true`.

---

## Field, Toggle, and Parameter Reference

### Inbound

| Panel Field | `tachyon-core` JSON Key | Allowed Values / Default | Purpose |
| :--- | :--- | :--- | :--- |
| **Listen Address** (`inb-listen`) | `listen` | IPv4/IPv6; default `0.0.0.0` (or `::`) | Network interface listening for HTTPS/DTLS |
| **Port** (`inb-port`) | `listen_port` | `1..65535`; preset `8468` (often `443` behind an external front) | Inbound OpenConnect port (TCP and UDP) |
| **Dialect (`flavor`)** (`inb-oc-flavor`) | `flavor` | `anyconnect`, `gp`, `fortinet`, `pulse` (core also accepts `nc`, `f5`) | Compatibility profile for enterprise SSL VPN clients |
| **MTU** (`inb-oc-mtu`) | `mtu` | `1200..1480`; default `1400` | Virtual L3 interface MTU |
| **L3 IP Mode** *(auto)* | `ip` | `true` (set automatically by panel) | Enables full IP packet routing through `smoltcp` |
| **Cookie / Client** (`clients[]`) | `cookie` | String (derived from first active client's password) | Authentication token/cookie for incoming sessions |
| **TLS Tab** (`inb-tls-*`) | `tls` | `tls` only (`enabled: true` mandatory) | Server certificate, private key, SNI, and TLS options |

### Outbound

| Panel Field | `tachyon-core` JSON Key | Allowed Values / Default | Purpose |
| :--- | :--- | :--- | :--- |
| **Server** (`out-server`) | `server` | IP or domain (`198.51.100.10`, `panel.example.com`) | Remote SSL VPN gateway address |
| **Port** (`out-port`) | `server_port` | `1..65535` | Remote SSL VPN gateway port |
| **Password** (`out-password`) | `password` | String (mandatory alongside `username` for `pulse`) | Account password |
| **Dialect** *(in `out-settings`)* | `flavor` | `anyconnect`, `gp`, `nc`, `pulse`, `f5`, `fortinet` | Remote gateway protocol dialect |
| **Username & Cookie** *(in `out-settings`)* | `username`, `cookie` | Strings | Login username and pre-obtained session cookie |
| **IP / DTLS / ESP** *(in `out-settings`)* | `ip`, `dtls`, `esp` | `bool` (`esp: true` requires `ip: true`; `dtls` for `anyconnect`) | Controls L3 IP mode and UDP transport |
| **Certificate Pinning** *(in `out-settings`)* | `server_cert_pin` | String (server certificate pin hash) | Pins the remote gateway certificate fingerprint |
| **TLS Tab** (`out-tls*`) | `tls` | `{ "enabled": true, ... }` (strictly mandatory) | Client TLS settings (`server_name`, `insecure`) |

---

## Networking, TLS, and Firewall

- **TLS / REALITY / ACME:** Both server and client TLS are **strictly mandatory**; REALITY is not used. For standard third-party clients (Cisco AnyConnect, OpenConnect, NetworkManager) to connect without certificate warnings, use a real domain (`panel.example.com`) with an ACME certificate managed by `tachyon-ui`. Built-in ACME is not started inside `tachyon-core` — the panel updates `certificate_path`/`key_path` directly.
- **Firewall:** `tachyon-ui` automatically opens both **`TCP` and `UDP`** on `listen_port` (for example, `8468/tcp` and `8468/udp`) to support both TLS/CSTP and accelerated UDP channels (DTLS/ESP).

---

## Subscription Export, Compatible Clients, and Valid `tachyon-core` JSON

- **Subscription Export Formats (`GET /sub/{sub_id}?format=...`):**
  - **URI (`raw` / Base64):** `openconnect://alice:secret-pass@panel.example.com:8468?flavor=anyconnect&ip=1#OpenConnect`
  - **Client JSON (`?format=singbox` / `tachyon` / `outbounds`):** generates `type: "openconnect"` with `flavor`, `username`, `password`, `cookie`, `ip: true`, and `tls`.
  - **Clash Meta YAML and SIP008:** not exported.
- **Compatible Clients:** `tachyon-core`, OpenConnect (Linux / OpenWrt / NetworkManager / macOS / Windows), compatible AnyConnect / GlobalProtect / FortiClient apps.

### Valid `tachyon-core` JSON Example

#### Inbound (`inbounds[]`)

```json
{
  "type": "openconnect",
  "tag": "openconnect-in-8468",
  "listen": "::",
  "listen_port": 8468,
  "flavor": "anyconnect",
  "mtu": 1400,
  "ip": true,
  "cookie": "oc-client-secret-pass",
  "tls": {
    "enabled": true,
    "server_name": "panel.example.com",
    "certificate_path": "data/certs/server.crt",
    "key_path": "data/certs/server.key"
  }
}
```

#### Outbound (`outbounds[]`)

```json
{
  "type": "openconnect",
  "tag": "openconnect-out",
  "server": "panel.example.com",
  "server_port": 8468,
  "flavor": "anyconnect",
  "username": "alice",
  "password": "oc-client-secret-pass",
  "cookie": "oc-client-secret-pass",
  "ip": true,
  "tls": {
    "enabled": true,
    "server_name": "panel.example.com"
  }
}
```

---

## Diagnostics and Common Errors

| Symptom / Error | Cause | Resolution |
| :--- | :--- | :--- |
| Outbound `mtu` validation error in manual JSON | The strict outbound schema validator expects `mtu` as a string or omitted | Omit `mtu` on outbound (uses the default) or pass it as a string when editing manually |
| Outbound startup fails with `esp: true` | `esp: true` was set while `ip` is missing or `false` | Always set `"ip": true` alongside `"esp": true` |
| Authentication failure in `pulse` mode | Only `password` or only `username` was provided | Provide both `username` and `password` on `flavor: "pulse"` outbounds |
| Enterprise client drops immediately after TCP connect | Untrusted self-signed certificate or `SNI` mismatch | Configure a domain (`panel.example.com`) with an ACME certificate or pass the matching `server_cert_pin` |

---

## See Also

- [Complete Guide](../00-complete-guide.md)
- [Protocols Encyclopedia](../09-protocols-encyclopedia.md)
- [UI Every Tab and Button Guide](../11-ui-every-tab-and-button-guide.md)
- [Protocol Catalog](./README.md)
- [18. Sudoku](./18-sudoku.md)
- [20. Cloudflared](./20-cloudflared.md)
