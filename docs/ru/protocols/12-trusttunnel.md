# 12. TrustTunnel (HTTP/2 и HTTP/3 QUIC)

> **Сводная карточка**
> - **Роль в `tachyon-core`**: `Inbound` (сервер) и `Outbound` (клиент)
> - **L4-транспорт**: `TCP` (HTTP/2 `CONNECT` поверх TLS — по умолчанию) или `UDP` (HTTP/3 QUIC с ALPN `h3` при `quic: true`). Переносит TCP-потоки, UDP-дейтаграммы (через внутренний псевдохост `_udp2` и UoT) и ICMP echo
> - **TLS / REALITY / ACME**: сертификатный `TLS` (`tlsModes: ['tls']`) обязателен в UI панели; на `Inbound` в `tachyon-core` `tls` строго обязателен при `quic: true`, на `Outbound` `tls.enabled: true` обязателен всегда. REALITY не поддерживается
> - **Нужен ли домен**: желателен домен со своим TLS-сертификатом (через ACME в панели); для тестов на `Outbound` допустимы `server_name` и стандартные настройки TLS

---

## Зачем нужен и как работает в `tachyon-core`

**Простая аналогия.** **TrustTunnel** можно представить как современный HTTPS-шлюз, который умеет говорить с внешним миром либо по классическому **HTTP/2** (TCP), либо по быстрому **HTTP/3 QUIC** (UDP). Внутри стандартного HTTPS-соединения с обычной Basic-авторизацией он аккуратно переносит не только веб-трафик (TCP), но и игровые/голосовые UDP-пакеты и даже сетевые `ping` (ICMP echo) — всё это выглядит как обычный обмен данными с HTTPS-сервером.

**Механика в `tachyon-core`:**
- В режиме по умолчанию (`quic: false`) работает поверх **TCP + TLS + HTTP/2 `CONNECT`**, что обеспечивает максимальную совместимость даже в сетях, где полностью заблокирован UDP или режется QUIC.
- При включении **`quic: true`** (`inb-tt-quic` в панели) ядро поднимает сервер на **HTTP/3 QUIC поверх UDP** (с ALPN `["h3"]`), уменьшая задержку установки соединений (устраняется TCP head-of-line blocking). На `Outbound` в этом режиме можно дополнительно регулировать алгоритм управления перегрузкой `quic_congestion_control` через JSON.
- Аутентификация выполняется по **HTTP Basic** (`username` + `password`). На стороне `Inbound` в `tachyon-core` проверка пароля выполняется за постоянное время (constant-time comparison) для защиты от атак по времени, и список `users[]` не может быть пустым. На стороне `Outbound` `username` и `password` должны задаваться **строго парой** (указание одного без другого отклоняется).
- UDP-трафик передаётся через специальный внутренний псевдохост `_udp2` и механизм UDP-over-TCP, а также поддерживается транзит ICMP echo.

---

## Скриншот формы в панели

![Форма настройки TrustTunnel в tachyon-ui](../../images/protocols/ru/proto-12-trusttunnel.png)

---

## Пошаговая настройка в `tachyon-ui`

### Настройка входящего подключения (`Inbound`)

1. Откройте раздел **Inbounds** и нажмите **Добавить Inbound**.
2. Выберите протокол **TrustTunnel (`trusttunnel`)**.
3. Выберите один из встроенных пресетов панели (`PROTOCOL_PRESETS.trusttunnel`):
   - **TrustTunnel HTTP/2** — порт `8465` (TCP), `tls`, SNI `www.microsoft.com`, ALPN `h2,http/1.1`, `quic: false`.
   - **TrustTunnel HTTP/3 QUIC** — порт `8466` (UDP), `tls`, SNI `www.cloudflare.com`, ALPN `h3`, `quic: true`.
4. Переключатель **Режим HTTP/3 QUIC** (`inb-tt-quic`):
   - Выключите (`false`) для работы по TCP / HTTP/2.
   - Включите (`true`) для работы по UDP / HTTP/3 QUIC.
5. Во вкладке **TLS** проверьте **Server Name (SNI)** (желательно указать ваш реальный домен `panel.example.com`, привязанный к сертификату в `data/certs/server.crt` / `server.key`) и убедитесь, что ALPN соответствует выбранному транспорту (`h2,http/1.1` для HTTP/2 или `h3` для HTTP/3).
6. Во вкладке клиентов создайте хотя бы одного активного пользователя (используются `username` и `password`). Сохраните и примените конфигурацию.

### Настройка исходящего подключения (`Outbound`)

1. В разделе **Outbounds** выберите тип **`trusttunnel`**.
2. Задайте сервер (`panel.example.com` или `198.51.100.10`), порт (`8465` / `8466`) и пароль (`out-password`).
3. **Обязательно** укажите имя пользователя **`username`** в JSON-поле расширенных настроек `out-settings` (например, `{"username": "alice"}`), а при использовании HTTP/3 добавьте `"quic": true` (и при желании `"quic_congestion_control": "bbr"`).
4. В секции **TLS** оставьте **Включить TLS** активным и задайте `server_name` / `alpn`.

---

## Разбор всех полей, переключателей и параметров

### Входящий узел (`Inbound`)

