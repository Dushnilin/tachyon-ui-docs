# 10. NaiveProxy (HTTPS/2 и HTTP/3 QUIC)

> **Сводная карточка**
> - **Роль в `tachyon-core`**: `Inbound` (сервер) и `Outbound` (клиент)
> - **L4-транспорт**: `TCP` (HTTP/2 `CONNECT` поверх TLS) и/или `UDP` (HTTP/3 QUIC с ALPN `h3` при `naive_quic: true`). UDP-трафик клиентов переносится через встроенный `UoT v1/v2` (`udp_over_tcp`)
> - **TLS / REALITY / ACME**: сертификатный `TLS` (`tls.enabled: true`) **строго обязателен** (`tlsModes: ['tls']`). REALITY не поддерживается. **Строгий запрет `tls.insecure: true` и `tls.disable_sni: true`** на `Outbound`
> - **Нужен ли домен**: **да, строго обязателен** реальный домен с доверенным TLS-сертификатом (ACME / CA), чей SAN совпадает с `server_name`

---

## Зачем нужен и как работает в `tachyon-core`

**Простая аналогия.** Большинство прокси с TLS просто заворачивают свой поток в TLS-оболочку, но длина ответов, заголовки HTTP и поведение при ошибках всё равно выдают прокси. **NaiveProxy** работает как настоящий веб-сервер (на уровне HTTP/2 или HTTP/3 QUIC) и использует сетевой стек браузера Chromium на стороне клиента: трафик туннелируется через стандартный метод `CONNECT` с padding HTTP/2-фреймов, неотличимый от обычного HTTPS-соединения с современным сайтом.

**Механика в `tachyon-core`:**
- В базовом режиме (HTTP/2 over TLS) сервер принимает HTTPS-соединения, проверяет аутентификацию `Basic` (`users[].{username, password}`) и открывает туннели через `CONNECT`, добавляя случайный паддинг в кадры HTTP/2 для сглаживания длин пакетов.
- При включении **HTTP/3 QUIC** (`inb-naive-quic` = `true`) панель выставляет `"network": "udp"` и `alpn: ["h3"]`, переводя транспорт на QUIC поверх UDP.
- **Жёсткий инвариант безопасности `naive`**: валидатор `tachyon-core` на уровне кода **категорически запрещает** `tls.insecure: true` и `tls.disable_sni: true` на исходящем подключении (`insecure is not supported on naive outbound`). Поэтому:
  1. В UI панели для `naive` скрыт переключатель разрешения самоподписанных сертификатов (`inb-tls-insecure-wrap`).
  2. При генерации подписок `tachyon-ui` принудительно выставляет `insecure = false` (никогда не добавляет `insecure=1` в URI и `insecure: true` в JSON).
  3. Экспорт подписок автоматически подбирает в `sni` домен, присутствующий в SAN реального активного сертификата сервера, чтобы клиент успешно прошёл строгую PKI-проверку.
- Если у клиента отсутствует валидный доверенный сертификат (например, используется самоподписанный сертификат без установки его в системное хранилище доверенных корневых CA), подключение NaiveProxy будет немедленно отклонено на этапе TLS handshake.

---

## Скриншот формы в панели

![Форма настройки NaiveProxy в tachyon-ui](../../images/protocols/ru/proto-10-naive.png)

---

## Пошаговая настройка в `tachyon-ui`

### Предварительное требование: домен и доверенный сертификат

1. Направьте A/AAAA-запись вашего домена (например, `panel.example.com`) на IP сервера `198.51.100.10`.
2. Выпустите доверенный TLS-сертификат (через встроенный ACME-менеджер панели или внешний скрипт в `data/certs/server.crt` и `data/certs/server.key`), чтобы ваш домен находился в поле SAN сертификата.

### Настройка входящего подключения (`Inbound`)

1. Откройте раздел **Inbounds** и нажмите **Добавить Inbound**.
2. Выберите протокол **NaiveProxy (`naive`)**.
3. Выберите один из пресетов панели (`PROTOCOL_PRESETS.naive`):
   - **NaiveProxy HTTPS/2** — порт `8450`, TLS `tls`, SNI `www.microsoft.com` (автоматически заменяется на `web_domain` панели, если он задан), ALPN `h2,http/1.1`, `quic: false`.
   - **NaiveProxy HTTP/3 QUIC** — порт `8459`, TLS `tls`, SNI `www.cloudflare.com` (или `web_domain`), ALPN `h3`, `quic: true`.
