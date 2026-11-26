# 2. Публичный сайт, Dual HTTP+HTTPS, домен, ACME и Self-Audit

[English version](../en/02-stealth-web-domain-tls.md) | [К оглавлению](../../README.ru.md)

Начиная с версии **v20**, `tachyon-ui` маскирует порт панели под полноценный многостраничный публичный веб-сервер и позволяет принимать на одном или раздельных портах одновременно HTTP и HTTPS трафик без использования стороннего прокси (хотя при желании поддерживает и режим за Nginx / Caddy).

---

## Как обрабатывается каждый входящий запрос

```mermaid
flowchart TD
    IN["Входящее TCP-соединение (listen_port / :80 / :443 / :8443)"] --> PEEK{"Режим web_tls_mode = auto?"}
    PEEK -->|"Первый байт 0x16"| TLS["Pure-Rust TLS handshake (HotCertResolver)"]
    PEEK -->|"Обычный HTTP"| PLAIN["Plain HTTP поток"]
    TLS --> ACME{"Путь /.well-known/acme-challenge/?"}
    PLAIN --> ACME
    ACME -->|"Токен найден"| ACME_OK["200 OK text/plain (HTTP-01 ответ)"]
    ACME -->|"Нет"| REDIR{"Порт HTTP-редиректа (:80)?"}
    REDIR -->|"Да"| R301["301 Moved Permanently -> HTTPS"]
    REDIR -->|"Нет"| SCOPE{"Путь внутри web_base_path?"}
    SCOPE -->|"Нет (или ?decoy=1)"| KNOCK{"Проверка Covert Knock (HMAC ±300с)?"}
    SCOPE -->|"Да (и не ?decoy=1)"| ROOT_CHK{"GET / или /index.html?"}
    ROOT_CHK -->|"Нет (API / login)"| PANEL["Админ-панель API (sid Cookie / Bearer)"]
    ROOT_CHK -->|"Да"| KNOCK_SPA{"Сначала Covert Knock (If-None-Match)?"}
    KNOCK_SPA -->|"Валидный стук"| SUB_OUT["Выдача подписки"]
    KNOCK_SPA -->|"Нет"| UI_CACHE["SPA UI_HTML: проверка ETag -> 304 Not Modified или gzip/raw"]
    KNOCK -->|"Валидный стук"| SUB_OUT
    KNOCK -->|"Нет"| SUBPATH{"Совпадение с sub_path_style?"}
    SUBPATH -->|"Да (валидный клиент, не сканер)"| SUB_OUT
    SUBPATH -->|"Нет / сканер / невалидный id"| SITE["SiteEngine: публичный сайт (200) или стилизованная 404"]
```

> **Порядок `Covert Knock` и кэширования `ETag`:** при запросе `GET /` (или `/index.html`) внутри `web_base_path` (без `?decoy=1`) сервер сначала проверяет `Covert Knock` (`If-None-Match` / `X-Client-Digest` / `?v=`), и только если стук не сработал — передаёт запрос в `serve_cached_ui_asset` для проверки `ETag` (`304 Not Modified`) и отдачи предсжатого `gzip` (`~225 КБ` против `~1.05 МБ`, с заголовками `Vary: Accept-Encoding` и `Cache-Control: no-cache`). Благодаря этому клиентский `If-None-Match` кэш браузера никогда не блокирует скрытый стук.

---

## Мастер «Сайт, Домен, HTTP/HTTPS и Анти-Детект»

Откройте **Настройки → Сайт, Домен, HTTP/HTTPS и Анти-Детект** — настройка разделена на 4 последовательных шага.

### Шаг 1. Публичный сайт (`SiteEngine`)

![Мастер: Шаг 1 — Публичный сайт и живой предпросмотр](../images/ru/10-wizard-site.png)

На первом шаге задаётся внешний облик сервера, который видят браузеры, веб-краулеры и активные зонды на корневом адресе `/`:

