# 6. Dashboard, Core Diagnostics, Themes, Snapshots, Privacy Mask (`Alt+P`) & REST API

[Back to Index](../../README.md) | [Chapter 5 — Protocols, Clusters & Bot](05-protocols-nodes-and-tgbot.md) | [UI Tab & Button Reference](11-ui-every-tab-and-button-guide.md) | [Русский](../ru/06-diagnostics-privacy-and-api.md) | [Next: Chapter 7 — Full Setup & Cluster Guide](07-full-setup-and-cluster-guide.md)

---

## Dashboard Overview (`#dashboard`)

![Tachyon-UI Dashboard](../images/en/02-dashboard.png)

The **Dashboard** gives a single-screen real-time summary of your node and `tachyon-core` runtime:

| Control / Block | Label / Trigger | Method / Endpoint | Purpose & Behavior |
| :--- | :--- | :--- | :--- |
| **Apply Config** | `Apply` (`I.check`) | `POST /api/core/apply` | Generates the full `tachyon-core` JSON config and performs hot apply/restart. |
| **Engine Lifecycle** | `Start`, `Stop`, `Restart Core` | `POST /api/core/start \| stop \| restart` | Controls the `tachyon-core` daemon; `Core` indicator in topbar opens `#modal-core-update`. |
| **Config Preview** | `Config Preview` (`I.code`) | `GET /api/config-preview` | Opens `#modal-config` showing the exact generated `tachyon-core` JSON. |
| **Quick Utilities** | Diagnostics, SNI Scan, Unlock Probe, RKN Check, Snapshots | `#modal-core-diag`, `#modal-sni-scanner`, `#modal-unlock-probe`, `#modal-snapshots` | 1-click launch of diagnostics, SNI health checks, speedtests, and rollback checkpoints. |
| **6 KPI Cards & `Trim RAM`** | Core status/uptime, Inbounds/clients, Outbounds/healthy, Subs, Up/Down traffic, RAM + `Trim RAM` | `GET /api/status`, `GET /api/core/memory`, `POST /api/core/memory-trim` | Real-time node overview; `Trim RAM` triggers memory buffer reclamation on low-RAM VPSs. |
| **Subsystem Badges & Protocol Matrix** | Core, TLS/ACME, SmartDNS, TG Bot, Anti-Share, Auto-Healer, Failover, Watchdog | Derived from `/api/status` | Shows active subsystems and listening protocol badges at a glance. |
| **2-Button Route & DNS Tracer** | Target input + `Check Route` + `Check DNS` | `POST /api/core/route-check`, `POST /api/core/dns-check` | Verifies through live `tachyon-core` which outbound handles a domain/IP and how DNS resolves it. |
| **Throughput Chart, `.trs` Card & Top Clients** | Live Up/Down curve, sysinfo bar, `.trs` pack card, active client table (`Kick`) | `POST /api/core/kick` | Visualizes live bandwidth and lets you force-disconnect a client's active sockets. |
| **Core Logs (`#dash-logs`)** | Level/search filter + `Copy Logs` | `GET /api/status` / `GET /api/logs` | Streams live `tachyon-core` stdout/stderr for handshake, failover, and startup inspection. |

---

## Live Connections Dock (`` Ctrl+` ``)

![Live Connections Dock](../images/en/03-live-connections-dock.png)

For continuous traffic inspection without leaving your active table or editor, press `` Ctrl+` `` (or open **Tools → Live Connections** in the topbar). It streams live `tachyon-core` connections in a bottom slide-up drawer (`#gpui-conn-dock`) with:

| Dock Control | Endpoint / Action | Behavior |
| :--- | :--- | :--- |
| **Search Filter** | Client-side 90 ms debounced filter | Filters active sockets by domain/SNI, destination IP, source IP, inbound, outbound, or rule. |
| **Pause / Resume Auto-Refresh** | `GET /api/core/connections` | Freezes the live connection table for inspection or resumes polling. |
| **Close One / Close All** | `POST /api/core/connections/close` | Terminates a single socket (`{ id }`) or closes all active sessions (`id: ""`). |
| **Block Domain** | `POST /api/core/block-domain` | Adds the domain to a `block` rule and applies it live in `tachyon-core`. |
| **Route Domain** | `POST /api/core/route-domain` | Pinpoints a domain to a chosen outbound or balancer on the fly. |

---

## Unlock Probe & Speedtest (`#modal-unlock-probe`)

Open **Tools → Network Probe & Speedtest** (or launch from the Dashboard) to audit streaming/AI service reachability (Netflix, YouTube, ChatGPT/OpenAI, etc.), regional block status (**RKN Check** via `POST /api/tools/rkn-check`), and server bandwidth (**Include Speedtest** checkbox → `POST /api/tools/unlock-probe`):

![Unlock Probe and Speedtest Modal](../images/ui/en/ui-03-unlock-speedtest-modal.png)

---

## Live 5-Tab Core Diagnostics Console

Click **Connections** in the topbar or **Live Connection Matrix** on the dashboard to open the real-time diagnostic suite backed by `tachyon-core` Runtime APIs:

![Tachyon Core Live Diagnostics Console](../images/en/32-core-diagnostics.png)

| Console Tab | Capabilities & Core Endpoints |
| :--- | :--- |
| **1. Connections & Route** | Live route tracer (`POST /api/core/route-check`), quick domain trace (`POST /api/core/route-domain`), active session filtering by user/domain/IP/protocol (`GET /api/core/connections`), single-connection kill or user kick (`POST /api/core/connections/close`, `POST /api/core/kick`), instant domain block (`POST /api/core/block-domain`), and core memory telemetry & trim (`GET /api/core/memory`, `POST /api/core/memory-trim`). |
| **2. Health & Anomalies** | Engine telemetry (`GET /api/core/health`: RSS memory, pressure level, file descriptors, NAT evictions, DNS cache hit/miss, FakeIP stats, per-outbound RTT and circuit-breaker states) and runtime anomaly feed (`GET /api/core/anomalies`). |
| **3. Failover & Proxies** | Failover group state (`GET /api/core/failover`, `POST /api/core/failover/restore`), live proxy groups (`GET /api/core/proxies`) with real-time `selector` switching, batch/group latency probes (`POST /api/core/proxies/delay`), and circuit-breaker resets (`POST /api/core/outbound/reset`). |
| **4. DNS & Leak Audit** | Side-by-side DNS check (`POST /api/core/dns-check`: engine resolver vs direct UDP:53 for spoofing/bogon checks), DNS cache flush (`POST /api/core/dns/flush`), and IPv4/IPv6/WebRTC/DNS leak audit (`GET / POST /api/core/leak`). |
| **5. Test & In-Flight** | 8-stage active connection test (DNS → Rule → Outbound → TCP → TLS → HTTP via `POST /api/core/connection-test`) and in-flight request ring inspector (`GET / DELETE /api/core/flight`). |

---

## Automated Healing, Failover & Watchdog Tools

`tachyon-ui` includes background resilience automation configurable across **Settings**, **Inbounds**, and **Nodes**:
- **SNI Scanner (`POST /api/tools/sni-scan`):** Probes candidate domains for TLS 1.3 + ALPN `h2` compatibility to select healthy REALITY targets.
- **Auto-Healer SNI (`auto_heal_enabled`, `POST /api/tools/auto-heal`):** Periodically verifies active REALITY SNI targets and automatically rotates degraded hosts without manual intervention.
- **Cloudflare DNS Failover (`POST /api/tools/cloudflare-dns`):** Monitors primary node reachability and updates Cloudflare DNS A records (`cf_record_name`) across `cf_backup_ips` automatically when a primary node fails.
- **Zero-Traffic Watchdog:** Watches expected active sessions and sends instant admin alerts (and optional automated recovery actions) if inbound traffic suddenly drops to zero.
- **Smart Relay Failover & Failback (`relay_auto_failover`):** Monitors relay outbounds in multi-node setups, automatically rerouting to healthy same-country backups on outage and failing back once the primary relay recovers.

