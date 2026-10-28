# Motion Pr — Памятка разработчика

Единый пайплайн видеопроизводства: **Сценарий → Озвучка → Промпты / Инфографика / Футажи → Генерация видео (Veo 3.1)**.

---

## 1. Куда что лить (репозитории)

| Если ты меняешь... | Лить в репозиторий | Ветка | Примечание |
|---|---|---|---|
| **Веб-интерфейс** (все вкладки `/script`, `/analytics`, `/prompts`, `/voice`, `/graphics`, `/batch`, `/history`) | **`Motion-Pr/fl-frontend`** | `main` | Next.js 16 (App Router) |
| **Backend API**: генерация Veo/Imagen, очередь батчей, R2-архивы, промпты (`/api/prompts*`) и инфографика (`/api/infographics*`) | **`Motion-Pr/fl-api`** | `main` | FastAPI. Крутится на Windows (воркер Veo `:8000`) и на Linux (инфографика/промпты/R2-фоллбэк `:8001`) |
| **Генератор сценариев**: веб-эндпоинты (`/api/scenario/*`), воркер очереди, аналитика тем, паспорт ниши, `.docx` по теме/брифу | **`Motion-Pr/fl-scenario`** | `main` | Полностью самостоятельный сервис (`api/` на порту `:8020`, `generate.py worker`, `bot.py`). Не зависит от `fl-api` |
| **Перевод и озвучка**: веб-эндпоинты (`/api/voice/*`), воркер очереди, бот и CLI для `.docx` на 7 языков + Fish S2.1 / Drama 3 | **`Motion-Pr/fl-voice_translator`** | `main` | Полностью самостоятельный сервис (`api/` на порту `:8010`, `worker.py`, `bot.py`). Не зависит от `fl-api` |
| **Промпты раскадровки (`.xlsx`) и стат-инфографика** по сценарию | **`Motion-Pr/hdoc-prompt`** | `main` | Python + Gemini API (HTTP API + Telegram-бот) |
| **Анимированные карточки** (Remotion: люди, битвы, гербы `.mp4`/`.mov`) | **`Motion-Pr/motion-aign`** | `main` | Remotion + Python (`scenario/`, `bot/`) |
| **Подбор реальных футажей** (стоки, архивы, YouTube под абзацы) | **`Motion-Pr/fl-footage`** | `main` | Автономный модуль / бот |
| **Дашборд логов и аналитики** (`flowstat`: статус аккаунтов, 429, скорость) | **`Motion-Pr/flow-logs`** | `main` | Принимает события `/ingest` от `fl-api` |

---

## 2. Где что крутится (серверы)

### Linux-сервер (`89.127.192.164`) — Web UI, прокси, инфографика, боты, R2
- **Входной прокси (Caddy, порт `:80`)** — конфиг `/etc/caddy/Caddyfile`:
  - Все запросы UI `/*` → `127.0.0.1:3100` (`fl-frontend`)
  - `/api/infographics*` и `/api/prompts*` → локальный Linux **`127.0.0.1:8001`** (`fl-api` + Remotion)
  - `/api/voice/*` → **Windows `130.17.13.222:8010`** (`fl-voice_translator` web API)
  - `/api/scenario/*` → **Windows `130.17.13.222:8020`** (`fl-scenario` web API)
  - Остальные `/api/*` → сначала на **Windows `130.17.13.222:8000`**, а если Windows недоступен — автоматический фоллбэк на локальный **`127.0.0.1:8001`** (`fl-api-fallback`).
- **`/opt/fl-frontend`** — сервис `fl-frontend.service` (порт `127.0.0.1:3100`).
- **`/opt/fl-api`** — сервис `fl-api-fallback.service` (порт `127.0.0.1:8001`, рендерит инфографику Remotion, генерирует промпты, раздаёт историю из R2 и ZIP-архивы).
- **`/opt/motion-aign`** — сервис `motion-aign-bot.service` (рендер карточек Remotion).
- **`/opt/hdoc-prompts`** — Docker-контейнер `hdoc-prompts-bot-1` (`docker compose up -d --build`).
- **`/opt/flow-logs`** — Docker-контейнер `flow-logs-app-1` (порт `:8080`, дашборд `flowstat`).