- **6 встроенных многостраничных шаблонов** (`site_mode`):
  1. `blank_nginx` — классическая нейтральная страница `nginx/1.26.2`;
  2. `cloud_docs` — портал облачной документации (`Northwind Cloud Docs`);
  3. `status_page` — панель статуса инфраструктуры и нод (`Vertex Edge Status`);
  4. `devops_blog` — инженерный DevOps/SRE блог;
  5. `corporate_it` — корпоративный IT-портал;
  6. `maintenance` — страница плановых технических работ (`503` / `200`).
  Встроенные шаблоны обслуживают не только `/`, но и `/index.html`, `/about`, `/status`, `/docs`, `/assets/app.css`, `/assets/manifest.json`, `/robots.txt`, `/sitemap.xml`, `/favicon.ico` и стилизованную страницу `404 Not Found`.
- **Собственный статический сайт (`custom_static`)**:
  - Загрузите архив `.zip` (**до 15 МБ** сжатый, не более **50 МБ** в распакованном виде и **500 файлов**). Встроенный pure-Rust распаковщик блокирует обход каталогов (zip-slip), абсолютные пути, символические ссылки и исполняемые бинарники.
  - Файлы распаковываются в `<data_dir>/www/`, где доступен встроенный редактор `www/index.html` и кнопка удаления.
- **Живой предпросмотр**: встроенный `<iframe>` обращается к `${BASE}/api/site/preview` и позволяет переключить маршруты `/`, `/about`, `/status`, `/docs` до сохранения.
- **Активный Honeypot**: при включении запросы на типичные пути сканеров (`/.env`, `/wp-admin`, `/phpmyadmin`) фиксируются в журнале безопасности с опциональной автоблокировкой IP.

---

### Шаг 2. Домен, HTTP/HTTPS и сертификаты TLS

![Мастер: Шаг 2 — Профили домена, HTTP/HTTPS и ACME TLS](../images/ru/11-wizard-domain-tls.png)

#### 5 профилей привязки (`web_domain_profile`)

| Профиль | Порты и схема | Когда выбирать |
| :--- | :--- | :--- |
| **`dual_single_port`** | Один порт (`listen_port`, напр. `2053`), режим `web_tls_mode = auto` | Самый быстрый старт: один порт отвечает как по `http://`, так и по `https://` без реверс-прокси. |
| **`native_80_443`** | `web_http_port = 80` (HTTP-01 ACME + `301` редирект на HTTPS), `web_https_port = 443` | Когда домен смотрит прямо на сервер без внешнего Nginx/Caddy и нужен стандартный HTTPS без указания порта. |
| **`reality_self_steal`** | `web_https_bind_addr = 127.0.0.1`, `web_https_port = 8443`, `web_http_port = 80`; порт `:443` отдан REALITY | Рекомендуемый вариант для **REALITY Self-Steal**: REALITY слушает внешний `:443`, а при зондировании или обычном HTTPS-запросе перенаправляет TLS-рукопожатие на локальный сайт `127.0.0.1:8443` самого `tachyon-ui`. |
| **`reverse_proxy`** | Панель в режиме `http` на loopback/порту панели; впереди Nginx или Caddy | Когда на одном сервере уже работают другие веб-проекты на `:80/:443`. Мастер сам генерирует готовые конфиги Nginx и Caddy с кнопкой копирования. |
| **`unified_443_front`** | Впереди NGINX на `443/tcp` (`ssl_preread` + HTTP/L7), `tachyon-core` на `443/udp` + локальные `127.0.0.1` инбаунды | Максимальная маскировка всей ноды на единственном открытом порту `443` (`80/tcp` используется только для ACME HTTP-01 и редиректа). Мастер автоматически адаптирует инбаунды и генерирует готовый `nginx.conf` / `Caddyfile`. |

> При нажатии **Применить профиль** сервер сначала автоматически создаёт точку восстановления (`/api/snapshots`) и обновляет порты через `ListenerSupervisor` без перезапуска процесса:
> - Для `reality_self_steal` синхронизирует `handshake.server = "127.0.0.1"` и `handshake.server_port = 8443` в REALITY inbound.
> - Для `unified_443_front` переводит активные L7-инбаунды (`ws`, `httpupgrade`, `grpc`, `xhttp`/`http`) на `127.0.0.1:10001+` без TLS с сохранением клиентского экспорта `<domain>:443` + строгий TLS (`external_port` / `external_tls`), а основной UDP-инбаунд (`hysteria2` или `tuic`) — на `0.0.0.0:443/udp` с loopback-маскарадом `http://127.0.0.1:18080`.

