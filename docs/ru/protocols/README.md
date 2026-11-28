# Каталог протоколов Tachyon (`tachyon-ui v0.4.12` + `tachyon-core 0.6.9`)

[Полное руководство](../00-complete-guide.md) | [Энциклопедия протоколов](../09-protocols-encyclopedia.md)

Практический справочник по всем **27 протоколам, транспортам и маршрутным мета-исходящим** в панели `tachyon-ui v0.4.12` и движке `tachyon-core 0.6.9`: поля форм, маппинг JSON, пресеты, ограничения ядра, правила файрвола, форматы подписок и диагностика.

---

## Быстрая матрица выбора протокола

| Задача / сценарий | Рекомендуемый протокол | L4-транспорт | TLS и свой домен | Почему именно он |
| :--- | :--- | :--- | :--- | :--- |
| **Основной универсальный канал Tachyon** | [01. TewFai](01-tewfai.md) | TCP + QUIC (UDP) + HTTP | REALITY (домен не нужен) или TLS | Единая сессия с переключением носителей, морфинг, XOR-FEC, PQC |
| **Быстрый старт без покупки домена** | [02. VLESS](02-vless.md) (`REALITY`) / [09. ShadowTLS v3](09-shadowtls.md) | TCP | REALITY / decoy TLS | Маскируется под реальный TLS 1.3 handshake публичного сайта |
| **Плохой мобильный канал, потери пакетов** | [01. TewFai](01-tewfai.md) (QUIC + FEC) / [03. Hysteria2](03-hysteria2.md) / [04. TUIC v5](04-tuic.md) | UDP (QUIC HTTP/3) | Обязательный TLS | Нет блокировки потоков (HoL), BBR/Brutal или FEC держат скорость |
| **Корпоративные сети, CDN и строгие прокси** | [10. NaiveProxy](10-naive.md) / [12. TrustTunnel](12-trusttunnel.md) / [02. VLESS](02-vless.md) (`ws`/`xhttp`) | TCP (H2) / UDP (H3) | Нужен реальный домен + сертификат | Стандартный HTTP/2 или HTTP/3 трафик |
| **Игры, звонки, минимальный ping на UDP** | [04. TUIC v5](04-tuic.md) / [07. Shadowsocks](07-shadowsocks.md) | UDP / TCP+UDP | TLS (TUIC) / без TLS (SS-2022) | Нативная передача UDP-дейтаграмм и 0-RTT |
| **L3 VPN для роутера (OpenWrt / Keenetic)** | [22. AmneziaWG](22-amneziawg.md) / [21. WireGuard](21-wireguard.md) / [11. MASQUE](11-masque.md) | UDP (L3 IP) | Без TLS (WG/AWG) / TLS (MASQUE) | Минимальный оверхед на слабом CPU роутера |
| **Обратная совместимость с классическими клиентами** | [05. Trojan](05-trojan.md) / [06. VMess](06-vmess.md) / [07. Shadowsocks](07-shadowsocks.md) | TCP (+ UDP в SS) | TLS / REALITY (Trojan/VMess) | Поддержка практически любым сторонним клиентом |
| **Отказоустойчивость и каскады узлов** | [27. Балансировщики и Chain](27-balancers-and-chain.md) | Наследуется от узлов | Наследуется от узлов | 12 стратегий `urltest`, ручной `selector` и многохоповый `chain` |

---

## Общие механизмы `tachyon-ui` и `tachyon-core 0.6.9`

### Режимы TLS, REALITY и ACME
- **Доступные режимы вкладки TLS (`tls.mode`)** определяются возможностями протокола в панели:
  - **Без TLS (`[]`)**: `shadowsocks`, `shadowtls`, `snell`, `mieru`, `mtproto`, `wdtt`, `wireguard`, `amneziawg`, `socks`, `cloudflared`.
  - **Только сертификатный TLS (`['tls']`)**: `hysteria2`, `tuic`, `naive`, `masque`, `trusttunnel`, `openconnect`.
  - **TLS или REALITY (`['tls', 'reality']`)**: `anytls`, `fptn`.
  - **Plain или сертификатный TLS (`['none', 'tls']`)**: `mixed`, `http`, `sudoku`.
  - **Полный спектр (`['none', 'tls', 'reality']`)**: `tewfai`, `vless`, `vmess`, `trojan`.