4. Переключатель **Режим HTTP/3 QUIC (UDP)** (`inb-naive-quic`): оставьте выключенным для работы по TCP/HTTP/2 или включите для работы по UDP/HTTP/3.
5. Во вкладке **TLS**:
   - Укажите ваш реальный домен в **Server Name (SNI)** (`inb-tls-sni`) — он **обязан** совпадать с SAN сертификата!
   - Проверьте пути к сертификату и приватному ключу (`inb-tls-cert`, `inb-tls-key`). Обратите внимание, что выбор `allow_insecure` в UI для `naive` намеренно скрыт.
6. Добавьте клиентов во вкладке клиентов (задаются `username` и `password`). Сохраните и примените конфигурацию.

### Настройка исходящего подключения (`Outbound`)

1. В разделе **Outbounds** выберите тип **`naive`**.
2. Задайте сервер (`panel.example.com` или `198.51.100.10`), порт (`8450` / `8459`), имя пользователя (`out-socks-user`) и пароль (`out-socks-pass`).
3. В разделе **TLS и Транспорт** оставьте **Включить TLS** активным, **не включайте** `Разрешить self-signed (insecure)` и укажите в **Server Name (SNI)** домен, совпадающий с SAN сертификата сервера.

---

## Разбор всех полей, переключателей и параметров

### Входящий узел (`Inbound`)