#### Профиль `Unified 443 Front (NGINX + Core)`

```mermaid
flowchart TD
    EXT["Входящий внешний трафик на сервер"] --> TCP443["TCP :443 — NGINX stream (ssl_preread)"]
    EXT --> UDP443["UDP :443 — tachyon-core (Hysteria2 / TUIC)"]
    TCP443 -->|"Свой SNI домена"| NGINX_HTTP["NGINX HTTP Frontend (127.0.0.1:8443 TLS / без proxy_protocol в REALITY)"]
    TCP443 -->|"Сторонний SNI"| CORE_REALITY["tachyon-core (локальный REALITY inbound)"]
    NGINX_HTTP -->|"/"| DECOY["Публичный сайт-заглушка"]
    NGINX_HTTP -->|"/<web_base_path> и /<sub_path>"| PANEL_LOCAL["tachyon-ui (127.0.0.1:listen_port HTTP)"]
    NGINX_HTTP -->|"/<ws|grpc|xhttp|httpupgrade>"| CORE_L7["tachyon-core L7 (127.0.0.1:10001+ без TLS)"]
    UDP443 -->|"HTTP/3 masquerade"| MASQ["NGINX Loopback (:18080 — только заглушка, админка и /sub/* возвращают 404)"]
```

- При генерации блока `stream { ... }` в `nginx.conf` намеренно **не** добавляется `proxy_protocol on;`, чтобы `tachyon-core` корректно принимал сырой `ClientHello` для REALITY.
- В веб-интерфейсе мастера доступна кнопка скачивания готового `nginx.conf` и сниппета `Caddyfile` (`GET /api/domain/proxy-snippet?profile=unified_443_front`), а в таблице инбаундов адаптированные ноды помечаются индикатором внешнего фронта `-> :443 TLS`.

#### Сертификаты TLS (`cert.pem` и `key.pem` в `<data_dir>`) и диагностика портов

- **SAN Self-Signed** (`POST /api/tls/self-signed`): мгновенно выпускает X.509 сертификат с расширениями SAN для домена и/или IPv4/IPv6 и сразу загружает его в `HotCertResolver`.
- **ACME Let's Encrypt — HTTP-01**: работает без внешнего Caddy или Nginx. При запуске выпуска панель (при включённом `auto_firewall_enabled`) открывает `80/tcp` в системном файрволе, поднимает временный HTTP-01 сервер на `:80` (если `:80` ещё не слушается панелью) и дожидается готовности слушателя перед отправкой запроса ACME. Если порт `80` занят другим процессом или закрыт провайдером, карточка возвращает точную диагностическую подсказку.
- **ACME Let's Encrypt — Cloudflare DNS-01**: автоматически создаёт и удаляет TXT-запись `_acme-challenge.<domain>` через Cloudflare API (`cf_api_token` + `cf_zone_id`), позволяя получить доверенный сертификат даже при закрытом 80 порту или REALITY на `:443`.
- **Автопродление (`acme_auto_renew`) и кэширование X.509 (`read_cert_info_cached`)**: фоновый цикл раз в 12 часов проверяет срок действия (`days_left <= 30`) для публичных доменных имён (IP-адреса и `localhost` автоматически пропускаются) и обновляет сертификат горячей подменой без обрыва соединений. Метаданные X.509-сертификатов кэшируются в памяти функцией `read_cert_info_cached` с автоматической инвалидацией по `(mtime, size)` файла при перевыпуске или обновлении сертификата.
- **Статус слушателей (`GET /api/tls/status`)**: в карточке TLS отдельно отображаются основной порт панели (`Primary`, принимающий HTTPS в режиме `auto`) и выделенный `web_https_port` (`HttpsDedicated`), список актуальных HTTPS URL и явные предупреждения о занятых или конфликтующих портах.

---

### Шаг 3. Изоляция панели и Stealth-настройки

![Мастер: Шаги 3 и 4 — Изоляция пути панели, Stealth-подписки и Self-Audit](../images/ru/12-wizard-stealth-audit.png)