---

## 16 Built-In Themes, Theme Studio & Firewall Settings (`#settings`)

![Settings — Firewall Automation & 16 Themes](../images/en/30-settings-firewall-themes.png)

In **Settings (`#settings`)** (and via the topbar theme menu or Command Palette `Ctrl+K`), you can customize the interface appearance alongside firewall and panel security controls:
- **16 Built-in Themes:** **13 dark themes** (`Default Dark`, `Legacy Dark`, `Catppuccin`, `Dracula`, `Everforest`, `Gruvbox Material`, `Monokai`, `Nord`, `One Dark Pro`, `Tokyo Night`, `Solarized Dark`, `Vesper`, `Material Dark`) and **3 light themes** (`Material Light`, `Apple Light`, `Gruvbox Light`).
- **Custom Theme Studio (`+ Custom Theme` / `#theme-builder-overlay`):** Live color pickers for all 12 semantic CSS tokens (`--bg`, `--surface`, `--hover`, `--elevated`, `--accent`, `--accent-hover`, `--text`, `--text2`, `--text3`, `--success`, `--warn`, `--danger`), contrast scheme toggle (`dark` / `light`), live preview, and **JSON / JSONC import & export** (supports VS Code-style theme color keys and comments, persisted in `localStorage`).

![Custom Theme Studio Modal](../images/ui/en/ui-10-theme-editor-modal.png)

- **Automated Firewall Reconciliation (`auto_firewall_enabled`):** Manages `tachyon-ui:managed` rules across `UFW`, `firewalld`, and `iptables` while protecting SSH and custom operator rules.

---

## 10-Revision Configuration Snapshots & Rollback

![Configuration Snapshots & Rollback](../images/en/31-snapshots-rollback.png)

Before risky edits or large cluster changes, `tachyon-ui` maintains a bounded **10-revision ring buffer** of configuration snapshots (`data/snapshots.json`):
- **Automatic & Manual Checkpoints:** Create named manual checkpoints (`POST /api/snapshots`) or review automatic pre-change snapshots (`GET /api/snapshots`) showing deltas across inbounds, outbounds, route rules, rule-sets, and subscriptions (`subs_count`).
- **One-Click Atomic Rollback (`POST /api/snapshots/restore`):** Restores the selected snapshot and immediately re-applies listener bindings, ACME renewal config, firewall rules, and `tachyon-core` runtime state.

---

## Privacy Mask Mode (`Alt+P`)

![Privacy Mask Mode Alt+P](../images/en/33-privacy-mask-alt-p.png)

The **Hide Data** toggle in the topbar (also mapped to `Alt+P`, Command Palette `Ctrl+K`, and Settings; persisted in `localStorage`) protects sensitive user data during screen sharing and screenshots.

**Masked automatically (`***`):**
- Private server IPv4/IPv6 addresses and non-standard ports (`198.51.100.21:443` → `***.***.***.**:****`).
- Private server and node hostnames.
- Subscription URLs and tokens (`/sub/...`, `/cdn/v2/...`, `?k=...`, `?v=...`, `If-None-Match` and `X-Client-Digest` headers).
- UUIDs, X25519/WireGuard/SS2022 keys, passwords, HWIDs, and Telegram bot tokens/handles.
- QR codes (blurred visually).
- Sensitive `<input>` / `<textarea>` elements are masked visually with an SVG overlay **without altering `.value`**, so form saves and clipboard copies remain intact.

