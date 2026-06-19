# Hydra — серверна підсистема музичного сервісу

**Кваліфікаційна робота бакалавра**  
**Тема:** Сервіс для підбору та прослуховування аудіофайлів. Бекенд.

| | |
|---|---|
| **Автор** | [Богдан Рикуш](https://github.com/NureRykushBohdan) |
| **Група** | ПЗПІ-22-5 |
| **Спеціальність** | 121 — Інженерія програмного забезпечення |
| **Освітня програма** | Програмна інженерія |
| **Університет** | [Харківський національний університет радіоелектроніки](https://nure.ua/) |
| **Факультет** | Комп'ютерних наук |
| **Кафедра** | Програмної інженерії |
| **Керівник** | ст. викл. каф. ПІ Катерина Зибіна |
| **Рік** | 2026 |

---

## Про проєкт

**Hydra** — серверна підсистема програмного комплексу для роботи з медіаконтентом: підбір треків, прослуховування, бібліотека плейлистів, рекомендації та інтеграція з Telegram. Клієнтський веб-застосунок (UI) розробляється співвиконавцем і споживає HTTP API, описаний у цій роботі.

Робота охоплює проєктування та реалізацію стійкої до навантаження серверної частини:

- REST API на **FastAPI**
- фонові **воркери** з чергою на **Redis Streams**
- реляційне сховище метаданих у **PostgreSQL**
- завантаження та видача файлів через **Cloudflare R2**
- контейнеризація та розгортання (**Docker Compose**, **Caddy**)
- автоматизоване тестування (**pytest**, CI на GitHub Actions)

## Архітектура

```
Клієнт (Web / Telegram) ──► Caddy (TLS) ──► FastAPI (api/)
                                                │
                    ┌───────────────────────────┼───────────────────────────┐
                    ▼                           ▼                           ▼
              PostgreSQL                     Redis                    Cloudflare R2
                    ▲                           ▲
                    └──────── Worker (worker/) ─┘
                              yt-dlp, проксі, Telegram Bot API
```

**Принцип:** API-first + async-offload — швидкі запити обробляє HTTP-шар, тривалі операції (завантаження, синхронізація з R2) виконують воркери через Redis Streams.

### Основні компоненти

| Каталог / сервіс | Призначення |
|---|---|
| `api/` | FastAPI-додаток, REST-роутери, стрімінг |
| `worker/` | Фонова обробка черги: yt-dlp, R2, оновлення БД |
| `common/` | Спільні моделі, конфігурація, Redis, ORM |
| `bot/` | Telegram-бот (aiogram) |
| `deploy/` | Конфігурація Caddy |
| `tests/` | Модульні та інтеграційні тести (135 сценаріїв) |

## Технологічний стек

| Категорія | Технології |
|---|---|
| Мова | Python 3.x |
| HTTP API | FastAPI, Uvicorn, Pydantic Settings |
| База даних | PostgreSQL, SQLAlchemy (asyncpg) |
| Черги / кеш | Redis Streams |
| Об'єктне сховище | Cloudflare R2 (S3-сумісний API) |
| Завантаження медіа | yt-dlp |
| Месенджер | Telegram Bot API, aiogram |
| Інфраструктура | Docker Compose, Caddy (HTTPS) |
| Якість коду | ruff, pytest, pytest-asyncio, GitHub Actions |

## API

Документація OpenAPI доступна після запуску на `/docs`.

| Група | Приклади ендпоінтів |
|---|---|
| Системні | `GET /health`, `GET /ready` |
| Треки | `GET /api/track/{youtube_id}` |
| Пошук | `GET /api/search`, `POST /api/recognize` |
| Метадані | `GET /api/metadata/...`, `/api/album/...` |
| Бібліотека | `/api/library/playlists...` |
| Залученість | `/api/favorites...`, `/api/recommendations...` |
| Користувач | `GET /api/user/me`, `PATCH /api/user/settings` |
| Адмін | `/api/admin/*` |
| Стрімінг | `/api/stream/...` (HTTP Range, 206 Partial Content) |

### Авторизація

- **Користувачі:** заголовок `X-Telegram-Data` (перевірка підпису Telegram WebApp initData)
- **Адміністратор:** заголовок `X-Admin-Key`
- **Пошук (службовий):** заголовок `X-Search-Api-Key`

## Розгортання

### Вимоги

- Docker і Docker Compose
- VPS або локальне середовище
- Зовнішній інстанс PostgreSQL
- Обліковий запис Cloudflare (R2)
- Файл `.env` з секретами (не комітити в репозиторій)

### Запуск

```bash
# Базовий стек
docker compose up -d

# Публічний контур з TLS (Caddy)
docker compose --profile public up -d

# Додаткові воркери
docker compose --profile extra-workers up -d
```

### Перевірка після розгортання

1. `docker compose ps` — стан контейнерів
2. `GET /health` — liveness
3. `GET /ready` — readiness (PostgreSQL + Redis)
4. Smoke-тест ключових маршрутів (`/api/track/*`, `/api/stream/*`, `/api/search`)

### Основні змінні оточення

| Змінна | Опис |
|---|---|
| `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` | Підключення до PostgreSQL |
| `REDIS_URL` | Підключення до Redis |
| `R2_PUBLIC_URL`, `R2_*` | Cloudflare R2 |
| `PROXY_URL` | Проксі для yt-dlp (можна кілька URI) |
| `BOT_TOKEN` | Telegram Bot API |
| `ADMIN_API_KEY` | Ключ адмін-API |
| `CORS_ORIGINS` | Дозволені origins для CORS |

## Тестування

```bash
pytest -q
ruff check tests
```

CI (`.github/workflows/ci.yml`) виконує lint, pytest і збірку Docker-образу на кожен push/PR.

## Структура репозиторію

```
.
├── README.md                          # цей файл
├── 2026_Б_ПІ_ПЗПІ-22-5_Рикуш_Б_Ю.zip  # архів матеріалів кваліфікаційної роботи
└── api/, worker/, common/, bot/       # вихідний код серверної підсистеми
    deploy/, tests/, docker-compose.yml
```

> Повний текст пояснювальної записки (37 стор., 14 рис., 9 табл.) — у архіві репозиторію та в файлі `2026_бакалавр_КН_Пр_ПЗПІ-22-5_Рикуш Б.Ю._.pdf_ звіт з практики.txt`.

## Результати роботи

- Спроєктовано багатошарову архітектуру (HTTP API, дані, воркери, інфраструктура)
- Реалізовано REST API з модульними роутерами та стрімінгом з підтримкою HTTP Range
- Забезпечено узгодження метаданих PostgreSQL з об'єктами в Cloudflare R2
- Реалізовано воркери з yt-dlp, проксі та оновленням БД
- Підготовлено сценарій розгортання через Docker Compose і Caddy
- 135 автоматизованих тестів, CI з ruff і pytest

## Наукова публікація

Стаття за матеріалами роботи: [UNIVERSUM](https://archive.liga.science/index.php/universum/issue/view/may2026/187)

## Ліцензія

Матеріали кваліфікаційної роботи наведено для освітніх та архівних цілей. Використання вихідного коду — згідно з умовами, визначеними автором.

## Контакти

**Богдан Рикуш**  
GitHub: [@NureRykushBohdan](https://github.com/NureRykushBohdan)