- **`web_base_path` (секретный путь панели)**: при задании префикса (например `/p-8f3c21a9`, генерируется одной кнопкой) корневые маршруты `/`, `/login`, `/api/*` перестают отдавать админ-интерфейс и API. Любой запрос вне `<web_base_path>` обрабатывается только как ACME challenge, Covert Knock, подписка или публичный сайт `SiteEngine`.
  > **Если потеряли секретный URL:** подключитесь к серверу по SSH и выполните `tachyon-panel reset-path` — утилита сбросит `web_base_path` в корень `/` с сохранением бэкапа и перезапустит панель.
- **`enable_3xui_compat_api` (по умолчанию `false`)**: при выключенном флаге маршруты `/panel/api/inbounds/*` отдают `404 Not Found` в стиле публичного сайта, убирая типовые сигнатуры сторонних панелей.
- **`redeem_path`**: отдельный путь для страницы активации ваучеров (при пустом значении и заданном `web_base_path` работает только внутри `<web_base_path>/redeem`).
- **Нейтральный HTTP-профиль**: панель всегда отвечает с `Server: nginx/1.26.2`, корректным заголовком `Date` (RFC-7231), корректными причинами статуса (`404 Not Found`), нейтральной cookie `sid=...; HttpOnly; SameSite=Strict` (с `; Secure` на HTTPS), без `Set-Cookie: 3x-ui` и без глобального `Access-Control-Allow-Origin: *`. Параметр `/?decoy=1` никогда не раскрывает интерфейс панели.

---

### Шаг 4. Covert Knock и Self-Audit (`0–100`)

После настройки нажмите **Запустить Self-Audit** (`POST /api/web/self-audit`). Сервер выполняет **9 автоматических проверок** (включая живую внутреннюю пробу `StealthSubGate` на несуществующий `sub_id`) и выставляет итоговый балл от `0` до `100` (грейды от `A+` до `D`):

| ID проверки | Критичность | Что проверяется | Как набрать балл |
| :--- | :--- | :--- | :--- |
| `tls_active` | `critical` | Встроенный TLS (`web_tls_mode != "http"`) или внешний TLS-профиль (`reverse_proxy` / `unified_443_front`) с заданным доменом (не `localhost`), плюс валидный неистёкший SAN-сертификат в `HotCertResolver` | Выберите режим `auto`/`https` и выпустите сертификат, либо используйте `reverse_proxy` / `unified_443_front` с доменом и валидным SAN-сертификатом. |
| `web_base_path_isolated` | `critical` | Задан непустой секретный `web_base_path` (отличный от `/`) | Сгенерируйте префикс (например `/p-...`) на Шаге 3 и сохраните закладку. |
| `public_site_active` | `high` | Настроен шаблон `SiteEngine` (или загружен `www/index.html` для `custom_static`), ответ на `/` не содержит `window.__PANEL_BASE__` и `tachyon_session` | Выберите шаблон или загрузите статический сайт на Шаге 1. |
| `compat_3xui_disabled` | `high` | `enable_3xui_compat_api == false` | Оставьте совместимый API `/panel/api/inbounds/*` выключенным. |
| `covert_knock_enabled` | `medium` | Включён `covert_knock_enabled == true` | Включите на Шаге 4 для скрытой выдачи подписок через публичный сайт. |
| `sub_block_scanners` | `high` | Включён `sub_block_scanners == true` | Включите фильтр сетевых сканеров и краулеров на Шаге 3. |
| `sub_path_obfuscated` | `high` | `sub_path_style != "sub"` (или нестандартный `sub_path`) И отключён `sub_legacy_alias == false` | Переключите стиль на `static_feed` / `cdn_v2` / `rss_updates` (или свой путь) и отключите старый `/sub/`. |
| `sub_portal_guard_secret` | `high` | `sub_portal_guard == "secret_key"` | Включите защиту браузерного портала ключом `?k=` (или Telegram WebApp). |
| `zero_oracle_neutral_headers` | `critical` | Живая внутренняя проба `StealthSubGate` для несуществующей подписки возвращает `ServeDecoy404` HTML `404 Not Found` с `Server: nginx/1.26.2` без JSON-оракулов | Работает автоматически при соблюдении нейтрального HTTP-профиля. |

---

[Назад: Глава 1 — Установка, Docker, сборка и CLI](01-installation-and-cli.md) | [К оглавлению](../../README.ru.md) | [Далее: Глава 3 — Подписки, веб-портал, ваучеры и Tachyon Covert Knock](03-subscriptions-and-covert-knock.md)