- **ACME-сертификаты**: встроенный запуск ACME внутри серверного рантайма `tachyon-core 0.6.9` отключён (валидатор проверяет поля `tls.acme` / `tls.certificate_provider`, но запуск возвращает ошибку). В продакшене выпуском и обновлением Let's Encrypt занимается сама панель `tachyon-ui`, передавая движку готовые пути `certificate_path` (`data/certs/server.crt`) и `key_path` (`data/certs/server.key`).
- **Reverse proxy**: поля `external_port` и `external_tls` в настройках Inbound подменяют порт и режим TLS в выдаваемых клиентам подписках, если трафик проходит через внешний NGINX / Caddy на порту `443`.

### Вкладка «Транспорт и Mux»
Для протоколов **VLESS**, **VMess** и **Trojan** доступны общие L7-транспорты (`tcp`, `ws`, `grpc`, `httpupgrade`, `xhttp`, `http`) и мультиплексор `h2mux` (+ `brutal` для контроля полосы пропускания). Для **Shadowsocks** доступна вкладка `Mux` (`h2mux` + `brutal`). В `vless` при наличии клиента с `xtls-rprx-vision` мультиплексирование отключается автоматически.

### Форматы экспорта подписок (`GET /sub/{sub_id}?format=...`)
| Параметр `format` | Что отдаёт панель | Особенности совместимости |
| :--- | :--- | :--- |
| *(пусто)* / `raw` / `uri` | Base64 или построчный список URI | Универсальные ссылки `tewfai://`, `vless://`, `hy2://`, `tuic://`, `trojan://`, `vmess://` и др. |
| `json` / `tachyon` / `outbounds` | Клиентский конфиг `tachyon-core` | Полная поддержка всех нативных полей (TewFai, XHTTP, HTTPUpgrade, PQC, балансировщики, `endpoints[]`) |
| `clash` / `clashmeta` / `mihomo` / `yaml` | YAML-профиль Clash Meta | Только поддерживаемые форматом типы (`vless`, `trojan`, `anytls`, `vmess`, `hysteria2`, `tuic`, `shadowsocks`, `mieru`, `snell`, `socks`/`http`, `wireguard`); `tewfai`, `xhttp`, `httpupgrade`, `chain`, `naive`, `masque`, `trusttunnel`, `fptn`, `wdtt` в YAML пропускаются |
| `sip008` | JSON SIP008 (`version: 1`) | Только для `shadowsocks` |
| `openwrt` / `router` / `html` | Профиль роутера / веб-портал | Автонастройка маршрутизации для шлюзов OpenWrt и удобная страница выдачи клиенту |

---

## Полный каталог протокольных гайдов (`01..27`)

