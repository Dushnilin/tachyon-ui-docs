# Bilingual Documentation & Privacy Mask Refinement Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to execute this plan cleanly and verify every deliverable.

**Goal:** Build complete RU and EN GitHub documentation for `tachyon-ui` v20 with 28 live UI screenshots (`1440x900`) and refine the privacy mask so public test domains/DNS resolvers stay readable.

**Architecture:** Fix `maskPrivateText` / `_isSensitiveInput` in `src/ui.html` first so both the app and screenshot `14-privacy-mask` reflect private-only masking, spin up a disposable local `tachyon-ui` instance seeded with synthetic RFC-5737 / `example.com` records to capture 14 RU + 14 EN PNGs via Chrome DevTools Protocol, then author mirrored Markdown guides in `README*.md` and `docs/{ru,en}/01..06.md` verified by automated link/content checks.

**Tech Stack:** Rust/Tokio (`tachyon-ui`), Vanilla JS SPA (`src/ui.html`), Node.js v24 (built-in `fetch` + `WebSocket`), Headless Chrome CDP, Markdown + Mermaid.

**Spec:** `docs/superpowers/specs/2026-11-23-bilingual-docs-design.md`

## Global Constraints
- Neutral engineering terminology only; no emojis anywhere in Markdown or UI changes.
- Zero real server IPs, credentials, tokens, or subscription URLs in docs or screenshots; synthetic RFC-5737 / `example.com` only.
- Clean teardown of temporary demo state, Chrome profile, and scratch helpers; no writes to live VPS `48.102.165.41` or router `192.168.1.1`.

## Tasks
- [ ] **Task 1 (`src/ui.html`)**: Exclude public test/probe domains and public DNS resolvers (`1.1.1.1`, `8.8.8.8`, `9.9.9.9`, `gstatic.com`, `example.com`, `cloudflare.com`, etc.) from `maskPrivateText` and diagnostic inputs from `_isSensitiveInput`, verify `vm.Script`, `cargo build`, and `cargo test`.
- [ ] **Task 2 (`docs/images/{ru,en}/01..14.png`)**: Capture 28 verified `1440x900` PNGs via headless Chrome CDP against a seeded local synthetic instance (with in-page live runtime mock payloads for `/api/core/*` views) and inspect key PNGs visually.
- [ ] **Task 3 (`README.ru.md`, `docs/ru/01..06.md`)**: Write complete Russian documentation covering installation/CLI, 4-step Stealth Web/Domain/HTTP+HTTPS/ACME wizard & Self-Audit, 4-level subscriptions & Covert Knock, 12 `urltest` modes + `chain` + routes, all protocols/nodes/TG bot, and diagnostics/privacy/API.
- [ ] **Task 4 (`README.md`, `docs/en/01..06.md`)**: Write full mirrored English documentation linking `docs/images/en/*.png`.
- [ ] **Task 5 (Verification)**: Audit all relative links, 28 PNG headers/sizes, zero forbidden terms/emojis, zero leaked secrets, `vm.Script`, `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, and `cargo test`.