### Windows-сервер (`130.17.13.222`) — Veo (`fl-api`), Озвучка (`fl-voice_translator`), Сценарии (`fl-scenario`)
- **`C:\fl-api`** (порт `0.0.0.0:8000`, автозапуск `fl-api`) — основной `fl-api`, который управляет профилями **Dolphin Anty** (`127.0.0.1:3001`), получает токены reCAPTCHA Enterprise в браузере и генерирует видео/картинки через Google Flow (`aisandbox-pa.clients6.google.com`).
  - Запуск вручную с логами в PowerShell:
    ```powershell
    cd C:\fl-api
    git pull
    $env:LOG_LEVEL="INFO"; uv run python scripts/serve.py --host 0.0.0.0 --port 8000
    ```
- **`C:\Users\Administrator\script-translator v31`** — **`fl-voice_translator`**:
  - Веб-сервер озвучки `/api/voice/*`: порт `0.0.0.0:8010` (`start_api.bat` / Scheduled Task `fl-voice-api`).
  - Воркер очереди перевода (`agy.exe`) и синтеза речи (Fish Audio): `start_worker.bat` / Scheduled Task `fl-voice-worker`.
- **`C:\Users\Administrator\script-generator`** — **`fl-scenario`**:
  - Веб-сервер сценариев `/api/scenario/*`: порт `0.0.0.0:8020` (`start_api.bat` / Scheduled Task `fl-scenario-api`).
  - Воркер очереди сценариев, брифов, паспортов ниш и аналитики (`claude` / `agy.exe`): `start_worker.bat` / Scheduled Task `fl-scenario-worker`.

---

## 3. Схема взаимодействия

```text
[fl-scenario (:8020)] (.docx)
      │
      ├──► [fl-voice_translator (:8010)]  ──► Перевод (7 яз.) + Озвучка (.mp3, Fish S2.1 / Drama 3)
      ├──► [motion-aign / fl-api (:8001)] ──► Анимированные карточки (.mp4 / .mov)
      ├──► [fl-footage]                   ──► Реальные архивные / стоковые кадры
      └──► [hdoc-prompt / fl-api (:8001)] ──► Таблица промптов (.xlsx)
                                                    │
                                                    ▼
                    [fl-frontend] ──► [fl-api (Windows :8000 + Dolphin Anty)] ──► Генерация сцен Veo 3.1
                                                    │
                                                    ├──► Cloudflare R2 (готовые .mp4 / .zip)
                                                    └──► [flow-logs :8080] (мониторинг аккаунтов)
```

---

## 4. CI/CD и правила выката в прод

Во всех репозиториях настроен GitHub Actions (`.github/workflows/ci-cd.yml`):

1. **Никаких сломанных сборок в `main`**:
   - В работу берём ветку `feat/<название>` или `fix/<название>` и открываем **Pull Request** в `main`.
   - На каждый пуш/PR автоматически запускается CI:
     - `fl-api`: проверка синтаксиса + прогон всех тестов `uv run pytest -v`.
     - `fl-frontend`: `npx next typegen && npx tsc --noEmit` + полная прод-сборка `npm run build`.
     - `motion-aign`: проверка TypeScript (`tsc --noEmit`) + компиляция Python (`bot`, `scenario`).
     - `hdoc-prompt`: проверка зависимостей и компиляции Python.
2. **Автодеплой только после зелёных тестов (`needs: test`)**:
   - Шаг `deploy-linux` запускается **только** при пуше/мерже в `main` и **только если все тесты прошли успешно**.
   - Для активации SSH-автодеплоя в Settings → Secrets → Actions репозитория задаётся секрет `DEPLOY_SSH_KEY` (приватный SSH-ключ от `root@89.127.192.164`).