| № | Протокол / группа | Роль в панели | L4-транспорт | TLS / REALITY | Статус в `tachyon-core 0.6.9` | Гайд |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| `01` | **TewFai** | Inbound + Outbound | TCP + QUIC (UDP) + HTTP | `none` / `tls` / `reality` | Полная поддержка | [01-tewfai.md](01-tewfai.md) |
| `02` | **VLESS** | Inbound + Outbound | TCP (+ HTTP-носители; `quic` только Outbound) | `none` / `tls` / `reality` | Полная поддержка | [02-vless.md](02-vless.md) |
| `03` | **Hysteria2** | Inbound + Outbound | UDP (QUIC ALPN `h3`) | Только `tls` | Полная поддержка | [03-hysteria2.md](03-hysteria2.md) |
| `04` | **TUIC v5** | Inbound + Outbound | UDP (QUIC ALPN `h3`) | Только `tls` | Полная поддержка | [04-tuic.md](04-tuic.md) |
| `05` | **Trojan** | Inbound + Outbound | TCP (+ HTTP-носители; `quic` Outbound) | `none` / `tls` / `reality` | Полная поддержка | [05-trojan.md](05-trojan.md) |
| `06` | **VMess** | Inbound + Outbound | TCP (+ HTTP-носители; `quic` Outbound) | `none` / `tls` / `reality` | Полная поддержка (Inbound AEAD `alterId: 0`) | [06-vmess.md](06-vmess.md) |
| `07` | **Shadowsocks** | Inbound + Outbound | TCP + UDP (UoT v1/v2) | Без TLS в UI | Полная поддержка (AEAD и SS-2022) | [07-shadowsocks.md](07-shadowsocks.md) |
| `08` | **AnyTLS** | Inbound + Outbound | TCP | `tls` / `reality` | Полная поддержка | [08-anytls.md](08-anytls.md) |
| `09` | **ShadowTLS v3** | Inbound + Outbound | TCP | Встроенный внешний TLS handshake | Полная поддержка (`v3`, `v1` отклонён) | [09-shadowtls.md](09-shadowtls.md) |
| `10` | **NaiveProxy** | Inbound + Outbound | TCP (H2) / UDP (H3 QUIC) | Только `tls` (`insecure` запрещён) | Полная поддержка | [10-naive.md](10-naive.md) |
| `11` | **MASQUE** | Inbound + Outbound | UDP (H3 `CONNECT-IP`) / H2 | Только `tls` | Полная поддержка | [11-masque.md](11-masque.md) |
| `12` | **TrustTunnel** | Inbound + Outbound | TCP (H2) / UDP (H3 QUIC) | Только `tls` | Полная поддержка | [12-trusttunnel.md](12-trusttunnel.md) |
| `13` | **Mieru** | Inbound + Outbound | TCP или UDP | Встроенное шифрование | Полная поддержка (Mieru v3) | [13-mieru.md](13-mieru.md) |
| `14` | **Snell** | Inbound + Outbound | TCP (+ UDP-релей) | Без TLS | Inbound v5/v6; Outbound v4/v5/v6 | [14-snell.md](14-snell.md) |
| `15` | **MTProto** | Inbound + Outbound | TCP (+ `telegram-ws` / `kws`) | Без TLS (или TLS для `kws`) | Полная поддержка (`32 hex` секрет) | [15-mtproto.md](15-mtproto.md) |
| `16` | **FPTN** | Inbound + Outbound | TCP (HTTPS + WS) | `tls` / `reality` | Полная поддержка (`RealityMode2`) | [16-fptn.md](16-fptn.md) |
| `17` | **WDTT** | Inbound + Outbound | UDP (RTP / DTLS / WG / CSQTT) | Встроенное шифрование | Полная поддержка | [17-wdtt.md](17-wdtt.md) |
| `18` | **Sudoku** | Inbound + Outbound (форма/схема) | TCP (+ HTTP Mask) | `none` / `tls` | Проверка схемы и экспорт URI (нет runtime в `0.6.9`) | [18-sudoku.md](18-sudoku.md) |
| `19` | **OpenConnect** | Inbound + Outbound | TCP (CSTP) + UDP (DTLS/ESP) | Только `tls` | Полная поддержка (6 диалектов) | [19-openconnect.md](19-openconnect.md) |
| `20` | **Cloudflared** | Inbound only | Исходящий QUIC / H2 к Edge | По токену Cloudflare | Требует сборки с флагом `with_cloudflared` | [20-cloudflared.md](20-cloudflared.md) |
| `21` | **WireGuard** | Endpoint (Inbound + Outbound UI) | UDP (L3 IP) | Без TLS (`Noise_IKpsk2`) | Полная поддержка в `endpoints[]` | [21-wireguard.md](21-wireguard.md) |
| `22` | **AmneziaWG** | Endpoint (Inbound + Outbound UI) | UDP (L3 IP + Junk/CPS) | Без TLS (`Noise_IKpsk2`) | Полная поддержка (AWG 2.0/3.x в `endpoints[]`) | [22-amneziawg.md](22-amneziawg.md) |
| `23` | **Mixed / SOCKS / HTTP** | Inbound + Outbound | TCP (+ UDP для SOCKS5/Mixed) | `none` / `tls` (`mixed`, `http`) | Полная поддержка | [23-mixed-socks-http.md](23-mixed-socks-http.md) |
| `24` | **Desync** | Outbound only | TCP / UDP (прямой выход) | Без TLS | Полная поддержка (адаптивный режим и `udp_fake`) | [24-desync.md](24-desync.md) |
| `25` | **Hysteria (v1)** | Outbound (схема) | UDP (`udp`/`wechat-video`/`faketcp`) | Требуется `tls` | Проверка схемы (нет runtime в `0.6.9`, используйте `03-hysteria2.md`) | [25-hysteria1.md](25-hysteria1.md) |
| `26` | **SSH / Tor / Bridge** | Outbound only (схема) | TCP (`ssh`) / процесс (`tor`) / L2-L3 (`bridge`) | Без TLS | Проверка схемы (нет runtime в `0.6.9`) | [26-ssh-tor-bridge.md](26-ssh-tor-bridge.md) |
| `27` | **Балансировщики и Chain** | Outbound-группы | Наследуется от узлов | Наследуется от узлов | Полная поддержка (`urltest`, `selector`, `chain`) | [27-balancers-and-chain.md](27-balancers-and-chain.md) |

---

## См. также
- [Энциклопедия протоколов простыми словами](../09-protocols-encyclopedia.md)
- [Сводный путеводитель по развёртыванию Tachyon-UI](../00-complete-guide.md)
