# 3. Subscriptions, Web Portal, Vouchers & Tachyon Covert Knock

[Русская версия](../ru/03-subscriptions-and-covert-knock.md) | [Back to Index](../../README.md)

The **Subscriptions** module consolidates all user protocols and endpoints into a single URL or web portal while enforcing four layers of endpoint protection.

![Subscription Links & Covert Knock Modal](../images/en/05-subscriptions-covert.png)

---

## Subscriptions, Profiles & Device Management

- **Grouping by `sub_id`**: Clients sharing the same `sub_id` across different inbound protocols (VLESS+REALITY, Hysteria2, TUIC, WireGuard/AWG, Sudoku, WDTT) are bundled into one subscription.
- **Subscription Profiles (`/api/sub-profiles`) & Quick Invite (`/api/subs/quick-invite`)**: Reusable presets (device limit, On-Hold `start_on_first_use`, `daily`/`weekly`/`monthly` traffic reset, balancer sets, `announce` banner) applied in one click or used for instant test/VIP invite generation.
- **360-Degree Inspector, Batch Actions & External Import**: Right-side subscription inspector drawer, batch operations (`/api/subs/batch-action` to enable/disable, extend days, or apply profiles), token rotation (`/api/subs/rotate-token`), strike clearing (`/api/subs/clear-strikes`), request header inspector (`/api/subs/inspect-headers`), and external subscription imports (`/api/subs/import-external`).
- **HWID Device Limits**:
  - `device_limit` caps bound devices / concurrent IPs; `strict_hwid` enforces an `x-hwid` header.
  - Clients can self-unbind old devices via the portal (`POST <sub_path>/unbind-device`), governed by `sub_unbind_cooldown_hours`.
- **Account Guard / Anti-Share**: Tracks concurrent `/24` subnets per subscription (`anti_share_max_subnets`), records strikes, and logs `suspect_ips`.
- **Custom Entry Hosts (`hosts`), External Front & `node_rules`**: Override public host/port/SNI/Host header for CDN or transit relays (while `unified_443_front` automatically exports localhost L7 nodes as `<domain>:443` with strict TLS and zero `insecure` flags), and restrict specific nodes to chosen `.trs` rule-sets or traffic classes (`only_this` / `prefer_node`, enforced server-side via `server_unmatched`: `block` / `direct` / `client_only`).
- **Web Portal Branding & Customization**: Configure layout (`sub_page_layout`), accent color (`sub_theme_accent`), logo URL (`sub_logo_url`), announcement text (`sub_announce`), support/buy/channel links (`sub_support_url`, `sub_buy_url`, `sub_buy_label`, `sub_tg_channel_url`), node list/FAQ/donation toggles (`sub_show_nodes`, `sub_show_faq`, `sub_show_donate`), allowed client app buttons (`sub_enabled_apps`), client DNS & ad-blocking (`sub_client_dns`, `sub_client_block_ads`), plus custom CSS/HTML (`sub_custom_css`, `sub_custom_html`).
- **Vouchers (`/redeem`)**: Batch-generate redemption codes (`/api/vouchers/batch`) that users exchange for active subscriptions at `/redeem` or a custom `redeem_path`.

---

## Supported Client Output Formats & Query Options

A single subscription endpoint adapts automatically via `User-Agent`, or explicitly via `?format=` (or `?fmt=`):

| Output Format | Query `?format=` | Notes |
| :--- | :--- | :--- |
| **`tachyon-core` / `sing-box`** | `singbox` / `sing-box` / `tachyon` / `json` | Full JSON profile including all 12 `urltest` modes, `balancer`, `chain`, `time_window`, `.trs` rules, and native protocols. |
| **JSON Outbounds Only** | `outbounds` | Returns only the JSON `outbounds` array without route/DNS wrapper blocks. |
| **Clash / Mihomo** | `clash` / `clashmeta` / `meta` / `mihomo` / `yaml` | YAML with `select`, `url-test`, `fallback`, `load-balance`, and `relay` groups. |
| **Raw URI List** | `raw` / `uri` / `plain` | Unencoded line-by-line URI list (`vless://`, `hy2://`, `tuic://`, `trojan://`, `ss://`, `vmess://`). |
| **Base64 (default)** | Auto / generic clients | Base64-encoded URI list with `Subscription-Userinfo`, `Profile-Update-Interval`, and Happ routing headers when `happ_routing_enabled` is active. |
| **SIP008 (Shadowsocks)** | `sip008` | Standard SIP008 JSON profile for Shadowsocks clients. |
| **OpenWrt Router Config** | `openwrt` / `router` | Ready-to-apply OpenWrt router configuration (`build_openwrt_router_config_with_opts`). |
| **OpenWrt 1-Click Installer** | `openwrt-install` / `router-install` / `install.sh` | Executable shell script for automated subscription deployment on OpenWrt routers. |
| **WireGuard / AmneziaWG** | Portal / node export | Ready-to-import `.conf` tunnel profile including `jc`/`jmin`/`jmax`/`s1`/`s2`/`h1..h4`. |
| **Web Portal** | `html` / `portal` / `page` (or `Accept: text/html`) | User portal displaying traffic quota, device list, QR code, and one-click client import links. |

