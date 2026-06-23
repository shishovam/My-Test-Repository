# Руководство разработчика

## 1. Структура проекта

```
epl-results-tracker/
├── frontend/                  # Cloudflare Pages (статика)
│   ├── index.html             # Главный экран после входа
│   ├── login.html             # Страница входа
│   ├── css/
│   │   └── styles.css
│   ├── js/
│   │   ├── api.js             # Клиент REST API (Fetch + JWT)
│   │   ├── auth.js            # Логин, хранение токена, выход
│   │   ├── rounds.js          # Селектор туров и матчи
│   │   └── table.js           # Турнирная таблица
│   └── config/
│       └── config.js          # API_URL
│
├── backend/                   # Cloudflare Worker
│   ├── src/
│   │   └── index.js           # Hono-приложение целиком
│   ├── schema.sql             # Схема D1
│   ├── seed.sql               # Команды АПЛ
│   ├── wrangler.toml          # Конфигурация Cloudflare
│   ├── package.json
│   ├── .dev.vars              # Локальные секреты (в .gitignore!)
│   └── .gitignore
│
└── README.md
```

## 2. Технологический стек

| Слой | Технология |
|------|-----------|
| Frontend | Vanilla JS (ES6+), Bootstrap 5, Fetch API |
| Хостинг frontend | Cloudflare Pages |
| Backend | Hono на Cloudflare Workers |
| Аутентификация | JWT (`hono/jwt`), пароль — PBKDF2 (Web Crypto API) |
| База данных | Cloudflare D1 (SQLite), D1 binding |
| Инструменты | Wrangler CLI, Node.js 18+, Git |

> Важно: в среде Workers нет полноценного Node.js runtime. Поэтому Express **не используется** — вместо него Hono. Модули вроде `bcrypt` или `pg` тоже недоступны: пароль хешируется через встроенный Web Crypto API, а к БД обращаемся через D1 binding.

## 3. Backend: организация кода

Весь Worker — в одном файле `src/index.js`. Базовый каркас:

```js
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { jwt, sign } from 'hono/jwt';

const app = new Hono();

// CORS — только домен frontend
app.use('/api/*', cors({ origin: 'https://epl-tracker.pages.dev' }));

// Health (без авторизации)
app.get('/api/health', (c) => c.json({ status: 'ok' }));

// Логин (без авторизации)
app.post('/api/auth/login', async (c) => { /* проверка пароля, выдача JWT */ });

// JWT middleware — на всё остальное под /api
app.use('/api/*', (c, next) => {
  // пропускаем health и login
  const open = ['/api/health', '/api/auth/login'];
  if (open.includes(c.req.path)) return next();
  return jwt({ secret: c.env.JWT_SECRET })(c, next);
});

// Защищённые маршруты
app.get('/api/teams', async (c) => { /* ... */ });
app.get('/api/matches/round/:round', async (c) => { /* ... */ });
app.post('/api/matches', async (c) => { /* ... */ });
app.put('/api/matches/:id', async (c) => { /* ... */ });
app.delete('/api/matches/:id', async (c) => { /* ... */ });
app.get('/api/table', async (c) => { /* расчёт таблицы */ });

export default app;
```

### 3.1 Хеширование пароля (Web Crypto API)

```js
async function hashPassword(password, saltHex) {
  const enc = new TextEncoder();
  const salt = saltHex
    ? Uint8Array.from(saltHex.match(/.{2}/g).map(b => parseInt(b, 16)))
    : crypto.getRandomValues(new Uint8Array(16));

  const key = await crypto.subtle.importKey(
    'raw', enc.encode(password), 'PBKDF2', false, ['deriveBits']
  );
  const bits = await crypto.subtle.deriveBits(
    { name: 'PBKDF2', salt, iterations: 100000, hash: 'SHA-256' },
    key, 256
  );
  const hashHex = [...new Uint8Array(bits)]
    .map(b => b.toString(16).padStart(2, '0')).join('');
  const outSalt = [...salt]
    .map(b => b.toString(16).padStart(2, '0')).join('');
  return { hash: hashHex, salt: outSalt };
}
```

При логине: берём `password_salt` из БД, хешируем введённый пароль с этой солью, сравниваем с `password_hash`.

