# CLAUDE.md — контекст проекта для AI-агента

Этот файл задаёт правила и контекст для AI-агента (ZCode / Claude Code), работающего над проектом. Читай его перед выполнением любой задачи.

## Что за проект

**EPL Results Tracker** — веб-приложение для учёта результатов матчей Английской премьер-лиги сезона 2026/2027 с автоматическим расчётом турнирной таблицы. Один пользователь (администратор), ручной ввод результатов.

## Технологический стек (СТРОГО соблюдать)

- **Frontend:** Vanilla JavaScript (ES6+), Bootstrap 5, Fetch API. Хостинг — Cloudflare Pages.
- **Backend:** Hono на Cloudflare Workers. НЕ Express, НЕ Node.js-специфичные модули.
- **Аутентификация:** JWT через `hono/jwt`. Пароль хешируется через Web Crypto API (PBKDF2/SHA-256). НЕ использовать `bcrypt`.
- **База данных:** Cloudflare D1 (SQLite). Доступ через binding `DB` и prepared statements. НЕ использовать `pg`, НЕ PostgreSQL-синтаксис.
- **Инструменты:** Wrangler CLI, Node.js 18+, Git.

## Чего НЕ делать

- НЕ использовать Express, `pg`, `bcrypt`, `jsonwebtoken` (вместо последнего — `hono/jwt`).
- НЕ использовать PostgreSQL-специфичный SQL (например `gen_random_uuid()`, `SERIAL`, `TIMESTAMP WITH TIME ZONE`). Это SQLite.
- НЕ реализовывать восстановление пароля / email — этого в проекте нет.
- НЕ коммитить секреты: `.dev.vars`, `.env`, значения JWT-секрета. Они должны быть в `.gitignore`.
- НЕ хранить пароль в открытом виде в seed-файлах.
- НЕ хранить турнирную таблицу в БД — она считается на сервере из таблицы `matches`.

## Структура репозитория

```
epl-results-tracker/
├── frontend/                  # Cloudflare Pages
│   ├── index.html             # Главный экран (после входа)
│   ├── login.html             # Вход
│   ├── css/styles.css
│   ├── js/{api,auth,rounds,table}.js
│   └── config/config.js       # API_URL
├── backend/                   # Cloudflare Worker
│   ├── src/index.js           # Hono-приложение целиком
│   ├── schema.sql             # Схема D1
│   ├── seed.sql               # Команды АПЛ
│   ├── wrangler.toml
│   ├── package.json
│   ├── .dev.vars              # локальные секреты (в .gitignore)
│   └── .gitignore
└── README.md
```

## Схема БД (SQLite / D1)

Таблицы: `users` (id, username, password_hash, password_salt, is_active, created_at), `teams` (id TEXT PK, name, short_name, sort_order), `matches` (id, round 1..38, home_team_id, away_team_id, home_score≥0, away_score≥0, match_date, created_at, updated_at). Ограничения: `home_team_id <> away_team_id`, `UNIQUE(round, home_team_id, away_team_id)`. Полная схема — в `backend/schema.sql` и в документе data-structure.

## API (все, кроме /api/health и /api/auth/login, требуют JWT)

- `POST /api/auth/login` — вход, возвращает JWT
- `GET /api/teams`
- `GET /api/matches` / `GET /api/matches/round/:round`
- `POST /api/matches` / `PUT /api/matches/:id` / `DELETE /api/matches/:id`
- `GET /api/table` — расчёт таблицы на сервере
- `GET /api/health`

## Правила расчёта таблицы

Победа +3, ничья +1, поражение 0. Сортировка: очки ↓, разница мячей ↓, забитые ↓, имя команды ↑.

## Команды АПЛ 2026/2027 (20)

ARS, AVL, BOU, BRE, BHA, CHE, COV, CRY, EVE, FUL, HUL, IPS, LEE, LIV, MCI, MUN, NEW, NFO, SUN, TOT. Полный seed — в `backend/seed.sql` и в документе data-structure.

## Стиль работы

- Делай задачи маленькими шагами, показывай результат для ревью.
- Валидируй данные и на клиенте, и на сервере.
- Используй понятные сообщения об ошибках в едином формате JSON: `{ "success": false, "error": { "code": "...", "message": "..." } }`.
- После изменения матчей фронтенд должен запрашивать `/api/table` заново.

## Документация

Подробности — в папках `requirements/`, `architecture/`, `data-structure/`, `deployment/`, `development/`, `user-guide/`. При расхождении кода и документации — спроси, не додумывай.
