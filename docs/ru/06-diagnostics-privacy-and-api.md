# 6. Диагностика ядра, скрытие приватных данных (`Alt+P`) и REST API

[English version](../en/06-diagnostics-privacy-and-api.md) | [К оглавлению](../../README.ru.md)

## Живая 5-вкладочная консоль диагностики ядра

Кнопка **Соединения** в топбаре или **Живая матрица соединений** на дашборде открывает модальное окно диагностики в реальном времени (`tachyon-core` Runtime API):

![Консоль диагностики ядра tachyon-core](../images/ru/13-core-diagnostics.png)

| Вкладка консоли | Функции и эндпоинты ядра |
| :--- | :--- |
| **1. Соединения и маршрут** | Встроенный живой трассировщик `POST /api/core/route-check`, проверка маршрута домена `POST /api/core/route-domain`, фильтр активных сессий по клиенту/домену/IP/протоколу (`GET /api/core/connections`), принудительное закрытие отдельных потоков или всех потоков пользователя (`POST /api/core/connections/close`, `POST /api/core/kick`), оперативная блокировка домена (`POST /api/core/block-domain`), мониторинг и очистка памяти ядра (`GET /api/core/memory`, `POST /api/core/memory-trim`). |
| **2. Health & Аномалии** | Телеметрия состояния ядра (`GET /api/core/health`: аптайм, RSS-память, число открытых файловых дескрипторов, RTT, success rate, состояние Circuit-Breaker по каждому outbound) и журнал аномалий (`GET /api/core/anomalies`). |
| **3. Failover & Прокси** | Состояние групп отказоустойчивости (`GET /api/core/failover`, принудительный возврат `POST /api/core/failover/restore`), живые группы балансировщиков (`GET /api/core/proxies`) с переключением активного узла в `selector` в реальном времени, массовым тестом задержек (`POST /api/core/proxies/delay`) и сбросом Circuit-Breaker (`POST /api/core/outbound/reset`). |
| **4. DNS & Leak Audit** | Сравнительная проверка DNS (`POST /api/core/dns-check` — встроенный движок vs прямой UDP:53 на предмет подмены или bogon), очистка DNS-кэша (`POST /api/core/dns/flush`) и тест утечек (`GET / POST /api/core/leak`: IPv4, IPv6, DNS, WebRTC). |
| **5. Тест & In-Flight** | 8-этапный тест подключения к любому URL/хосту (`POST /api/core/connection-test`) и просмотр/очистка буфера незавершённых запросов ядра (`GET / DELETE /api/core/flight`). |

### Нижний Live Connections Dock (`Ctrl+\``)

Для постоянного контроля трафика без перекрытия текущей таблицы предусмотрен нижний выдвижной док **Live Connections Dock** (`Ctrl+\`` или меню инструментов топбара). Он показывает поток активных сессий в реальном времени, позволяет фильтровать домены/клиентов, проверять маршрут (`route-domain`) и блокировать домен в 1 клик (`block-domain`).

---

## Режим скрытия приватных данных (`Alt+P`)

![Режим скрытия приватных данных Alt+P](../images/ru/14-privacy-mask.png)

Тумблер **Скрыть данные** в правом верхнем углу топбара (дублируется горячей клавишей `Alt+P`, пунктом в палитре `Ctrl+K` и настройкой в `#settings`; состояние запоминается в `localStorage`) позволяет безопасно записывать скринкасты, делать скриншоты и проводить удалённые демонстрации панели.

**Что маскируется (`***`):**
- Все приватные IPv4/IPv6-адреса и нестандартные порты (`198.51.100.21:443` → `***.***.***.**:****`).
- Приватные доменные имена сервера и нод.
- Ссылки и токены подписок (`/sub/...`, `/cdn/v2/...`, `?k=...`, `?v=...`, заголовки `If-None-Match` и `X-Client-Digest`).
- UUID, X25519/WireGuard/SS2022 ключи, пароли, HWID устройств, токены и ID Telegram-бота.
- QR-коды (размываются, чтобы ссылку нельзя было считать с экрана).
- Поля `<input>` / `<textarea>` с секретными значениями маскируются визуально (через SVG-паттерн поверх текста) **без изменения фактического `.value`**, поэтому формы продолжают работать и копирование в буфер копирует реальные значения.

**Что остаётся видимым и читаемым:**
- Общеизвестные публичные тестовые/проверочные домены и URL (`https://www.gstatic.com/generate_204`, `https://cp.cloudflare.com/generate_204`, `example.com`, `youtube.com`, `google.com`, `cloudflare.com`, `github.com`, `microsoft.com`, `telegram.org` и др.).
- Публичные DNS-резолверы (`1.1.1.1`, `1.0.0.1`, `8.8.8.8`, `8.8.4.4`, `9.9.9.9`) и `0.0.0.0`.
- Поля ввода в диагностических инструментах, трассировщиках маршрутов, симуляторе и строке `urltest`-проверки.

---

## Командная палитра (`Ctrl+K`) и горячие клавиши