### 3.2 Выдача JWT

```js
const token = await sign(
  { sub: user.id, username: user.username, exp: Math.floor(Date.now()/1000) + 86400 },
  c.env.JWT_SECRET
);
```

### 3.3 Доступ к D1

```js
// Чтение
const { results } = await c.env.DB
  .prepare('SELECT * FROM teams ORDER BY sort_order').all();

// Запись с параметрами
await c.env.DB
  .prepare('INSERT INTO matches (round, home_team_id, away_team_id, home_score, away_score, match_date) VALUES (?, ?, ?, ?, ?, ?)')
  .bind(round, homeId, awayId, hs, as, date)
  .run();
```

## 4. Расчёт турнирной таблицы

Таблица не хранится в БД, а считается при запросе `GET /api/table`:

```js
// 1. Загрузить все команды и все матчи
// 2. Инициализировать статистику для каждой команды нулями
// 3. Пройти по матчам: начислить P/W/D/L, GF/GA, очки
// 4. Вычислить GD = GF - GA
// 5. Отсортировать: очки ↓, разница ↓, забитые ↓, имя ↑
// 6. Проставить позиции
```

Правила: победа +3, ничья +1, поражение 0.

## 5. Frontend: организация

- `config/config.js` — единственное место с `API_URL`
- `js/api.js` — обёртка над Fetch: добавляет `Authorization: Bearer`, при 401 редиректит на `login.html`
- `js/auth.js` — форма входа, сохранение/удаление JWT в `localStorage`
- `js/rounds.js` — селектор туров 1–38, рендер матчей, формы (модальные окна Bootstrap)
- `js/table.js` — запрос `/api/table`, рендер таблицы, вызывается после любого изменения матчей

Главный экран (`index.html`) состоит из трёх секций: селектор тура, список матчей выбранного тура с кнопкой «Добавить», турнирная таблица.

## 6. Локальная разработка

```bash
cd backend
npm install
wrangler dev        # запускает Worker локально + локальную D1
```

Применить схему к локальной БД:
```bash
wrangler d1 execute epl-tracker --local --file=./schema.sql
wrangler d1 execute epl-tracker --local --file=./seed.sql
```

Frontend — любой статический сервер:
```bash
cd frontend
npx serve .
```
Не забудьте временно указать в `config.js` локальный адрес Worker (обычно `http://localhost:8787/api`).

## 7. Git-гигиена

В `.gitignore` обязательно:
```
node_modules/
.dev.vars
.env
.wrangler/
*.log
```

Никогда не коммитьте `.dev.vars`, значения секретов и `database_id` со скриншотов. `database_id` в `wrangler.toml` — не секрет, но менять его на чужой нельзя.

## 8. Добавление новой функции (пример)

**Добавить фильтр «только сыгранные матчи»:**
1. Backend: добавить query-параметр в `GET /api/matches` и условие в SQL
2. Frontend: добавить переключатель в UI, передавать параметр в `api.js`
3. Перерисовать список в `rounds.js`

## 9. Тестирование

- **Unit** (расчёт таблицы, валидация): функции вынести так, чтобы их можно было тестировать изолированно; запускать через Vitest
- **API**: коллекция запросов (REST-клиент / Postman) ко всем endpoints, проверка статус-кодов и схемы ответа
- **E2E / ручное**: сценарии входа, CRUD матчей, пересчёта таблицы

Подробности — в задачах тестирования (роль Yana).

## 10. Возможные улучшения

### Краткосрочные
- [ ] Поиск по командам
- [ ] Подсветка зон (еврокубки / вылет) в таблице
- [ ] Тёмная тема

### Долгосрочные
- [ ] PWA / offline-режим
- [ ] Графики (Chart.js)
- [ ] AI-функции: предсказание результатов, генерация комментариев, чат-бот по статистике

## 11. Полезные ресурсы

- [Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Cloudflare D1](https://developers.cloudflare.com/d1/)
- [Cloudflare Pages](https://developers.cloudflare.com/pages/)
- [Hono](https://hono.dev/)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/)
- [Bootstrap 5](https://getbootstrap.com/docs/5.3/)
- [Web Crypto API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API)
