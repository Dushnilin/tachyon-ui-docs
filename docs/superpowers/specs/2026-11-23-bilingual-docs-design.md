# Спецификация: Двуязычная документация `tachyon-ui` v20 (RU + EN) со скриншотами

**Дата:** 2026-11-23  
**Статус:** Утверждено пользователем (Вариант 1 — модульная документация для GitHub)

## 1. Цели и область охвата
- Создать полную, структурированную документацию по всем возможностям `tachyon-ui` версии **20261123-v20** на русском и английском языках.
- Сопроводить разделы наглядными скриншотами `1440x900` (отдельно для русского и английского интерфейса: 14 RU + 14 EN = 28 PNG).
- Исключить маскировку общеизвестных публичных тестовых доменов и публичных DNS-адресов в режиме скрытия приватных данных (`Alt+P`), чтобы скрывались только пользовательские приватные адреса, ключи, токены и учётные данные.

## 2. Структура файлов документации
- Корневые точки входа:
  - `README.md` (EN) и `README.ru.md` (RU) с переключателем языков, обзором архитектуры (pure-Rust без C-зависимостей, Dual HTTP+HTTPS, Stealth Web + Covert Knock, 12 режимов `urltest` + `chain`, встроенный Telegram-бот), быстрым стартом (`install.sh` / Docker) и оглавлением.
- Модульные руководства (`docs/ru/01..06.md` и зеркальные `docs/en/01..06.md`):
  1. `01-installation-and-cli.md` — установка, Docker, сборка `musl`, автообновление, CLI `tachyon-panel` (`info`, `reset-path`, `open-web-ports`), бэкапы и снапшоты.
  2. `02-stealth-web-domain-tls.md` — Dual HTTP+HTTPS, пайплайн запроса (Mermaid), 4-шаговый мастер сайта/домена/TLS/ACME (`SiteEngine`, загрузка `.zip`, 4 профиля привязки домена включая `REALITY Self-Steal`, изоляция `web_base_path`, 9 критериев `Self-Audit`).
  3. `03-subscriptions-and-covert-knock.md` — подписки, ваучеры `/redeem`, веб-портал, HWID, клиентские форматы и 4-уровневая защита (`sub_path_style`, Zero-Oracle 404, фильтр сканеров, Portal Guard, Covert Knock HMAC-SHA256).
  4. `04-balancers-chain-and-routes.md` — все 12 режимов `urltest`, объект `balancer` (`pool`, `pool_tolerance`, `sticky_hash`, `quota_mib`, `tiers`), `selector` (`auto_failover`), цепочки `chain` (`strip_evasion`, `strip`, `rewrite`), правила маршрутизации (`time_window`, `udp_nat_mode`, `smart_queue`) и живой симулятор маршрутов.
  5. `05-protocols-nodes-and-tgbot.md` — входящие/исходящие протоколы (VLESS/REALITY, VMess, Trojan, Hysteria/Hy2, TUIC, AnyTLS, Naive, ShadowTLS, WireGuard/AWG port hopping, MASQUE, MTProto, WDTT, Sudoku, Cloudflared, SSH, Tor, Bridge), ноды и Telegram-бот + Mini App.
  6. `06-diagnostics-privacy-and-api.md` — 5-вкладочная диагностика ядра (`/api/core/*`), маскировка приватных данных (`Alt+P`), `Ctrl+K` и справочник REST API.
- Изображения: `docs/images/ru/01..14.png` и `docs/images/en/01..14.png`.

## 3. Политика демо-стенда и качества
- Скриншоты снимаются исключительно с временного локального экземпляра `tachyon-ui` на `127.0.0.1` с полностью синтетическими RFC-5737 (`198.51.100.x`, `203.0.113.x`) и `example.com` данными.
- В документации и на скриншотах запрещена нестандартная жаргонная терминология, эмодзи и реальные реквизиты инфраструктуры.