| Сочетание | Действие |
| :--- | :--- |
| **`Ctrl + K`** (или `Cmd + K`) | Глобальная палитра команд и быстрый поиск по нодам, клиентам, подпискам, темам оформления и действиям (`Self-Audit`, `Hot-Reload`, инвайт в 1 клик, установка `tachyon-lists`). |
| **`Ctrl + \``** (или `Cmd + \``) | Открыть или скрыть нижний живой поток соединений (**Live Connections Dock**). |
| **`Alt + P`** | Мгновенно включить или выключить режим скрытия приватных данных звёздочками. |
| **`Alt + 1 … 8`** | Мгновенное переключение между 8 основными разделами (`Обзор`, `Входящие`, `Исходящие`, `Подписки`, `Ноды`, `Бот`, `Маршруты`, `Настройки`). |
| **`Shift + Клик`** | Выделение диапазона чекбоксов во Входящих, Исходящих и Подписках с вызовом плавающей панели пакетных операций. |
| **`?`** | Открыть встроенную памятку по всем горячим клавишам и жестам. |
| **`Esc`** | Закрыть открытое модальное окно, боковой инспектор или палитру команд. |

---

## Справочник REST API

При настроенном `web_base_path` (например `/p-8f3c21a9`) все маршруты панели и `/api/*` доступны **только** с этим префиксом (`<web_base_path>/api/...`).

### Аутентификация

| Метод и путь | Описание |
| :--- | :--- |
| `POST /api/login` | Вход по `{ "username", "password" }`. Выдаёт нейтральную cookie `sid=...; HttpOnly; SameSite=Strict` (`+ ; Secure` на TLS) и `{ "token", "expires" }` (поддерживается в `Authorization: Bearer <token>`). |
| `POST /api/logout` | Удаляет серверную сессию и очищает cookie `sid` и legacy `tachyon_session`. |

### Основные ресурсы, инструменты, TLS и API ядра

| Группа | Маршруты |
| :--- | :--- |
| **Статус, настройки и снапшоты** | `GET /api/status`, `GET /api/config-preview`, `GET/PUT /api/settings`, `GET /api/backup`, `POST /api/restore`, `GET/POST /api/snapshots`, `POST /api/snapshots/restore`, `GET /api/logs` |
| **Управление ядром и панелью** | `POST /api/core/start \| stop \| restart \| apply`, `GET /api/core/releases`, `POST /api/core/update`, `GET /api/panel/releases`, `POST /api/panel/self-update` |
| **Публичный сайт (`SiteEngine`)** | `GET /api/site/info`, `POST /api/site/upload-zip` (raw ZIP или JSON `zip_base64`), `GET/PUT /api/site/index`, `DELETE /api/site/custom`, `GET /api/site/preview?mode=&path=&title=` |
| **Домен и TLS** | `GET /api/tls/status`, `POST /api/tls/self-signed`, `POST /api/tls/issue-acme`, `POST /api/domain/apply-profile` (включая `unified_443_front`), `GET /api/domain/proxy-snippet` |
| **Self-Audit и безопасность** | `POST /api/web/self-audit` (возвращает `score`, `grade`, 9 `checks`), `GET /api/security/honeypot`, `POST /api/security/clear-honeypot` |
| **Инбаунды, аутбаунды и маршруты** | `GET/POST /api/inbounds`, `PUT/DELETE /api/inbounds/:id`, `POST /api/inbounds/batch-action` (`enable`, `disable`, `delete`, `set_country`), `POST /api/inbounds/traffic-policy`, `GET /api/share/:id`, `GET/POST /api/outbounds`, `POST /api/outbounds/health-check`, `POST /api/outbounds/batch-delete`, `GET/PUT /api/routes`, `POST /api/routes/tachyon-lists-pack`, `POST /api/routes/simulate` |
| **Генераторы и сетевые инструменты** | `POST /api/generate/uuid \| reality-keys \| wg-keys \| ss-key \| password`, `POST /api/tools/sni-scan \| auto-heal \| port-hop \| unlock-probe \| cloudflare-dns` (и диагностика региональной связности в `/api/tools/*`) |
| **Подписки, Covert Knock и ваучеры** | `GET/POST /api/subs`, `POST /api/subs/:id/covert-token`, `POST /api/subs/:id/rotate-secrets`, `POST /api/subs/rotate-token`, `POST /api/subs/clear-strikes`, `POST /api/subs/import-external`, `GET/POST /api/sub-profiles`, `POST /api/subs/apply-profile`, `POST /api/subs/quick-invite`, `POST /api/subs/batch-action`, `POST /api/subs/inspect-headers`, `POST /api/vouchers/batch`, `POST /api/redeem` |
| **Ноды, каскады и Telegram-бот** | `GET/POST /api/nodes`, `POST /api/nodes/sync \| health-check \| multi-clone`, `POST /api/cascade/preview \| /api/cascade`, `GET/PUT /api/tg-bot`, `POST /api/tg-bot/verify \| broadcast \| send-message \| users/adjust \| backup-now \| test-alert`, `POST/DELETE /api/tg-bot/promo \| orders/* \| tickets/*`, `POST /api/billing/issue-sub` |
| **Мост `tachyon-core` Runtime API** | `GET /api/core/connections`, `POST /api/core/connections/close \| kick \| block-domain \| route-domain`, `GET /api/core/proxies`, `POST /api/core/proxies/select \| delay`, `POST /api/core/outbound/reset`, `GET /api/core/health`, `GET /api/core/anomalies`, `GET/POST /api/core/failover`, `POST /api/core/failover/restore`, `POST /api/core/route-check`, `POST /api/core/connection-test`, `POST /api/core/dns-check \| dns/flush`, `GET/POST /api/core/leak`, `GET/DELETE /api/core/flight`, `GET /api/core/memory`, `POST /api/core/memory-trim`, `POST /api/core/warp/register` |

---

[Вернуться к оглавлению](../../README.ru.md)
