# FlipCTL — архитектура (сжатая версия)

> Полные версии: [`frontend.md`](./frontend.md), [`backend.md`](./backend.md). Этот файл — выжимка для онбординга.

**Цель:** удобный доступ к инструментам Linux (сеть, systemd, процессы) через pixel-perfect интерфейс небольшого экрана с огранниченным набором физических кнопок.

## Общая архитектура

```mermaid
flowchart TB
    WEB["**Web браузер**<br> или локальный Cog/WPE"]
    TUI["**TUI**<br>Go + bubbletea, локально/SSH"]
    REMOTE["**Remote HW**"]

    CADDY["**Caddy**<br>статика Web-сборки + reverse proxy /api/*"]

    subgraph CORE["**flipctld** — backend-ядро (Go)"]
        API["HTTP + SSE API"]
        REG["Plugin Registry"]
        JM["Job Manager"]
        BUS["Event Bus"]
    end

    subgraph PLUGINS["Плагины — отдельные процессы, любой язык"]
        SYS["system: wifi, power, ..."]
        COM["community: ping, nmap, ..."]
    end

    subgraph OS["Linux (Raspberry Pi 4)"]
        DBUS["D-Bus: NetworkManager, systemd"]
        BIN["CLI-утилиты"]
    end

    WEB --> CADDY -- "HTTP/SSE" --> API
    REMOTE -- "SSH" --> TUI
    TUI -- "HTTP/SSE" --> API
    API --> REG
    API --> JM
    JM -- "spawn + NDJSON/stdio" --> PLUGINS
    PLUGINS -- события --> BUS --> API
    SYS --> DBUS
    COM --> BIN
```

Ключевые идеи:
- **общий контракт, вместо общего кода.** Frontend (Web и TUI) и backend — три независимых, идиоматичных для своей платформы стека, связанных API/схемами (`contract/`), а не общим рантаймом.
- **Система пользовательских и ситемных плагинов в лёгкой Go обертке**. Плагины можно писать на любом подходящем языке или использовать готовые механизмы Линукса типа D-bus, libs (libcurl).
- **Основа GUI выполнена на web технологиях**. Легко разрабатывать, дешево поддерживать.
- **Web и TUI используют общий API к бекэнду, но не шарят между собой UI логику и ассеты**. Имеют раздельные движки для рендеринга.
- **Рендеринг на экран устройства осуществляется из headless браузера через Linux DRM**. Нет возни с драйверами, небольшой футпринт памяти. Полноценный веб-интерфейс в качестве бонуса.

## Frontend

- **Web** — React + `react-dom`, рендер в один `<canvas>` (`PixelSurface`/`CanvasSurface`) ради pixel-perfect 1-bit стиля прототипа. Не используем Yoga пытаясь объединить UI с TUI. Вместо этого используем фиксированные пиксельные константы, как в `fake-FlipCTL`.
- **TUI** — Go + `bubbletea`/`lipgloss`/`bubbles`. Тот же язык, что backend — реальная синергия общих типов (`contract/go/apitypes`). Не pixel-perfect, обычный текстовый UI. Компилируемый бинарник, экономия ресурсов на портативном SBC. SSH — через forced-command системного `sshd` (или `wish` из того же стека `bubbletea`).
- Общее между Web и TUI:
  - семантика ввода (`InputAction`: Up/Down/Ok/Back/SoftKey1/...);
  - модель навигации (стек экранов + отдельный overlay-стек);
  - реестр приложений — приходит из backend registry manager, не хранится во frontend.
- Типичный вызов плагина `GET /api/registry` в рантайме:
  ```jsonc
  { "id": "ping", "kind": "generic",
    "inputs": [{ "id": "target", "type": "text" }],
    "actions": [{ "id": "run", "endpoint": "/api/plugins/ping/run", "bind": "slot:2" }] }
  ```
  - `kind: "custom"` → написанный вручную экран (Wi-Fi, Ethernet, Power).
  - `kind: "generic"` → один общий компонент (`GenericActionScreen`/`generic_action.go`) для простых CLI-wrapper плагинов, без platform-специфичного кода на каждый новый плагин.

## Backend

