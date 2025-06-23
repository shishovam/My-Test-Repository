# EPL Results Tracker

Веб-приложение для учета результатов матчей Английской премьер-лиги с автоматическим расчетом турнирной таблицы. Система построена на современной клиент-серверной архитектуре с облачным хранилищем данных.

## 🎯 Основные функции

- Управление 38 турами чемпионата с возможностью добавления до 10 матчей в каждом туре
- Автоматический расчет турнирной таблицы в реальном времени
- Редактирование и удаление результатов с мгновенным обновлением статистики
- Валидация вводимых данных (счет от 0 до 100)
- Защищенный доступ с JWT авторизацией
- Восстановление пароля через email

## 🛠 Технологический стек

### Frontend
- **Язык:** JavaScript (ES6+)
- **UI Framework:** Bootstrap 5
- **Хостинг:** Netlify

### Backend
- **Язык:** Node.js
- **Framework:** Express.js
- **Аутентификация:** JWT
- **Хостинг:** Railway.app или Render.com

### База данных
- **СУБД:** PostgreSQL
- **Хостинг:** Supabase (500MB бесплатно) или Neon.tech (3GB бесплатно)

## 🚀 Быстрый старт

### Локальная разработка
```bash
# Клонировать репозиторий
git clone https://github.com/your-username/epl-results-tracker.git

# Frontend
cd frontend
npm install
npm run dev

# Backend (в отдельном терминале)
cd backend
npm install
npm run dev
```

### Продакшн
Следуйте инструкции в [DEPLOYMENT.md](deployment/README.md)

## 📚 Документация

- [Требования к проекту](requirements/README.md)
- [Архитектура приложения](architecture/README.md) - включает C4 диаграммы
- [Структура данных](data-structure/README.md) - схема БД и API
- [Руководство пользователя](user-guide/README.md)
- [Инструкция по развертыванию](deployment/README.md)
- [Руководство разработчика](development/README.md)

## 📁 Структура проекта

```
epl-results-tracker/
├── frontend/               # Клиентская часть
│   ├── index.html         
│   ├── css/
│   │   └── styles.css     
│   ├── js/
│   │   ├── app.js         # Инициализация
│   │   ├── api.js         # API клиент
│   │   ├── auth.js        # Авторизация
│   │   ├── rounds.js      # Управление турами
│   │   └── table.js       # Турнирная таблица
│   └── config/
│       └── config.js      # Конфигурация
│
├── backend/               # Серверная часть
│   ├── src/
│   │   ├── server.js      # Express сервер
│   │   ├── routes/        # API маршруты
│   │   ├── controllers/   # Контроллеры
│   │   ├── models/        # Модели данных
│   │   ├── middleware/    # Middleware
│   │   └── utils/         # Утилиты
│   ├── migrations/        # Миграции Б