**Kept readable:**
- Well-known public test/probe domains and URLs (`https://www.gstatic.com/generate_204`, `https://cp.cloudflare.com/generate_204`, `example.com`, `youtube.com`, `google.com`, `cloudflare.com`, `github.com`, `microsoft.com`, `telegram.org`, etc.).
- Public DNS resolvers (`1.1.1.1`, `1.0.0.1`, `8.8.8.8`, `8.8.4.4`, `9.9.9.9`) and `0.0.0.0`.
- Diagnostic, route tracer, route simulator, and `urltest` probe input boxes.

---

## Command Palette (`Ctrl+K`) & Shortcuts

![Global Command Palette Ctrl+K](../images/en/34-command-palette.png)

![Keyboard Shortcuts Reference Modal](../images/ui/en/ui-05-shortcuts-modal.png)

| Shortcut | Action |
| :--- | :--- |
| **`Ctrl + K`** (or `Cmd + K`) | Global command palette & search across nodes, clients, subscriptions, themes, and instant actions (`Self-Audit`, `Hot-Reload`, 1-click invite, `tachyon-lists`). |
| **`` Ctrl + ` ``** (or `` Cmd + ` ``) | Toggle the bottom **Live Connections Dock** with 1-click domain blocking. |
| **`Alt + P`** | Toggle Privacy Mask Mode anywhere in the panel. |
| **`Alt + 1 … 9`** | Jump directly between all 9 tabs (`Dashboard`, `Inbounds`, `Outbounds`, `Subscriptions`, `Nodes`, `Bot & Billing`, `Routes`, `SmartDNS`, `Settings`). |
| **`Shift + Click`** | Range-select checkboxes in Inbounds, Outbounds, or Subscriptions to reveal the Floating Bulk Action Bar. |
| **`?`** | Open the built-in keyboard shortcuts reference modal (`#modal-shortcuts`). |
| **`Esc`** | Close mobile menu, tools/theme popover, inspector drawer, open modal, or Live Connections Dock in priority order. |

---

## Performance, HTTP Compression & Caching

- **Transparent `gzip` HTTP Compression:** `parse_accepts_gzip` evaluates `q`-weights in `Accept-Encoding` (including proper opt-out for `gzip;q=0`). For `200 OK` responses of `1024` bytes or larger (JSON APIs and subscriptions when `Content-Encoding` is not already set), `send_response` compresses the body with `GzEncoder` (`Compression::fast()`), sends the compressed payload when smaller than raw, and adds `Content-Encoding: gzip` and `Vary: Accept-Encoding`.
- **Pre-Compressed Embedded SPA `UI_HTML` Cache & Strict `ETag`:** `CachedUiAsset` (`UI_ASSET_CACHE`) holds both the `raw` and pre-compressed `gzip` variants of `ui.html` with `web_base_path` injected (~`225 KB` gzip vs ~`1.05 MB` raw, saving ~`78%` bandwidth). `ETag` (16-byte SHA-256) is matched against `If-None-Match` (including tag lists and weak `W/` tags) to return `304 Not Modified` with `Cache-Control: no-cache` and `Vary: Accept-Encoding`. `ETag` matching runs strictly **after** Covert Knock verification so subscription knocks carrying `If-None-Match` on the root path are not intercepted by the UI cache.
- **X.509 Certificate Caching:** `read_cert_info_cached` validates disk certificate metadata by `(mtime, size)`, while `read_cert_info_from_pem_cached` caches inline PEM parsing by SHA-256 digest (up to 64 entries, recalculating `days_left` on hit). During subscription export, `CertLookupCtx` resolves the default certificate path once per request instead of probing the filesystem per client/inbound.
- **Non-Blocking Read Endpoints:** `GET /api/status` avoids cloning `LiveTelemetry` as a whole by taking three short, non-overlapping lock reads (`live.read()` for counters and active connections, `data.read()` for aggregated summaries, and `live.read()` to serialize `&*live` directly). `GET /api/inbounds`, `/api/outbounds`, `/api/routes`, `/api/settings`, `/api/config-preview`, `/api/subs`, `/api/backup`, `/api/nodes`, `/api/tg-bot`, `/api/domain/proxy-snippet`, `/api/site/info` (`inspect_www_dir` runs outside the read lock), and `/api/sub-profiles` serialize payloads via `serde_json::to_vec` and release `RwLock` before socket writes via `send_json_bytes`.
- **Linear Client Resolution & SPA Optimizations:** `InboundUserNameResolver` disambiguates client identifiers in `O(C)` per inbound across config generation, subscription export, and telemetry matching, while `poll_stats` aggregates per-user deltas in a nested `user_deltas` map. In `ui.html`, tables use per-tick microtask `Map`/`Set` lookup caches (`_getInboundLookupMaps` cleared via `queueMicrotask`), `90 ms` search debouncing with instant `0 ms` reset on empty input, and `requestAnimationFrame` batching for the Privacy Mask `MutationObserver` with ancestor deduplication.