| Поле в панели (`ID`) | Ключ в `tachyon-core` JSON | Допустимые значения / дефолт | Зачем нужен и как настраивать |
|---|---|---|---|
| **Режим HTTP/3 QUIC** (`inb-tt-quic`) | `quic` | `false` (HTTP/2 TCP, дефолт) / `true` (HTTP/3 UDP) | Переключает транспорт между HTTP/2 over TLS (TCP) и HTTP/3 over QUIC (UDP). При `true` панель автоматически проставляет `quic: true` и адаптирует правила файрвола. |
| **Клиенты** (вкладка клиентов) | `users[].{username, password}` | Непустой массив пар `{ username, password }` | Учётные данные HTTP Basic авторизации. При 0 активных клиентов панель подставляет заглушку `__disabled__`. |
| **TLS режим** (`inb-tls-mode`) | `tls.enabled: true` | Только `tls` | В UI обязателен сертификатный TLS; в ядре `tls` строго обязателен при `quic: true`. |
| **SNI / ALPN / Сертификат** (`inb-tls-*`) | `tls.{server_name, alpn, certificate_path, key_path}` | `h2,http/1.1` для HTTP/2; `h3` для QUIC | Рекомендуется использовать реальный домен и доверенный сертификат, выпущенный панелью (ACME). |

### Исходящий узел (`Outbound`)

| Поле в панели (`ID` / `out-settings`) | Ключ в `tachyon-core` JSON | Допустимые значения / дефолт | Зачем нужен и как настраивать |
|---|---|---|---|
| **Сервер / Порт** (`out-server`, `out-port`) | `server`, `server_port` | Хост и порт (`8465` / `8466`) | Адрес удалённого TrustTunnel-сервера. |
| **Пароль и Пользователь** (`out-password` + `out-settings`) | `password`, `username` | Обязательная пара непустых строк | На `Outbound` `username` и `password` принимаются только вместе. Задайте `username` в `out-settings`. |
| **HTTP/3 QUIC** (`out-settings`) | `quic`, `quic_congestion_control` | `bool`; опционально алгоритм CC (`bbr`, `cubic` и др.) | Включает HTTP/3 вместо HTTP/2 на клиентской стороне. |
| **TLS** (`out-tls*`) | `tls.{enabled, server_name, alpn, insecure}` | `tls.enabled: true` строго обязательно | Должен соответствовать настройкам сервера. |

---

## Сеть, TLS и файрвол

- **Порты файрвола (`inbound_firewall_rules`)**:
  - При `trusttunnel_quic: false` открывается **`TCP`** на `inb.port` (в пресете `8465`).
  - При `trusttunnel_quic: true` панель открывает **`TCP` и `UDP`** на `inb.port` (в пресете `8466`).
- **ACME**: сам `tachyon-core` не запускает ACME-клиент; выпуск и обновление сертификатов выполняет `tachyon-ui`, передавая ядру готовые `certificate_path` / `key_path`.
- **Обратный прокси**: в режиме HTTP/2 (`quic: false`) можно использовать L4 SNI-прокси (`stream`) на порту `443` с `external_port: 443` в настройках инбаунда. При работе через L7 HTTP-прокси следите за поддержкой протокола `CONNECT` и передачей заголовка `Authorization`.

---

## Экспорт подписок, клиенты и пример JSON `tachyon-core`

- **URI** (`?format=raw` или Base64):
  - HTTP/2: `trusttunnel://alice:password@panel.example.com:8465?sni=panel.example.com#TrustTunnel-H2`
  - HTTP/3: `trusttunnel://alice:password@panel.example.com:8466?quic=1&sni=panel.example.com#TrustTunnel-H3`
  *(При `allow_insecure: true` добавляется `&insecure=1`.)*
- **Клиентский JSON** (`?format=singbox`, `?format=tachyon`, `?format=outbounds`): экспортируется как `"type": "trusttunnel"`, `"username"`, `"password"`, `"quic": true` (если включён QUIC), `"tls"`.
- **Clash Meta YAML** и **SIP008**: **не экспортируется** — используйте формат `singbox`, `tachyon` или URI.
- **Совместимые клиенты**: `tachyon-core` (включая роутеры OpenWrt) и клиенты с поддержкой TrustTunnel HTTP/2 / HTTP/3.

### Пример валидного JSON для `tachyon-core`

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

## Диагностика и частые ошибки

| Симптом | Причина | Решение |
|---|---|---|
| Ошибка валидации `Outbound` при сохранении/старте: неполная авторизация | Задан `password` без `username` (или наоборот) в ручном `Outbound` | Обязательно укажите оба поля парой (`username` через `out-settings`, `password` в форме). |
| В режиме HTTP/3 (`quic: true`) клиент не подключается | На сервере не открыт UDP-порт `8466`, либо на клиентском провайдере заблокирован QUIC/UDP | Откройте UDP на порту инбаунда или переключите узел на пресет **TrustTunnel HTTP/2** (TCP). |
| Ошибка TLS-проверки на клиенте при работе без собственного домена | Самоподписанный сертификат не разрешён на клиенте | Либо выпустите доверенный сертификат для домена (рекомендуется), либо включите `allow_insecure` только для тестового стенда. |
| Ошибка `tls` при старте ядра с `quic: true` | В ручном JSON не передан объект `tls` при включённом `quic: true` | Для HTTP/3 QUIC сертификатный блок `tls` с `enabled: true` строго обязателен. |

---

## См. также

- [Каталог протоколов](README.md)
- [Энциклопедия протоколов](../09-protocols-encyclopedia.md)
- [10. NaiveProxy](10-naive.md)
- [11. MASQUE (HTTP/3 CONNECT-IP)](11-masque.md)
- [13. Mieru v3](13-mieru.md)