### Dynamic Subscription Query Parameters

Append filter or routing overrides directly to any subscription URL:
- `?country=DE,NL` (or `?cc=US`) — limits exported nodes to the comma-separated 2-letter ISO country codes.
- `?mode=<mode>` — switches generated client routing rules on the fly:
  - `antiblock` — routes selected international services and platforms through the tunnel while keeping other traffic direct;
  - `ru-direct` / `split` — keeps regional `.ru` / `.su` / `.рф` and local banking/portal domains direct while proxying everything else;
  - `gaming` / `game` — prioritizes low-latency interactive and gaming traffic routing;
  - `full` / `global` — routes all client traffic through the tunnel outbound.

---

## 4-Level Stealth Subscription Protection (`StealthSubGate`)

### Level 1 — Polymorphic Paths (`sub_path_style`)

Configured in Wizard Step 3. `resolve_subscription_url` produces consistent URLs across the Web UI, REST API, and Telegram bot:

| `sub_path_style` | URL Pattern |
| :--- | :--- |
| `sub` | `/<sub_path>/<sub_id>` (default `/sub/<sub_id>`) |
| `static_feed` | `/static/feed/<sub_id>` |
| `cdn_v2` | `/cdn/v2/<sub_id>` |
| `rss_updates` | `/rss/updates/<sub_id>` |

Enabling `sub_legacy_alias` preserves backward compatibility with `/sub/<sub_id>` during migration; disable it once clients update to maximize your Self-Audit score.

### Level 2 — Zero-Oracle & Automated Scanner Filtering

- **Zero-Oracle**: Unknown, disabled, or expired `sub_id` requests **never** return JSON errors (`{"error":"subscription not found"}`). They always return `HTTP/1.1 404 Not Found` HTML rendered by `SiteEngine` with `Server: nginx/1.26.2`.
- **Scanner User-Agent Filter (`sub_block_scanners`)**: Requests from automated scanners and crawlers (`curl`, `wget`, `python-requests`, `go-http-client`, `zgrab`, `censys`, `shodan`, `masscan`, `nmap`, `httpx`, `nuclei`, `googlebot`, `bingbot`, `yandex`, `ahrefs`, `semrush`, `sqlmap`, `nikto`) receive the public site or decoy 404 even on **valid** subscription URLs, while legitimate proxy clients (`tachyon`, `sing-box`, `clash`, `mihomo`, `v2ray`, `hiddify`, `nekobox`, `shadowrocket`, `happ`, `streisand`, `karing`, `flclash`) are served normally.

### Level 3 — Browser Portal Guard (`sub_portal_guard`)

When set to `secret_key`, opening a subscription URL in a web browser requires either `?k=<portal_key>` or validated Telegram Mini App `tgWebAppData`. Browsers without the key receive the public decoy response while proxy apps continue fetching without extra query parameters.

### Level 4 — Tachyon Covert Knock (Subscription Delivery over Normal Website Routes)

**Tachyon Covert Knock** eliminates dedicated `/sub/` or `/cdn/v2/` URLs completely. Subscriptions are served on ordinary website routes (such as `/assets/manifest.json` or `/`) only when a valid HMAC knock token is presented.

**How it works:**
1. Every subscription owns a unique `knock_secret`.
2. Tokens follow `<sub_id>.<unix_ts>.<hmac16>`, where `<hmac16>` is the first 16 hex characters of `HMAC-SHA256(knock_secret, "{sub_id}:{unix_ts}")`, verified in constant time (`constant_time_eq`).
3. Requests must fall within a `±300s` replay window (`|now - unix_ts| <= 300`).
4. Clients can send the token via:
   - `If-None-Match: W/"<sub_id>.<ts>.<hmac16>"` (resembles a standard browser HTTP cache ETag);
   - `X-Client-Digest: <sub_id>.<ts>.<hmac16>`;
   - query parameter `?v=<sub_id>.<ts>.<hmac16>` (with optional `&fmt=<format>`).
5. **Behavior**:
   - Valid knock → returns live subscription payload (`200 OK`);
   - Missing, expired, or tampered token → returns normal `200 OK` public site HTML or `/assets/manifest.json` **with zero knock error indicators**.

In the **Links / Knock** modal (`#subs`), you can issue temporary tokens with ready-to-run `curl` snippets (`POST /api/subs/:id/covert-token`) or rotate both `portal_key` and `knock_secret` (`POST /api/subs/:id/rotate-secrets`).

```bash
# Without knock header -> normal public site manifest (200 OK, application/json)
curl -s https://edge.example.com/assets/manifest.json

# With valid ETag knock -> live subscription payload
curl -s -A "Happ/2.5.0" \
  -H 'If-None-Match: W/"sub-pro-team.1795400000.a1b2c3d4e5f60708"' \
  https://edge.example.com/assets/manifest.json
```

---

Next: [Chapter 4 — Balancers (12 `urltest` Modes), `chain` Outbounds & Routing](04-balancers-chain-and-routes.md).
