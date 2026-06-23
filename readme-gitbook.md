# EPL Results Tracker

Веб-приложение для учёта результатов матчей Английской премьер-лиги сезона 2026/2027 с автоматическим расчётом турнирной таблицы. Система построена на serverless-архитектуре Cloudflare с edge-хостингом и облачной базой данных.

## 🎯 Основные функции

- Управление 38 турами чемпионата с возможностью добавления матчей в каждом туре
- Автоматический расчёт турнирной таблицы в реальном времени
- Добавление, редактирование и удаление результатов с мгновенным обновлением статистики
- Валидация вводимых данных (счёт ≥ 0, команда не играет сама с собой, уникальность матча в туре)
- Защищённый доступ с авторизацией по логину и паролю (JWT)
- Единый рабочий экран после входа: селектор тура, список матчей, турнирная таблица

## 🛠 Технологический стек

### Frontend
- **Язык:** JavaScript (ES6+)
- **UI Framework:** Bootstrap 5
- **Хостинг:** Cloudflare Pages

### Backend
- **Среда выполнения:** Cloudflare Workers (edge, serverless)
- **Framework:** Hono
- **Аутентификация:** JWT (`hono/jwt`)
- **Хеширование паролей:** Web Crypto API (PBKDF2 / SHA-256)

### База данных
- **СУБД:** Cloudflare D1 (SQLite)
- **Доступ:** нативный D1 binding (prepared statements)

## ☁️ Почему Cloudflare

- Бесплатный тариф не «усыпляет» и не удаляет проекты за неактивность — важно для долгоживущего портфолио
- Edge-выполнение без cold start
- Бесплатные публичные адреса `*.pages.dev` и `*.workers.dev` с HTTPS
- Единый провайдер для frontend, backend и БД — упрощает деплой

## 🚀 Быстрый старт

### Предварительные требования
- Node.js 18+
- Git
- Аккаунт Cloudflare (бесплатный)
- Wrangler CLI: `npm install -g wrangler`

### Локальная разработка
```bash
# Клонировать репозиторий
git clone https://github.com/shishovam/epl-results-tracker.git
cd epl-results-tracker

# Backend (Worker)
cd backend
npm install
wrangler dev          # локальный запуск Worker + локальная D1

# Frontend (в отдельном терминале)
cd frontend
npx serve .           # или любой статический сервер
```

### Продакшн
Полная инструкция — в [deployment](deployment/deployment-gitbook.md).

## 📚 Документация

- [Требования к проекту](requirements/requirements-gitbook.md)
- [Архитектура приложения](architecture/architecture-gitbook.md) — C4-диаграммы
- [Структура данных](data-structure/data-structure-gitbook.md) — схема D1 и API
- [Руководство пользователя](user-guide/user-guide-gitbook.md)
- [Инструкция по развёртыванию](deployment/deployment-gitbook.md)
- [Руководство разработчика](development/development-gitbook.md)

## 📁 Структура проекта

```
epl-results-tracker/
├── frontend/                  # Cloudflare Pages
│   ├── index.html             # Главный экран (после входа)
│   ├── login.html             # Страница входа
│   ├── css/
│   │   └── styles.css
│   ├── js/
│   │   ├── api.js             # Клиент REST API
│   │   ├── auth.js            # Логин, хранение JWT
│   │   ├── rounds.js          # Селектор и матчи тура
│   │   └── table.js           # Турнирная таблица
│   └── config/
│       └── config.js          # API_URL и настройки
│
├── backend/                   # Cloudflare Worker
│   ├── src/
│   │   └── index.js           # Hono-приложение: роуты, auth, логика
│   ├── schema.sql             # Схема и индексы D1
│   ├── seed.sql               # Начальные данные (команды, admin)
│   ├── wrangler.toml          # Конфигурация Cloudflare
│   └── package.json
│
└── README.md
```

## 👥 Команда

- **Alex Shishov** — администрирование, аналитика, DevOps
- **Yana Shishova** — тестирование
- **ai.model** — разработка кода (через агентную среду ZCode / Claude Code)

## 📄 Лицензия

MIT
