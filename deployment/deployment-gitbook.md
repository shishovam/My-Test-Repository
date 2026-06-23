# Инструкция по развёртыванию

Развёртывание выполняется на платформе Cloudflare: backend — Worker, БД — D1, frontend — Pages. Все шаги выполняются через **Wrangler CLI** (можно запускать через агент в ZCode / Claude Code).

## 1. Предварительные требования

- Node.js 18+
- Git
- Аккаунт Cloudflare (бесплатный)
- Wrangler CLI:
```bash
npm install -g wrangler
```

## 2. Авторизация в Cloudflare

```bash
wrangler login
```
Откроется браузер — подтвердите доступ. Это единственный шаг, который нельзя автоматизировать агентом: вход подтверждает человек.

Проверка:
```bash
wrangler whoami
```

## 3. Создание базы данных D1

```bash
cd backend
wrangler d1 create epl-tracker
```

Команда выведет блок с `database_id`. Скопируйте его в `wrangler.toml`:

```toml
name = "epl-tracker-api"
main = "src/index.js"
compatibility_date = "2026-01-01"

[[d1_databases]]
binding = "DB"
database_name = "epl-tracker"
database_id = "ВСТАВЬТЕ_СЮДА_ID"
```

## 4. Применение схемы и начальных данных

```bash
# Схема (таблицы + индексы) — в удалённую БД
wrangler d1 execute epl-tracker --remote --file=./schema.sql

# Команды АПЛ
wrangler d1 execute epl-tracker --remote --file=./seed.sql
```

Учётная запись администратора создаётся отдельным скриптом (хеш пароля считается через Web Crypto API). Пример проверки данных:
```bash
wrangler d1 execute epl-tracker --remote --command="SELECT COUNT(*) FROM teams;"
```
Ожидаемо: 20 команд.

## 5. Настройка секретов

JWT-секрет хранится в защищённом хранилище Cloudflare, не в коде:
```bash
wrangler secret put JWT_SECRET
```
Введите значение секрета в терминал (его вводит человек, агент не должен знать значение).

Для локальной разработки создайте файл `backend/.dev.vars` (он в `.gitignore`):
```
JWT_SECRET=локальный-секрет-для-разработки
```

## 6. Деплой backend (Worker)

```bash
cd backend
npm install
wrangler deploy
```

После деплоя Worker будет доступен по адресу вида:
```
https://epl-tracker-api.<ваш-аккаунт>.workers.dev
```
Проверка:
```bash
curl https://epl-tracker-api.<ваш-аккаунт>.workers.dev/api/health
```

## 7. Деплой frontend (Pages)

### Способ 1: через Wrangler
```bash
cd frontend
wrangler pages deploy . --project-name=epl-tracker
```

### Способ 2: через GitHub-интеграцию (авто-деплой)
1. Cloudflare Dashboard → Workers & Pages → Create → Pages → Connect to Git
2. Выберите репозиторий
3. Настройки сборки:
   - Build command: оставить пустым (статический сайт)
   - Build output directory: `frontend`
4. Deploy

Frontend будет доступен по адресу:
```
https://epl-tracker.pages.dev
```

### Важно: указать адрес API
В `frontend/config/config.js` пропишите URL вашего Worker:
```js
const config = {
  API_URL: 'https://epl-tracker-api.<ваш-аккаунт>.workers.dev/api'
};
```

### Важно: CORS
В Worker (`hono/cors`) разрешите домен frontend:
```js
app.use('/api/*', cors({ origin: 'https://epl-tracker.pages.dev' }));
```

## 8. Публичные адреса и домен

После деплоя проект уже публично доступен:
- Frontend: `https://epl-tracker.pages.dev`
- Backend: `https://epl-tracker-api.<ваш-аккаунт>.workers.dev`

Оба адреса бесплатные, с HTTPS, доступны из любой точки мира. **Покупка собственного домена не требуется.**

Опционально (для портфолио) можно привязать собственный домен:
1. Купить домен (Cloudflare Registrar — по себестоимости) или использовать имеющийся
2. Pages → Custom domains → добавить домен
3. DNS настроится автоматически, если домен в Cloudflare

## 9. Обновление приложения

```bash
# Изменили код backend
cd backend && wrangler deploy

# Изменили frontend
cd frontend && wrangler pages deploy . --project-name=epl-tracker
# (или просто git push, если настроена GitHub-интеграция)
```

## 10. Проверка развёртывания (smoke-тест)

1. Открыть `https://epl-tracker.pages.dev`
2. Войти с учётными данными администратора
3. Выбрать тур, добавить тестовый матч
4. Убедиться, что таблица пересчиталась
5. Удалить тестовый матч
6. Выйти из системы

## 11. Лимиты бесплатного тарифа Cloudflare

| Сервис | Бесплатный лимит |
|--------|------------------|
| Workers | 100 000 запросов/день |
| D1 | 5 ГБ хранилища (до 500 МБ на базу на free), 5 млн строк чтения/день, 100 тыс. записи/день |
| Pages | безлимитный трафик, 500 сборок/месяц |

Для данного проекта (десятки команд, до 380 матчей в сезон) этих лимитов хватает с огромным запасом. Важное преимущество: на бесплатном тарифе Cloudflare проекты **не приостанавливаются и не удаляются** за неактивность.

## 12. Troubleshooting

**Worker не отвечает / ошибка D1 binding:**
- Проверьте `database_id` в `wrangler.toml`
- Убедитесь, что схема применена: `wrangler d1 execute ... --command="SELECT name FROM sqlite_master WHERE type='table';"`

**CORS-ошибка в браузере:**
- Проверьте, что `origin` в `hono/cors` совпадает с адресом frontend

**401 при запросах:**
- Проверьте, что `JWT_SECRET` задан через `wrangler secret put`
- Убедитесь, что клиент отправляет заголовок `Authorization: Bearer <token>`

**Не сохраняются изменения локально:**
- При `wrangler dev` без `--remote` используется локальная D1; данные удалённой и локальной БД различаются