- **`flipctld` (Go)** — тонкий супервизор. Не хранилище бизнес-логики. Заменяет `server.js` из `fake-FlipCTL`. Реализует API в то время, как статику Web-сборки и внешние обращения берёт на себя отдельный `Caddy`. Перед ним (reverse proxy `/api/*`), не сам `flipctld`.
- **Плагины** — отдельные OS-процессы. Можно использовать любой язык (единственное требование — читать/писать JSON). Подключаютсячерез небольшую обёртку на Go. Упакованы вместе с манифестом, бинарниками, иконками, декларативным описанием UI. Можно делить и классифицировать их по разным параметрам.

  - По уровню доступа `tier`:
    - `system` (Wi-Fi, power, cron, ...) — обязательный базовый минимум. Написаны и проверены командой Flipper.
    - `community` (ping, nmapб curl, ...) — пользовательские.
  - По типу запуска `execution`:
    - `one-shot` — запустился/отработал/вышел. `whoami`.
    - `stream` — живёт, шлёт данные и события. `ping`
    - `daemon`  — живёт независимо от клиентов, большую часть времени чё-то ждёт. Например система уведомлений.
  - По типу UI `ui_type`:
    - `custom` — собственный уникальный UI, который представлен настоящими ассетами в `frontend/UI`.
    - `generic` — использует примитивы из дефолтового UI-kit: list, grid, MenuBar, softKeys, dropDown, ...

- **IPC** — NDJSON поверх stdin/stdout:
  ```json
  {"type":"start","inputs":{"target":"8.8.8.8"}}
  {"type":"output","fields":{"status":"reachable","rtt_ms":13.2}}
  {"type":"done","exit_code":0}
  ```
- **Job / Session / Event Bus** — job общий для всех клиентов с самого начала: кто угодно может подписаться на уже бегущий job через SSE (Web и TUI видят один и тот же живой поток одновременно).

  ```mermaid
  stateDiagram-v2
      [*] --> Running: spawn OK
      Running --> Completed: done
      Running --> Stopped: stop_job
      Completed --> [*]
      Stopped --> [*]
  ```

- **Button input в плагинах**: у action в манифесте есть `bind` (`slot:0..4` — позиция в панели экрана, либо `input:back` — универсальное действие) и раздельные `on_press`/`on_release` с эффектом `start_job | stop_job | send_event`. `send_event` шлёт именованное событие в stdin уже бегущего процесса, не спавнит новый.
- **System-плагины могут (и должны) использовать D-Bus** (NetworkManager/systemd напрямую) вместо простого парсинга вывода CLI через regex.
- **Привилегии (временно отложены)**: пока все плагины равнодоверенные. Кандидат на будущее — D-Bus policy + **polkit**, тот же стек, что использует сам NetworkManager; работает только если у каждого плагина свой D-Bus-identity (плагин сам держит клиента, не `flipctld` от его имени).

## Контракт
- `contract/openapi.yaml` — HTTP/SSE API (`/api/registry`, `/api/plugins/{id}/{action}`, `/api/jobs/*`).
- `contract/manifest.schema.json` — JSON Schema манифеста плагина.
- `contract/ipc-messages.schema.json` — схема NDJSON-сообщений `flipctld` ↔ плагин.
- Формализовано, чтобы риск ручной рассинхронизации типов закрывался кодогенерацией: backend и TUI (оба на Go) используют один сгенерированный пакет `contract/go/apitypes` напрямую, Web (TypeScript) — отдельно генерирует TS-типы из той же схемы.

## Notes
- Был вариант сначала рисовать TUI и потом из него генерировать web — отброшен т.к. не добиться pixel perfect картинки на экране (в TUI мы оперируем вертикальной ячейкой, а не квадратным пикселем), а так же тянет за собой nodejs и WASM.
- Я бы наметил critical path и сделал быстрый прототип для проверки гипотез этой архитектуры. В нём бы вовсе не было ветки с TUI.
- Нужно менять или дорабатывать (или просто глубже ресёрчить) COG т.к. сейчас есть проблемы с его запуском в чистом DRM на реальном железе.
- Backend = **Go** (`bubbletea`) специально ради синергии — backend и TUI используют общий сгенерированный пакет типов напрямую. Bubbletea и вся его экосистема — старый, хорошо опробованный проект.
- Sandboxing плагинов — отложен.
- Система прав плагинов — отложено.
- Caddy как отдельный процесс выбран сознательно ради низкого порога входа и независимого релиза Web-фронтенда — цена: второй резидентный процесс на SBC.
