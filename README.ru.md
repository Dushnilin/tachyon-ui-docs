# Tachyon UI

[English](README.md) | **Русский**

Лёгкая, полностью автономная панель управления движком `tachyon-core` на чистом Rust: единый статический бинарник без C-зависимостей и внешнего рантайма, встроенный веб-интерфейс в тёмной инженерной теме Graphite Dark, двухпротокольный HTTP+HTTPS веб-стек, нативный ACME-менеджер, система маскировки под публичный сайт, 4-уровневая защита подписок с Covert Knock, встроенный Telegram-бот с Mini App и полный паритет с балансировщиками и диагностикой `tachyon-core`.

---

## Ключевые возможности (версия `20261123-v20`)

| Подсистема | Возможности |
| :--- | :--- |
| **Стек без C-зависимостей** | Единый бинарник (`x86_64-unknown-linux-musl`, ~5,5 МБ) со встроенным SPA `src/ui.html`. TLS построен на 100% pure-Rust (`rustls`, `tokio-rustls`, `oxitls-rustcrypto-provider`, без `openssl-sys`/`ring`). |
| **Dual HTTP + HTTPS и SiteEngine** | Одновременный приём HTTP и HTTPS на одном порту через инспекцию первого байта (`0x16`), поддержка выделенных `:80` + `:443`, горячий ребинд портов и сертификатов. 6 встроенных многостраничных шаблонов сайтов или загрузка собственного `.zip` / HTML. |
| **Домены, ACME и REALITY Self-Steal** | 4 готовых профиля привязки (`dual_single_port`, `native_80_443`, `reality_self_steal` на `127.0.0.1:8443`, `reverse_proxy` Nginx/Caddy). Выпуск и автопродление ACME HTTP-01 / Cloudflare DNS-01 / SAN Self-Signed. |
| **Изоляция панели и Self-Audit** | Строгий префикс `web_base_path` (запросы вне пути не раскрывают панель), нейтральные заголовки `nginx/1.26.2`, `sid` cookie, активный Honeypot, автоматический аудит скрытности (0–100 баллов). |
| **4-уровневые Stealth-подписки и Covert Knock** | Полиморфные URL (`sub`, `static_feed`, `cdn_v2`, `rss_updates`), Zero-Oracle 404 HTML, фильтр автоматических сканеров, Portal Guard (`?k=`) и HMAC-стук (`If-None-Match` / `X-Client-Digest`) поверх обычных маршрутов сайта. |
| **Балансировщики `tachyon-core`** | Все 12 режимов `urltest`, расширенный объект `balancer` (`pool`, `pool_tolerance`, `sticky_hash`, `quota_mib`, `tiers`), `selector` (`auto_failover`, живое переключение) и многохоповые цепочки `chain` (`strip_evasion`, `strip`, `rewrite`). |
| **Протоколы и маршрутизация** | VLESS/REALITY, VMess, Trojan, Hysteria/Hy2, TUIC, AnyTLS, Naive, ShadowTLS, WireGuard/AWG (port hopping), MASQUE, MTProto (`telegram-ws`/`kws`), WDTT, Sudoku, Cloudflared, SSH, Tor, Bridge; `time_window`, `rule_set`, `udp_nat_mode`, `smart_queue`. |
| **Кластер, Telegram-бот и диагностика** | Мультинодовая синхронизация и каскады, встроенный бот с тарифами, ваучерами (`/redeem`) и Mini App, 5-вкладочная живая диагностика ядра, режим скрытия приватных данных (`Alt+P`) и палитра команд (`Ctrl+K`). |

---

## Быстрый старт

### Установка в одну команду (Ubuntu / Debian / RHEL / Alpine Linux)

```bash
bash <(curl -Ls https://raw.githubusercontent.com/Dushnilin/tachyon-ui/main/install.sh)
```

Скрипт устанавливает бинарник `tachyon-ui`, сервис `systemd`, консольную утилиту `tachyon-panel`, открывает порт панели в фаерволе и выводит адрес и временный пароль входа (после входа их сразу нужно сменить в разделе Настройки).

### Запуск в Docker

```bash
docker build -t tachyon-ui .
docker run -d --name tachyon-ui --restart unless-stopped \
  -p 2053:2053 -p 443:443 \
  -v /opt/tachyon-ui/data:/data \
  tachyon-ui
```

Подробнее о флагах установки, сборке `musl` из исходников и CLI-командах (`tachyon-panel info`, `reset-path`, `open-web-ports`) — в [главе 1](docs/ru/01-installation-and-cli.md).

---

## Интерфейс панели

![Обзор панели управления tachyon-ui](docs/images/ru/02-dashboard.png)

| Мастер сайта, домена, HTTP/HTTPS и защиты | Конструктор 12 балансировщиков, `tiers` и `chain` |
| :---: | :---: |
| ![Мастер сайта и домена](docs/images/ru/10-wizard-site.png) | ![Конструктор балансировщиков](docs/images/ru/06-balancer-builder.png) |

---

## Подробная документация

1. [Установка, Docker, сборка и CLI `tachyon-panel`](docs/ru/01-installation-and-cli.md)
2. [Публичный сайт, Dual HTTP+HTTPS, домен, ACME и Self-Audit](docs/ru/02-stealth-web-domain-tls.md)
3. [Подписки, веб-портал, ваучеры и Tachyon Covert Knock](docs/ru/03-subscriptions-and-covert-knock.md)
4. [Балансировщики (12 режимов `urltest`), `chain` и маршрутизация](docs/ru/04-balancers-chain-and-routes.md)
5. [Протоколы, мультиноды, каскады и Telegram-бот](docs/ru/05-protocols-nodes-and-tgbot.md)
6. [Диагностика ядра, скрытие приватных данных (`Alt+P`) и REST API](docs/ru/06-diagnostics-privacy-and-api.md)
