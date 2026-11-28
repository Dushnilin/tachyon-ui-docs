# 19. OpenConnect (AnyConnect / GlobalProtect / FortiGate / Pulse)

| Параметр | Значение |
| :--- | :--- |
| **Роль в `tachyon-ui`** | `Inbound` + `Outbound` (`outbounds[]` и `inbounds[]` полнофункциональны; секция `endpoints[]` поддерживается только на этапе `check`) |
| **L4-транспорт** | `TCP` (TLS / CSTP) + `UDP` (DTLS для `anyconnect` на outbound; ESP при `ip: true`) |
| **TLS / REALITY / ACME** | **Строго обязателен** сертификатный `tls` (REALITY не поддерживается) |
| **Статус в `tachyon-core 0.6.9`** | Полная поддержка Inbound и Outbound в схеме (`check`) и рантайме (`run`) |

---

## Зачем нужен и как работает в `tachyon-core`

**OpenConnect** — открытая реализация корпоративных SSL VPN-протоколов, совместимая с клиентами Cisco AnyConnect, Palo Alto GlobalProtect, Fortinet FortiGate, Juniper Pulse Secure и другими. Для внешнего наблюдателя и сетевых шлюзов такое подключение выглядит как легитимный корпоративный SSL VPN поверх HTTPS.

Как протокол работает в `tachyon-core 0.6.9`:

1. Клиент подключается по стандартному `HTTPS` (`TLS`) на выбранный порт, указывая совместимый диалект (`flavor`): `anyconnect`, `gp`, `pulse`, `fortinet` (на уровне ядра поддерживаются также `nc` и `f5`).
2. После проверки учётных данных и `cookie` сервер поднимает L3-туннель через userspace-стек `smoltcp` (в `tachyon-ui` для inbound всегда выставляется `"ip": true`).
3. Полезные IP-пакеты передаются внутри TLS-сессии (`CSTP`) либо — при доступности UDP — по ускоренному каналу `DTLS` (для `anyconnect`) или `ESP` (для совместимых диалектов при активном `ip: true`).

---

## Скриншот формы в панели

![Форма настройки протокола OpenConnect в tachyon-ui](../../images/protocols/ru/proto-19-openconnect.png)

---

## Пошаговая настройка в `tachyon-ui`

### Настройка входящего подключения (`Inbound`)

1. Откройте **Входящие (`Inbounds`) → Добавить** и выберите протокол **`openconnect`**.
2. Выберите один из готовых пресетов панели:
   - **`Cisco AnyConnect SSL VPN`** — порт `8468`, TLS, SNI `www.microsoft.com`, `flavor: anyconnect`, `mtu: 1400`.
   - **`Palo Alto GlobalProtect`** — порт `8468`, TLS, SNI `www.cloudflare.com`, `flavor: gp`, `mtu: 1400`.
   - **`Fortinet FortiGate`** — порт `8468`, TLS, SNI `www.microsoft.com`, `flavor: fortinet`, `mtu: 1400`.
3. В блоке параметров протокола выберите нужный **`Диалект (flavor)`** (`anyconnect`, `gp`, `fortinet`, `pulse`) и **`MTU`** (`1200..1480`, по умолчанию `1400`).
4. На вкладке **TLS** проверьте домен `SNI` (`panel.example.com`) и убедитесь, что заданы действительные `certificate_path` и `key_path` (например, выпущенные панелью через ACME).
5. Добавьте клиента (`Имя` и `Пароль`): пароль первого активного клиента панель автоматически использует в качестве `cookie` для авторизации туннеля. Сохраните правило.

### Настройка исходящего подключения (`Outbound`)

1. Перейдите во вкладку **Исходящие (`Outbounds`) → Добавить** и выберите **`openconnect`**.
2. Задайте `Сервер`, `Порт`, `Пароль` и настройки `TLS` (`tls.enabled: true` обязателен).
3. В дополнительных JSON-настройках (`out-settings`) при необходимости укажите `flavor`, `username`, `cookie`, `ip: true`, `dtls`, `esp` и `server_cert_pin`. Обратите внимание: для `flavor: "pulse"` на outbound обязательны и `username`, и `password`, а включение `esp: true` обязательно требует `ip: true`.

---

## Разбор всех полей, переключателей и параметров

### Входящее подключение (`Inbound`)

| Поле в панели | Ключ `tachyon-core` JSON | Допустимые значения / дефолт | Назначение |
| :--- | :--- | :--- | :--- |
| **Адрес прослушивания** (`inb-listen`) | `listen` | IPv4/IPv6; дефолт `0.0.0.0` (или `::`) | Интерфейс прослушивания HTTPS/DTLS |
| **Порт** (`inb-port`) | `listen_port` | `1..65535`; пресет `8468` (часто `443` за внешним маршрутом) | Входящий порт OpenConnect (TCP и UDP) |
| **Диалект** (`inb-oc-flavor`) | `flavor` | `anyconnect`, `gp`, `fortinet`, `pulse` (в ядре также `nc`, `f5`) | Профиль совместимости с конкретным семейством корпоративных клиентов |
| **MTU** (`inb-oc-mtu`) | `mtu` | `1200..1480`; дефолт `1400` | Размер MTU виртуального L3-интерфейса |
| **Режим L3 IP** *(авто)* | `ip` | `true` (панель выставляет автоматически) | Включает полноценную маршрутизацию IP-пакетов через стек `smoltcp` |
| **Cookie / клиент** (`clients[]`) | `cookie` | Строка (из пароля первого активного клиента) | Токен/cookie авторизации входящей сессии |
| **Вкладка TLS** (`inb-tls-*`) | `tls` | Только `tls` (`enabled: true` обязательно) | Сертификат сервера, ключ, SNI и настройки TLS |

