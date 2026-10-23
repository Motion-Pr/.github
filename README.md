# Motion Pr

Единый пайплайн производства видео: **Сценарий → Озвучка → Промпты / Инфографика / Футажи → Генерация видео (Veo)**.

## Проекты и где они живут

| Репозиторий | Что делает | Сервер / путь |
|---|---|---|
| **`fl-frontend`** | Единый Web UI (`/prompts`, `/voice`, `/graphics`, `/batch`, `/history`) | Linux (`89.127.192.164:3100`, `/opt/fl-frontend`) |
| **`fl-api`** | Основной Backend API: генерация видео/картинок (Google Flow / Veo 3.1), озвучка (`/api/voice`), архивы R2 | **Windows** (`130.17.13.222:8000`, `C:\fl-api` — воркер Veo + Dolphin Anty)<br>**Linux** (`89.127.192.164:8001`, `/opt/fl-api` — R2 / история / фоллбэк) |
| **`fl-scenario`** | Генератор сценариев `.docx` по теме/брифу (Claude / Gemini) | Windows / локально (`script-generator`) |
| **`fl-voice_translator`** | Перевод сценариев на 7 языков + озвучка Fish Audio (S2.1 / Drama 3) | Встроен в `fl-api` (`/api/voice`) + отдельный бот |
| **`hdoc-prompt`** | Генерация раскадровки `.xlsx` (промпты для Veo) и инфографики из `.docx` (Gemini) | Linux (`89.127.192.164`, `/opt/hdoc-prompts`, Docker `hdoc-prompts-bot-1`) |
| **`motion-aign`** | Рендер анимированных карточек `.mp4`/`.mov` (Remotion + Gemini: люди, битвы, гербы) | Linux (`89.127.192.164`, `/opt/motion-aign`, `motion-aign-bot.service`) |
| **`fl-footage`** | Подбор и нарезка реальных кадров (стоки, архивы, YouTube) под сценарий | Отдельный модуль / бот |
| **`flow-logs`** | Дашборд статистики и логов генерации (`flowstat`) | Linux (`89.127.192.164:8080`, `/opt/flow-logs`, Docker `flow-logs-app-1`) |

## Как взаимодействуют

```text
[fl-scenario] (.docx)
      │
      ├──► [fl-voice_translator / fl-api] ──► Перевод (7 языков) + Озвучка (.mp3, Fish S2.1 / Drama 3)
      ├──► [motion-aign]                  ──► Анимированные карточки (.mp4 / .mov)
      ├──► [fl-footage]                   ──► Реальные архивные / стоковые кадры
      └──► [hdoc-prompt]                  ──► Таблица промптов (.xlsx)
                                                    │
                                                    ▼
                    [fl-frontend] ──► [fl-api (Windows + Dolphin Anty)] ──► Генерация сцен Veo 3.1
                                                    │
                                                    ├──► Cloudflare R2 (готовые .mp4 / .zip)
                                                    └──► [flow-logs] (мониторинг аккаунтов и скорости)
```