| Поле в панели (`ID`) | Ключ в `tachyon-core` JSON | Допустимые значения / дефолт | Зачем нужен и как настраивать |
|---|---|---|---|
| **Режим HTTP/3 QUIC (UDP)** (`inb-naive-quic`) | `network: "udp"` (при `true`) | `false` (HTTPS/2 TCP) / `true` (HTTP/3 UDP) | При включении добавляет в конфиг инбаунда `"network": "udp"` и в TLS `alpn: ["h3"]`. В экспортах подписок меняет схему URI на `naive+quic://` и добавляет `"quic": true` в JSON-аутбаунд. |
| **Клиенты** (вкладка клиентов) | `users[].{username, password}` | Непустые `username` и `password` | Используются для HTTP Basic аутентификации в `CONNECT`. При отсутствии активных клиентов панель подставляет заглушку `__disabled__`. |
| **TLS режим** (`inb-tls-mode`) | `tls.enabled: true` | Только `tls` | Обязательный сертификатный TLS. REALITY и `none` запрещены. |
| **Server Name (SNI)** (`inb-tls-sni`) | `tls.server_name` | Реальный домен из SAN сертификата (`panel.example.com`) | Критически важный параметр: без совпадения `server_name` с SAN доверенного сертификата клиенты `naive` откажутся подключаться. |
| **ALPN** (`inb-tls-alpn`) | `tls.alpn` | `["h2", "http/1.1"]` для HTTP/2; `["h3"]` для QUIC | Явно задаёт допустимые версии HTTP при TLS-согласовании. |
| **Сертификат и ключ** (`inb-tls-cert`, `inb-tls-key`) | `tls.certificate_path`, `tls.key_path` | По умолчанию `data/certs/server.crt` / `data/certs/server.key` | Публичный сертификат (желательно полная цепочка fullchain от доверенного УЦ / Let's Encrypt) и приватный ключ. |

### Исходящий узел (`Outbound`)

| Поле в панели (`ID`) | Ключ в `tachyon-core` JSON | Допустимые значения / дефолт | Зачем нужен и как настраивать |
|---|---|---|---|
| **Сервер / Порт** (`out-server`, `out-port`) | `server`, `server_port` | Домен или IP и порт (`8450` / `8459`) | Если сертификат выписан на домен, удобнее сразу указывать домен в `server` (либо задавать его же в `tls.server_name`). |
| **Имя пользователя / Пароль** (`out-socks-user`, `out-socks-pass`) | `username`, `password` | Строки | Учётные данные клиента для HTTP Basic авторизации. |
| **QUIC режим** (через `out-settings`) | `quic`, `quic_congestion_control` | `bool`, `bbr` / `cubic` / `reno` / `bbr2` | Переключает клиент на HTTP/3 QUIC. |
| **TLS** (`out-tls*`) | `tls.{enabled, server_name, alpn}` | `tls.enabled: true`, **`insecure` и `disable_sni` запрещены** | В `tachyon-core` на outbound `naive` нельзя задавать `insecure: true` и `disable_sni: true` — это приведёт к ошибке валидации. |

---

## Сеть, TLS и файрвол

- **Порты файрвола (`inbound_firewall_rules`)**:
  - При `naive_quic: false` открывается **`TCP`** на `inb.port` (в пресете `8450`).
  - При `naive_quic: true` открываются **`TCP` и `UDP`** на `inb.port` (в пресете `8459`).
- **ACME и сертификаты**: `tachyon-core` не запускает встроенный ACME-клиент (при проверке схемы поля `acme` допустимы, но в рантайме выключены). Выпускайте и обновляйте сертификат средствами `tachyon-ui` (раздел домена/TLS) или внешнего ACME-клиента (например, Caddy/certbot).
- **Обратный прокси**: если порт `443` занят веб-сервером (Caddy/NGINX), Caddy может терминировать TLS и передавать трафик на NaiveProxy через `forward_proxy` либо через L4 SNI-проброс (`stream`), при этом в панели можно задать `external_port: 443` для экспорта правильных клиентских ссылок.

---

## Экспорт подписок, клиенты и пример JSON `tachyon-core`

- **URI** (`?format=raw` или Base64):
  - HTTP/2: `naive+https://alice:password@panel.example.com:8450?sni=panel.example.com#NaiveProxy`
  - HTTP/3 QUIC: `naive+quic://alice:password@panel.example.com:8459?sni=panel.example.com#NaiveProxy-QUIC`
  *(Параметр `insecure=1` никогда не добавляется.)*
- **Клиентский JSON** (`?format=singbox`, `?format=tachyon`, `?format=outbounds`): экспортируется как `"type": "naive"`, `"username"`, `"password"`, `"quic": true` (если включён QUIC), `"tls": { "enabled": true, "server_name": "panel.example.com", ... }` (строго без `insecure`).
- **Clash Meta YAML** и **SIP008**: **не экспортируется** — используйте формат `singbox`, `tachyon` или URI-ссылки.
- **Совместимые клиенты**: `tachyon-core` (включая роутеры OpenWrt), оригинальный клиент `naive` / интеграции на базе Chromium.

### Пример валидного JSON для `tachyon-core`

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

## Диагностика и частые ошибки

| Симптом | Причина | Решение |
|---|---|---|
| Ошибка валидации `tachyon-core`: `insecure is not supported on naive outbound` | В ручной конфигурации или импортированном профиле указано `tls.insecure: true` или `tls.disable_sni: true` | Удалите `insecure` и `disable_sni` из объекта `tls` аутбаунда `naive` и используйте доверенный сертификат. |
| Клиент обрывает соединение с ошибкой сертификата (`ERR_CERT_AUTHORITY_INVALID` / `x509`) | На сервере установлен самоподписанный сертификат, истёк срок или `sni` не совпадает с SAN сертификата | Выпустите доверенный Let's Encrypt / ZeroSSL сертификат на ваш домен и укажите этот же домен в `tls.server_name` (и проверьте, что в `data/certs/server.crt` лежит полная цепочка `fullchain`). |
| При включённом `HTTP/3 QUIC` клиент не подключается | Не открыт `UDP` на выбранном порту (например, `8459`) или провайдер режет QUIC/UDP | Откройте UDP на порту инбаунда в файрволе хостера или переключитесь на пресет `NaiveProxy HTTPS/2` (TCP). |
| Подписка `clash` не содержит узел `naive` | Clash Meta YAML не поддерживает протокол `naive` в автоэкспорте | Скачайте подписку в формате `?format=singbox` или `?format=tachyon` либо используйте URI `naive+https://`. |

---

## См. также

- [Каталог протоколов](README.md)
- [Энциклопедия протоколов](../09-protocols-encyclopedia.md)
- [09. ShadowTLS v3](09-shadowtls.md)
- [11. MASQUE (HTTP/3 CONNECT-IP)](11-masque.md)
- [12. TrustTunnel](12-trusttunnel.md)
