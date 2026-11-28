# 26. SSH, Tor, and Bridge

| Parameter | Value |
| :--- | :--- |
| **Role in `tachyon-ui`** | Outbound only (`#outbounds`: types `ssh`, `tor`, `bridge`) |
| **L4 Transport** | `ssh`: TCP (default `22`); `tor`: local Tor child process; `bridge`: Linux interface/table binding (`iproute2`) |
| **TLS / REALITY** | Not used by any of the three types (`bridge` also forbids `dialer` fields) |
| **Status in `tachyon-core 0.6.9`** | **Pass `tachyon-core check` schema validation, but runtime is not implemented in `tachyon-core 0.6.9`** (startup fails at initialization) |

---

## 1. Purpose and How They Work in `tachyon-core`

`tachyon-ui` provides three specialized system outbound types for dedicated routing scenarios:

1. **`ssh`** — Tunnels outbound traffic through any standard Linux/BSD SSH server (password or PEM private-key authentication with `host_key[]` verification).
2. **`tor`** — Spawns and routes selected domains (including `.onion`) through a local Tor binary (`executable_path`, `data_directory`, `extra_args`).
3. **`bridge`** — Egresses traffic directly onto a specific Linux physical/virtual network `interface`, `bridge_name`, or `iproute2` routing table (`iproute2_table_index`, `iproute2_rule_index`).

> [!WARNING]
> **Actual Status in `tachyon-core 0.6.9`**:
> All three types (`ssh`, `tor`, `bridge`) are supported by `tachyon-ui` forms and pass JSON schema validation in `tachyon-core check`, but **their runtime handlers are not implemented in `tachyon-core 0.6.9`**. Enabling any of them in an active core configuration will cause initialization to fail.
>
> Use these types only for schema preparation/validation or importing `ssh://` links for future use. For working inter-node links, use `tewfai`, `vless`, `wireguard`, or `amneziawg`; for interface binding in a running core, use OS routing or `dialer.bind_interface` on supported outbounds.

---

## 2. Panel Form Screenshots

### SSH Outbound

![SSH outbound configuration form in tachyon-ui](../../images/protocols/en/proto-26-ssh-out.png)

### Tor Outbound

![Tor outbound configuration form in tachyon-ui](../../images/protocols/en/proto-26-tor-out.png)

### Bridge Outbound

![Bridge outbound configuration form in tachyon-ui](../../images/protocols/en/proto-26-bridge-out.png)

---

## 3. Step-by-Step Setup in `tachyon-ui`

There are no built-in presets in the panel for `ssh`, `tor`, or `bridge`.

### Configuring `ssh`

1. Open **Outbounds (`#outbounds`)** → **+ Add Outbound** and choose type **`ssh`**.
2. Enter `Server Address` (e.g., `198.51.100.10`) and `Server Port` (`22`).
3. Set `SSH User` (`root` or a dedicated service user), and provide either a password **or** a `PEM Private Key` (with optional `Key Passphrase`).
4. Optionally specify `Client Version` (e.g., `SSH-2.0-OpenSSH_9.6`) and trusted server `Host Keys`.

### Configuring `tor`

1. Select type **`tor`**. `server` and `server_port` are stripped automatically on save and config generation (Tor runs as a local process).
2. Set `Binary Path` (e.g., `/usr/bin/tor`), `Data Directory` (`data/tor`), and optional extra CLI arguments (`--quiet`, etc.).

### Configuring `bridge`

1. Select type **`bridge`**. As with `tor`, `server`, `server_port`, and any shared `dialer` fields are automatically stripped by the generator.
2. Specify the network interface (`interface`, e.g., `eth0`), bridge name (`bridge_name`, e.g., `br-lan`), and optional `iproute2_table_index` / `iproute2_rule_index` values.

---

## 4. Field, Toggle, and Parameter Reference

### `ssh` Parameters

| Panel Field | `tachyon-core` JSON Key | Allowed Values / Default | Purpose |
| :--- | :--- | :--- | :--- |
| **Server Address** | `server` | IP or domain (`198.51.100.10`) | SSH server host. |
| **Server Port** | `server_port` | `1..65535` (placeholder `22`) | SSH TCP port. |
| **SSH User (`user`)** | `user` | String (`username` alias is auto-converted to `user`) | SSH account username. |
| **Password (`password`)** | `password` | String (optional; omitted when empty) | SSH password authentication. |
| **PEM Private Key** | `private_key` | Multi-line PEM key (`-----BEGIN OPENSSH PRIVATE KEY-----`) | Private key for passwordless SSH authentication. |
| **Key Passphrase** | `private_key_passphrase` | String (optional) | Passphrase for an encrypted PEM private key. |
| **Client Version** | `client_version` | String (e.g., `SSH-2.0-OpenSSH_9.6`) | SSH client identification banner. |
| **Host Keys** | `host_key` | String array (`["ssh-ed25519 AAAA..."]`) | Pinned server host public keys to prevent MITM attacks. |