---

## REST API Summary

When `web_base_path` is set (e.g. `/p-8f3c21a9`), all panel and `/api/*` routes are strictly scoped under `<web_base_path>/api/...`.

### Authentication

| Method & Path | Description |
| :--- | :--- |
| `POST /api/login` | Authenticates `{ "username", "password" }`. Sets neutral `sid=...; HttpOnly; SameSite=Strict` cookie (`+ ; Secure` on TLS) and returns `{ "token", "expires" }` (accepted as `Authorization: Bearer <token>`). |
| `POST /api/logout` | Invalidates the active session token and clears `sid` and legacy `tachyon_session` cookies. |

### Endpoint Groups

| Group | Endpoints |
| :--- | :--- |
| **Status, Store & Snapshots** | `GET /api/status` (including `core.user_online_ips`, `core.sub_online_ips`, and `access_grant`), `GET /api/config-preview`, `GET/PUT /api/settings`, `GET/PUT /api/access-grant`, `GET /api/backup`, `POST /api/restore`, `GET/POST /api/snapshots` (10-revision ring), `POST /api/snapshots/restore`, `GET /api/logs` |
| **Engine & Panel Lifecycle** | `POST /api/core/start \| stop \| restart \| apply`, `GET /api/core/releases`, `POST /api/core/update`, `GET /api/panel/releases`, `POST /api/panel/self-update` |
| **Public Site (`SiteEngine`)** | `GET /api/site/info`, `POST /api/site/upload-zip` (raw ZIP or JSON `zip_base64`), `GET/PUT /api/site/index`, `DELETE /api/site/custom`, `GET /api/site/preview?mode=&path=&title=` |
| **Domain & TLS** | `GET /api/tls/status`, `POST /api/tls/self-signed`, `POST /api/tls/issue-acme`, `POST /api/domain/apply-profile` (including `unified_443_front`), `GET /api/domain/proxy-snippet` |
| **Self-Audit & Security** | `POST /api/web/self-audit` (returns `score`, `grade`, 9 `checks`), `GET /api/security/honeypot`, `POST /api/security/clear-honeypot` |
| **Inbounds, Outbounds & Routes** | `GET/POST /api/inbounds`, `PUT/DELETE /api/inbounds/:id`, `POST /api/inbounds/:id/clients`, `POST /api/inbounds/sync-clients` (accepts `clients`, `sub_configs`, `delete_sub_ids`, `inbound_ids`; returns `{ ok, added, updated, deleted, sub_configs_synced }`), `GET /api/inbounds/client-stats`, `POST /api/inbounds/batch-action` (`enable`, `disable`, `delete`, `set_country`), `POST /api/inbounds/traffic-policy`, `GET /api/share/:id`, `GET/POST /api/outbounds`, `PUT/DELETE /api/outbounds/:tag`, `POST /api/outbounds/health-check`, `POST /api/outbounds/batch-delete`, `GET/PUT /api/routes`, `POST /api/routes/tachyon-lists-pack`, `POST /api/routes/simulate` |
| **Generators & Network Tools** | `GET /api/generate/uuid \| reality-keys \| wg-keys \| ss-key \| password`, `POST /api/tools/sni-scan \| auto-heal \| port-hop \| unlock-probe \| cloudflare-dns` (plus regional reachability probes under `/api/tools/*`) |
| **Subscriptions, Covert Knock & Vouchers** | `GET/POST /api/subs` (supports `old_sub_id` for in-place rename with `409 Conflict` guard and `total_limit_bytes`), `DELETE /api/subs/:id`, `POST /api/subs/:id/reset-traffic \| rotate \| toggle \| extend \| covert-token \| rotate-secrets`, `DELETE /api/subs/:id/devices/:dev_id`, `POST /api/subs/rotate-token`, `POST /api/subs/clear-strikes`, `POST /api/subs/import-external`, `GET/POST /api/sub-profiles`, `DELETE /api/sub-profiles/:id`, `POST /api/subs/apply-profile`, `POST /api/subs/quick-invite`, `POST /api/subs/batch-action`, `POST /api/subs/inspect-headers`, `POST /api/vouchers/batch`, `POST /api/redeem` |
| **Nodes, Cluster, Cascades & Telegram Bot** | `GET/POST /api/nodes`, `DELETE /api/nodes/:id`, `POST /api/nodes/test-managed`, `POST /api/nodes/:id/sync-clients`, `POST /api/nodes/:id/export-outbound`, `DELETE /api/nodes/:id/export-outbound/:inb_id`, `POST /api/nodes/sync \| health-check \| multi-clone`, `POST /api/cascade/preview \| /api/cascade`, `GET/PUT /api/tg-bot`, `POST /api/tg-bot/verify \| broadcast \| send-message \| users/adjust \| backup-now \| test-alert`, `POST /api/tg-bot/promo`, `DELETE /api/tg-bot/promo/:code`, `POST /api/tg-bot/orders/:id/approve \| reject`, `DELETE /api/tg-bot/orders/:id`, `DELETE /api/tg-bot/tickets/:id`, `POST /api/billing/issue-sub` |
| **SmartDNS / GeoHide & SNI Proxy** | `GET/PUT /api/smart-dns`, `GET /api/smart-dns/probe`, `GET /api/smart-dns/lists`, `POST /api/smart-dns/lists/sync`, `POST /api/smart-dns/export-node-egress`, `POST /api/smart-dns/wizard-apply`, `POST /api/smart-dns/bind-my-ip`, `DELETE /api/smart-dns/dynamic-ip/:ip`, `POST /api/smart-dns/rotate-token`, plus public `/dns-bind/<token>` & `/dns-query[/<token>]` |
| **`tachyon-core` Runtime Bridge** | `GET /api/core/connections`, `POST /api/core/connections/close \| kick \| block-domain \| route-domain`, `GET /api/core/proxies`, `POST /api/core/proxies/select \| delay`, `POST /api/core/outbound/reset`, `GET /api/core/health`, `GET /api/core/anomalies`, `GET/POST /api/core/failover`, `POST /api/core/failover/restore`, `POST /api/core/route-check`, `POST /api/core/connection-test`, `POST /api/core/dns-check \| dns/flush`, `GET/POST /api/core/leak`, `GET/DELETE /api/core/flight`, `GET /api/core/memory`, `POST /api/core/memory-trim`, `POST /api/core/warp/register` |

---

[Previous: Chapter 5 — Protocols, Multi-Node Clusters, Cascades & Telegram Bot](05-protocols-nodes-and-tgbot.md) | [UI Tab & Button Reference](11-ui-every-tab-and-button-guide.md) | [Back to Index](../../README.md) | [Next: Chapter 7 — Full Setup & Cluster Guide](07-full-setup-and-cluster-guide.md)