### Исходящее подключение (`Outbound`)

| Поле в панели | Ключ `tachyon-core` JSON | Допустимые значения / дефолт | Назначение |
| :--- | :--- | :--- | :--- |
| **Сервер** (`out-server`) | `server` | IP или домен (`198.51.100.10`, `panel.example.com`) | Адрес SSL VPN-шлюза |
| **Порт** (`out-port`) | `server_port` | `1..65535` | Порт SSL VPN-шлюза |
| **Пароль** (`out-password`) | `password` | Строка (обязательно для `pulse` в паре с `username`) | Пароль учётной записи |
| **Диалект** *(в `out-settings`)* | `flavor` | `anyconnect`, `gp`, `nc`, `pulse`, `f5`, `fortinet` | Выбор протокола удалённого шлюза |
| **Логин и cookie** *(в `out-settings`)* | `username`, `cookie` | Строки | Имя пользователя и готовый session cookie |
| **IP / DTLS / ESP** *(в `out-settings`)* | `ip`, `dtls`, `esp` | `bool` (`esp: true` требует `ip: true`; `dtls` для `anyconnect`) | Управление L3-режимом и UDP-транспортом |
| **Пиннинг сертификата** *(в `out-settings`)* | `server_cert_pin` | Строка (pin-хеш сертификата сервера) | Привязка к отпечатку сертификата удалённого шлюза |
| **Вкладка TLS** (`out-tls*`) | `tls` | `{ "enabled": true, ... }` (строго обязательно) | Параметры TLS-клиента (`server_name`, `insecure`) |

---

## Сеть, TLS и файрвол

- **TLS / REALITY / ACME:** серверный и клиентский TLS **строго обязателен**; REALITY не применяется. Для корректной работы стандартных клиентов (Cisco AnyConnect, OpenConnect, NetworkManager) используйте действительный домен (`panel.example.com`) и сертификат, выданный панелью через ACME. Встроенный ACME внутри `tachyon-core` не поддерживается — панель сама обновляет файлы `certificate_path`/`key_path`.
- **Файрвол:** панель `tachyon-ui` автоматически открывает **`TCP` и `UDP`** на порту `listen_port` (например, `8468/tcp` и `8468/udp`) для поддержки как TLS/CSTP, так и быстрого UDP-канала (DTLS/ESP).

---

## Экспорт подписок, совместимые клиенты и пример валидного JSON `tachyon-core`

- **Форматы экспорта подписок (`GET /sub/{sub_id}?format=...`):**
  - **URI (`raw` / Base64):** `openconnect://alice:secret-pass@panel.example.com:8468?flavor=anyconnect&ip=1#OpenConnect`
  - **Клиентский JSON (`singbox` / `tachyon` / `outbounds`):** генерируется объект `type: "openconnect"` с `flavor`, `username`, `password`, `cookie`, `ip: true` и `tls`.
  - **Clash Meta YAML и SIP008:** не экспортируется.
- **Совместимые клиенты:** `tachyon-core`, OpenConnect (Linux / OpenWrt / NetworkManager / macOS / Windows), совместимые клиенты AnyConnect / GlobalProtect / FortiClient.

### Пример валидного JSON `tachyon-core`

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

## Диагностика и частые ошибки

| Симптом / ошибка | Причина | Решение |
| :--- | :--- | :--- |
| Ошибка валидации `mtu` на outbound в ручном JSON | Строгий валидатор схемы на outbound ожидает `mtu` в виде строки либо без этого поля | Опустите `mtu` на outbound (по умолчанию используется стандартный) или передавайте его строкой при ручной правке |
| Ошибка запуска outbound с включённым `esp: true` | Включён `esp: true` при отсутствующем или `false` параметре `ip` | Обязательно добавьте `"ip": true` вместе с `"esp": true` |
| Ошибка авторизации в режиме `pulse` | Указан только пароль или только имя пользователя | Для `flavor: "pulse"` на outbound задавайте оба поля — `username` и `password` |
| Корпоративный клиент отклоняет подключение сразу после TCP Connect | Самоподписанный сертификат без доверия в системе или несовпадение `SNI` | Подключите домен `panel.example.com` с сертификатом ACME либо передайте корректный `server_cert_pin` |

---

## См. также

- [Каталог протоколов](README.md)
- [Энциклопедия протоколов](../09-protocols-encyclopedia.md)
- [18. Sudoku](18-sudoku.md)
- [20. Cloudflared](20-cloudflared.md)