### `tor` Parameters

| Panel Field | `tachyon-core` JSON Key | Allowed Values / Default | Purpose |
| :--- | :--- | :--- | :--- |
| **Binary Path** | `executable_path` | File path (e.g., `/usr/bin/tor`) | Path to the system Tor executable. |
| **Data Directory** | `data_directory` | Directory path (e.g., `data/tor`) | Tor circuit state and cache directory. |
| **Extra Arguments** | `extra_args` | String array (`["--quiet"]`) | Additional CLI flags passed to the Tor process. |

### `bridge` Parameters

| Panel Field | `tachyon-core` JSON Key | Allowed Values / Default | Purpose |
| :--- | :--- | :--- | :--- |
| **Network Interface** | `interface` | Interface name (`eth0`) | Linux egress network interface. |
| **Bridge Name** | `bridge_name` | Bridge name (`br-lan`) | Logical Linux bridge to bind traffic to. |
| **iproute2_table_index** | `iproute2_table_index` | Integer (e.g., `200`) | `iproute2` routing table index. |
| **iproute2_rule_index** | `iproute2_rule_index` | Integer (e.g., `200`) | `ip rule` priority/index. |

---

## 5. Networking and Firewall

- None of these three outbound types open inbound server ports or generate `inbound_firewall_rules`.
- During config generation, `tachyon-ui` strips empty string fields on `ssh`/`tor`/`bridge`, and for `bridge` strictly retains only the 6-field whitelist (`type`, `tag`, `interface`, `bridge_name`, `iproute2_table_index`, `iproute2_rule_index`)—without `server`, `server_port`, or `dialer` parameters.

---

## 6. Subscription Export, Clients, and Valid `tachyon-core` JSON Example

- **Inbound Subscription Parser**: Supports parsing `ssh://user:pass@host:port#label` links on import into `#outbounds`.
- **Client Subscription Export**: Because runtime execution is not implemented in `0.6.9`, do not include these types in active client subscriptions.

### JSON Schema Examples for `tachyon-core check`

```json
{
  "outbounds": [
    {
      "type": "ssh",
      "tag": "ssh-out",
      "server": "198.51.100.10",
      "server_port": 22,
      "user": "tunnel",
      "private_key": "-----BEGIN OPENSSH PRIVATE KEY-----\n...\n-----END OPENSSH PRIVATE KEY-----",
      "host_key": ["ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIExampleHostKey"]
    },
    {
      "type": "tor",
      "tag": "tor-out",
      "executable_path": "/usr/bin/tor",
      "data_directory": "data/tor",
      "extra_args": ["--quiet"]
    },
    {
      "type": "bridge",
      "tag": "bridge-out",
      "interface": "eth0",
      "bridge_name": "br-lan",
      "iproute2_table_index": 200,
      "iproute2_rule_index": 200
    }
  ]
}
```

---

## 7. Diagnostics and Common Errors

- **Core fails to start with `ssh`, `tor`, or `bridge`**: These types are not implemented in the `tachyon-core 0.6.9` runtime (only `check` schema validation is active). Disable or remove them from active configs.
- **`unknown field 'server'` or `'connect_timeout'` when hand-editing `tor` or `bridge`**: The validator enforces `reject_unknown_fields` and rejects `server` and `dialer` fields on `tor` and `bridge`.
- **Using `username` instead of `user` in `ssh`**: The `tachyon-core` schema requires `user` (`tachyon-ui` automatically converts `username` → `user`).

---

## 8. See Also

- [Protocol Catalog](./README.md)
- [Complete Guide](../00-complete-guide.md)
- [Tachyon Protocol Encyclopedia](../09-protocols-encyclopedia.md)
- [UI Every Tab & Button Guide](../11-ui-every-tab-and-button-guide.md)
- [25. Hysteria v1](./25-hysteria1.md)
- [27. Balancers and Chain (urltest / selector / chain)](./27-balancers-and-chain.md)
