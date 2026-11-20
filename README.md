# Kittygram

Приложение для публикации фотографий котов.
Пользователи могут регистрироваться, создавать карточки котов
и просматривать публикации.

Проект: https://kitty2gram.ddns.net/

Taski на той же ВМ: https://taski25698.duckdns.org/

## Автор

Автор проектной работы — [barbarastep](https://github.com/barbarastep).

В рамках работы настроены контейнеризация, CI/CD и развёртывание проекта.

Репозиторий: [kittygram_final](https://github.com/barbarastep/kittygram_final).

Исходный код приложения предоставлен Яндекс Практикумом.

## Технологии

Python 3.12, Django, Django REST Framework, Gunicorn,
React, PostgreSQL 13, Nginx, Docker Compose и GitHub Actions.

## Структура проекта

- `backend/` — Django API и Dockerfile бэкенда.
- `frontend/` — React-приложение и Dockerfile фронтенда.
- `nginx/` — Dockerfile и конфигурация внутреннего Nginx.
- `docker-compose.yml` — локальная сборка и запуск.
- `docker-compose.production.yml` — запуск готовых образов на сервере.
- `.env.example` — пример переменных окружения.
- `.github/workflows/main.yml` — workflow GitHub Actions.
- `kittygram_workflow.yml` — копия workflow для проверки задания.
- `tests.yml` — данные для проверочных тестов.
- `tests/` — проверочные тесты учебного проекта.

## Контейнеры и тома

В Docker Compose настроены четыре сервиса:

- `db` — PostgreSQL 13.
- `backend` — Django API под управлением Gunicorn.
- `frontend` — копирование собранного фронтенда в общий том.
- `gateway` — Nginx: раздача файлов и проксирование запросов к API.

Сервис `frontend` завершается с кодом `0` после копирования файлов.

Данные хранятся в именованных томах:

- `pg_data` — база данных PostgreSQL.
- `static` — фронтенд и статика Django.
- `media` — загруженные фотографии.

## Локальный запуск на Mac

Необходимы Git и запущенный Docker Desktop с Docker Compose.
Все команды этого раздела выполняются в терминале Mac.

### 1. Подготовка

```bash
git clone https://github.com/barbarastep/kittygram_final.git
cd kittygram_final
```

Если репозиторий уже клонирован, перейдите в его директорию.

При первом запуске создайте `.env`:

```bash
cp .env.example .env
```

В `.env` замените `POSTGRES_PASSWORD` на собственный пароль
базы данных, а `SECRET_KEY` — на случайный секретный ключ Django.

Для генерации ключа:

```bash
openssl rand -hex 32
```

Для локального запуска оставьте:

```dotenv
DB_HOST=db
DB_PORT=5432
DEBUG=False
ALLOWED_HOSTS=localhost,127.0.0.1
```

Файл `.env` содержит секреты и не хранится в Git.

### 2. Сборка образов и запуск базы

```bash
docker compose build
docker compose up -d --wait db
```

### 3. Миграции и статика Django

```bash
docker compose run --rm backend python manage.py migrate --noinput
```

```bash
docker compose run --rm backend sh -c 'python manage.py collectstatic --noinput && mkdir -p /static/backend_static && cp -r /app/collected_static/. /static/backend_static/'
```

### 4. Запуск приложения

```bash
docker compose up -d
```

Приложение доступно по адресу http://localhost:9000/.

Для доступа к административной панели создайте суперпользователя:

```bash
docker compose exec backend python manage.py createsuperuser
```

Административная панель: http://localhost:9000/admin/.

### 5. Проверка

```bash
docker compose ps -a
curl -I http://localhost:9000/backend_static/admin/css/base.css
curl -i http://localhost:9000/api/cats/
```

Статика должна возвращать `200 OK`.
API без авторизации возвращает `401 Unauthorized`.

### Остановка

```bash
docker compose down
```

Именованные тома с базой данных, статикой и фотографиями сохраняются.

## CI/CD

Workflow запускается при пуше в `main`.
Также доступен ручной запуск через GitHub Actions.

Этапы:

1. Проверка бэкенда через flake8 и запуск тестов Django.
2. Запуск тестов фронтенда.
3. После успешных тестов — сборка и публикация трёх образов:
   - `barbarastep/kittygram_backend:latest`;
   - `barbarastep/kittygram_frontend:latest`;
   - `barbarastep/kittygram_gateway:latest`.
4. Копирование production Compose-файла на ВМ по SSH.
5. Загрузка образов, выполнение миграций, сборка и перенос статики
   в общий том, пересоздание контейнеров.
6. Проверка доступности сайта и статики по HTTPS.
7. Отправка сообщения об успешном деплое в Telegram.

Образы для сервера собираются для платформы `linux/amd64`.

Уведомление в Telegram отправляется после успешного деплоя.

### GitHub Actions Secrets

| Секрет | Назначение |
| --- | --- |
| `DOCKER_USERNAME` | Логин Docker Hub |
| `DOCKER_PASSWORD` | Токен Docker Hub с правом публикации образов |
| `SSH_HOST` | Публичный IP сервера |
| `SSH_USER` | Пользователь сервера |
| `SSH_PRIVATE_KEY` | Приватный ключ для деплоя |
| `SSH_KNOWN_HOSTS` | Проверенная запись SSH-ключа сервера |
| `TELEGRAM_TOKEN` | Токен Telegram-бота |
| `TELEGRAM_TO` | ID чата для уведомлений |

## Серверное окружение

Проект размещён в `/home/ubuntu/kittygram/`.

На сервере установлены Docker, Docker Compose,
внешний Nginx и Certbot. Пользователю деплоя доступен
`sudo` без запроса пароля для команд workflow.

Серверный `.env` создаётся отдельно.
В `ALLOWED_HOSTS` добавлен домен `kitty2gram.ddns.net`.

Production Compose использует готовые образы из Docker Hub.
Внутренний gateway доступен на `127.0.0.1:9000`.

Внешний Nginx принимает запросы к домену Kittygram
и передаёт их на этот порт.
HTTPS настроен через Certbot и сертификат Let's Encrypt.

Taski работает отдельно на той же ВМ, на порту `9001`.

## Проверочные тесты задания

Данные проектов указаны в корневом `tests.yml`.
Для сетевых проверок оба проекта должны быть доступны.

На Mac, из корня репозитория:

```bash
docker compose run --rm --build \
  -v "$PWD:/project:ro" \
  -w /project \
  backend pytest -ra
```

После изменения `.github/workflows/main.yml` необходимо
обновлять его копию `kittygram_workflow.yml` перед сдачей.
