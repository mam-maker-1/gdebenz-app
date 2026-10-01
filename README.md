# Где Бенз? («Where's the Gas?»)

Многопользовательский сервис отзывов о наличии и цене топлива на АЗС России:
интерактивная карта, отзывы с проверкой цен на реалистичность, заявки на
добавление АЗС, роли пользователь/модератор/коренной модератор, избранное,
комментарии, жалобы, REST API со Swagger-документацией.

## 1. Архитектура и технологии

| Слой | Технология | Комментарий |
|---|---|---|
| Backend | **FastAPI** (async), Python 3.12 | Явный выбор вместо Django — задание требует асинхронности и разрешает FastAPI |
| ORM | SQLAlchemy 2.0 (async, `asyncpg`) | Модели в `backend/app/models/` |
| Миграции | Alembic (async env.py) | `backend/alembic/` |
| БД | PostgreSQL 16 + PostGIS 3.4 | Образ `postgis/postgis:16-3.4` |
| Кэш | Redis 7 | Кэширование списков АЗС, TTL настраивается |
| Фронтенд | Jinja2 + Bootstrap 5 + Leaflet.js | Серверный рендеринг страниц, вся динамика — через fetch() к REST API |
| Аутентификация | JWT (access + refresh), bcrypt | Токены в `localStorage`, авто-обновление в `static/js/app.js` |
| Контейнеризация | Docker Compose (app, db, redis, nginx) | `docker-compose up` — единственная команда для запуска |
| Тесты | pytest + pytest-asyncio, SQLite (aiosqlite) | `backend/tests/` |

### Геоданные и PostGIS

Координаты АЗС хранятся как обычные `Float`-колонки (широко переносимо и
удобно тестировать). При этом **радиус-поиск на PostgreSQL выполняется через
функции PostGIS** (`ST_DistanceSphere`, `ST_MakePoint`, `ST_SetSRID` —
см. `backend/app/services/geo.py`), а расширение `postgis` включается прямо
в первой миграции Alembic (`0001_initial.py`). Для юнит- и интеграционных
тестов используется SQLite (без PostGIS) — для него в том же модуле
реализован эквивалентный фолбэк на чистом Python (формула гаверсинуса),
поэтому один и тот же код роутов работает на обеих БД.

### Почему JWT в `localStorage`, а не httpOnly-cookie

Для проекта, где один и тот же REST API должен одинаково обслуживать и
сайт, и (в будущем) мобильное приложение, единая схема Bearer-токенов проще
и предсказуемее, чем смешение cookie-сессий и JWT. Это осознанный
компромисс: `localStorage` уязвим к XSS сильнее, чем httpOnly-cookie. Для
продакшена с повышенными требованиями к безопасности рекомендуется:
перейти на httpOnly + Secure + SameSite=Strict cookie для refresh-токена,
добавить CSRF-токен для form-based запросов, включить чёрный список
отозванных refresh-токенов в Redis.

### Модерация цен

Диапазоны реалистичности (`backend/app/services/price_validation.py`):

| Топливо | Диапазон, руб |
|---|---|
| АИ-92 / АИ-95 / АИ-97 | 30–200 / литр |
| Дизельное топливо | 35–200 / литр |
| Газ | 15–80 / литр |
| Электрозаправка | 10–100 / кВт·ч |

Отзыв с ценой внутри диапазона публикуется мгновенно и сразу обновляет
`StationFuel.current_price` + пишет запись в `PriceHistory`. Отзыв с ценой
вне диапазона получает статус `pending` и ждёт решения модератора
(`/api/moderation/reviews/{id}/approve|reject`). Та же логика применяется к
ценам, указанным в заявке на добавление АЗС.

## 2. Структура проекта

```
gde-benz/
├── docker-compose.yml
├── .env.example
├── nginx/nginx.conf
└── backend/
    ├── Dockerfile, entrypoint.sh, requirements.txt, requirements-dev.txt
    ├── alembic.ini, alembic/{env.py, versions/0001_initial.py}
    ├── pytest.ini
    ├── app/
    │   ├── main.py                  — сборка FastAPI-приложения
    │   ├── config.py                — настройки из .env
    │   ├── database.py              — async SQLAlchemy engine/session
    │   ├── security.py              — bcrypt, JWT
    │   ├── dependencies.py          — get_current_user, проверки ролей
    │   ├── cache.py, email_utils.py, logging_config.py
    │   ├── models/                  — User, GasStation, StationFuel, Review,
    │   │                              PriceHistory, StationRequest, Favorite,
    │   │                              Comment, Complaint
    │   ├── schemas/                 — Pydantic-схемы запросов/ответов
    │   ├── services/                — price_validation, rating, geo,
    │   │                              review_effects, notifications
    │   ├── routers/                 — auth, users, stations, reviews,
    │   │                              station_requests, favorites, comments,
    │   │                              complaints, moderation, pages
    │   ├── templates/                — Jinja2-шаблоны (Bootstrap + Leaflet)
    │   └── static/{css,js}
    ├── scripts/
    │   ├── create_superuser.py      — идемпотентное создание коренного модератора
    │   ├── seed_demo_data.py        — демо-АЗС для быстрого просмотра сайта
    │   └── import_osm_stations.py   — импорт реальных АЗС из OpenStreetMap
    └── tests/                        — pytest (модели, API, валидация, гео)
```

## 3. Источник геоданных

Основной источник — **OpenStreetMap через Overpass API**
(`https://overpass-api.de/api/interpreter`), общедоступные данные под
лицензией ODbL. Скрипт `backend/scripts/import_osm_stations.py` запрашивает
узлы/пути с тегом `amenity=fuel` в заданном регионе (`--bbox` или `--area`)
и создаёт по ним записи `GasStation`. Overpass не хранит актуальные цены —
они появляются позже из отзывов пользователей.

Для регионов, где данных OSM недостаточно, работает ручной путь: форма
«Добавить АЗС» → заявка `StationRequest` → модерация → публикация.

Для быстрой демонстрации без обращения к внешнему API есть
`scripts/seed_demo_data.py` с ~12 демонстрационными АЗС в разных городах
России (координаты приблизительные, только для показа возможностей карты).

## 4. Запуск

```bash
git clone <repo> gde-benz && cd gde-benz
cp .env.example .env
# при желании отредактируйте .env (пароли, SMTP и т.д.)

docker-compose up --build
```

При старте `app`-контейнера `entrypoint.sh` автоматически:
1. дожидается готовности PostgreSQL;
2. применяет миграции Alembic (создаёт все таблицы + расширение PostGIS);
3. создаёт коренного модератора из `SUPERUSER_EMAIL` / `SUPERUSER_PASSWORD`;
4. если `SEED_DEMO_DATA=true` (по умолчанию в `.env.example`) — наполняет
   базу демо-АЗС.

Сайт: **http://localhost:8080** (через Nginx). Прямой доступ к приложению
(без Nginx) — **http://localhost:8000**. Swagger-документация REST API:
**http://localhost:8080/api/docs** (ReDoc — `/api/redoc`).

### Данные для входа коренного модератора

```
email:    admin@example.com      (или значение SUPERUSER_EMAIL из .env)
password: admin12345             (или значение SUPERUSER_PASSWORD из .env)
```

⚠️ Обязательно смените эти значения в `.env` перед реальным продакшн-развёртыванием.

Демо-аккаунты (создаются вместе с демо-данными, `SEED_DEMO_DATA=true`):

```
Модератор:    moderator@example.com / moder12345
Пользователь: driver@example.com    / driver12345
```

### Импорт реальных данных из OSM

```bash
docker-compose exec app python -m scripts.import_osm_stations --area "Москва" --limit 500
docker-compose exec app python -m scripts.import_osm_stations --bbox 55.5,37.3,56.0,37.9 --limit 500
```

### Запуск тестов

```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
pytest -v --cov=app
```

Тесты используют изолированную SQLite-базу (файл в `/tmp`, создаётся заново
перед каждым тестом) — Docker и PostgreSQL для тестов не требуются.

## 5. Переменные окружения

Полный список — в `.env.example`. Ключевые:

| Переменная | Назначение |
|---|---|
| `DATABASE_URL` | Строка подключения к PostgreSQL (async, `postgresql+asyncpg://...`) |
| `JWT_SECRET_KEY`, `SECRET_KEY` | Секреты для подписи токенов — обязательно сменить в проде |
| `SUPERUSER_EMAIL`, `SUPERUSER_PASSWORD` | Учётные данные коренного модератора, создаваемого при первом запуске |
| `EMAIL_BACKEND` | `console` (письма в лог контейнера) или `smtp` (реальная отправка) |
| `REDIS_URL`, `CACHE_TTL_SECONDS` | Подключение к Redis и время жизни кэша списков АЗС |
| `SEED_DEMO_DATA` | `true`/`false` — наполнять ли базу демо-АЗС при первом запуске |

## 6. Роли пользователей

- **Обычный посетитель** — аккаунт не заводит. Оставляет отзывы, комментарии
  и заявки на добавление АЗС анонимно, указывая только отображаемое имя
  (`author_name`). Избранные АЗС и последняя геопозиция хранятся в
  `localStorage` браузера — это личные данные устройства, на сервере их нет.
- **Модератор** (`is_moderator=True`) — единственная роль с аккаунтом.
  Регистрируется на `/register`, но только зная секретный
  `MODERATOR_INVITE_CODE` из `.env`. Может: одобрять/отклонять отзывы с
  нереалистичной ценой и заявки на добавление АЗС; разбирать жалобы;
  **редактировать и удалять любую АЗС** (`PATCH`/`DELETE /api/stations/{id}`,
  панель прямо на странице АЗС); вручную править цену/наличие топлива
  (`PUT /api/stations/{id}/fuels/{fuel_type}`); **удалять любой отзыв и
  комментарий**, включая анонимные (`DELETE /api/reviews/{id}`,
  `DELETE /api/comments/{id}`).
- **Коренной модератор / суперпользователь** (`is_superuser=True`) —
  создаётся автоматически при первом запуске из `.env`. Дополнительно может
  создавать других модераторов напрямую, без кода приглашения
  (`POST /api/moderation/moderators`, вкладка «Модераторы» на `/moderation`).

⚠️ `MODERATOR_INVITE_CODE` в `.env.example` — заглушка. Обязательно задайте
свой длинный случайный код в `.env` перед тем, как выкладывать сайт в
интернет (в т.ч. через localtunnel, см. раздел 9) — иначе кто угодно сможет
зарегистрироваться модератором и получить полный доступ к управлению сайтом.

## 7. Использованные библиотеки

FastAPI, Uvicorn, SQLAlchemy 2.0, asyncpg, aiosqlite, Alembic, Pydantic v2 +
pydantic-settings, python-jose (JWT), passlib + bcrypt, python-multipart,
Jinja2, redis-py (async), httpx, email-validator, gunicorn — см.
`backend/requirements.txt`. Тестовые зависимости — pytest, pytest-asyncio,
pytest-cov, Faker — см. `backend/requirements-dev.txt`.
Фронтенд подключается через CDN: Bootstrap 5.3, Bootstrap Icons, Leaflet
1.9.4, Chart.js 4.4, шрифты Oswald/Inter (Google Fonts).

## 8. Дальнейшие улучшения (сознательно оставлено за рамками MVP)

- Чёрный список отозванных refresh-токенов в Redis (сейчас logout — чисто
  клиентская операция, JWT остаётся валиден до истечения срока).
- Вынос отправки email-уведомлений об изменении цены в очередь задач
  (Celery/RQ) вместо `BackgroundTasks` — важно при большом числе избранных.
- CSRF-защита для форм при переходе на httpOnly-cookie вместо `localStorage`.
- Постраничная предзагрузка (виртуализация) списка АЗС на карте при очень
  большом количестве маркеров (сейчас используется простая пагинация).
- Хранение геометрии в нативном PostGIS-столбце (`Geography(Point)`) вместо
  вычисления `ST_MakePoint` "на лету" — имеет смысл при переходе на
  пространственные индексы (`GIST`) для миллионов записей.

## 9. Как открыть сайт в интернет через localtunnel (для теста, без деплоя)

`localtunnel` — бесплатная утилита с открытым кодом (npm-пакет `localtunnel`),
которая пробрасывает ваш локальный порт наружу и даёт временную публичную
ссылку вида `https://<что-то>.loca.lt`. Никаких платных сервисов и API-ключей
не требуется. Подходит, чтобы показать сайт кому-то удалённо, пока он крутится
на вашем компьютере — это не продакшн-хостинг, а именно "закрытая" демо-ссылка
(её никто не найдёт, если вы её не дадите, но и защиты от перебора URL нет).

### Шаг 1. Поднимите сайт локально

```cmd
cd gde-benz
docker-compose up --build -d
```

Проверьте, что сайт открывается на **http://localhost:8080**.

### Шаг 2. Установите Node.js (если ещё не установлен)

Скачайте LTS-версию с https://nodejs.org и установите (localtunnel — это
npm-пакет, ему нужен Node.js). Проверка после установки:

```cmd
node -v
npm -v
```

### Шаг 3. Запустите localtunnel

Можно без предварительной установки, через `npx` (Node сам скачает пакет при
первом запуске):

```cmd
npx localtunnel --port 8080
```

Или установить один раз глобально и запускать короче:

```cmd
npm install -g localtunnel
lt --port 8080
```

В терминале появится строка вида:

```
your url is: https://shaggy-goats-dance.loca.lt
```

Это и есть ваша публичная ссылка — отправьте её тому, с кем хотите поделиться
сайтом. Пока команда выполняется (терминал открыт) — ссылка работает; закроете
терминал — ссылка перестанет открываться.

### Важные нюансы localtunnel

- **Первое открытие ссылки** может показать промежуточную страницу
  "Click to Continue" с полем — там просят ввести IP-адрес, который
  localtunnel сам показывает на этой же странице (это защита от ботов
  на стороне сервиса localtunnel, не имеет отношения к нашему сайту).
  Нажмите на странице кнопку — и дальше сайт откроется как обычно.
- Ссылка **меняется при каждом перезапуске** `lt`/`npx localtunnel`, если вы
  не указали `--subdomain` (см. ниже).
- Чтобы попытаться зафиксировать поддомен (не гарантируется, если он занят
  другим пользователем):

  ```cmd
  lt --port 8080 --subdomain gde-benz-demo
  ```

  Тогда ссылка будет `https://gde-benz-demo.loca.lt`.
- Так как снаружи сайт открывается по **https**, а Nginx внутри слушает по
  **http**, в `.env` держите `BASE_URL=http://localhost:8080` как есть — на
  ссылки, которые генерирует сам сайт (например, в письмах), это не влияет,
  поскольку регистрация модератора теперь не отправляет писем.
- Если сайт через localtunnel не открывается, а `curl http://localhost:8080`
  локально работает — проверьте, что Docker Desktop не блокирует исходящие
  подключения, и что порт 8080 не занят другим процессом (`netstat -ano | findstr :8080` в cmd).
- **Не забудьте сменить `MODERATOR_INVITE_CODE`, `SECRET_KEY`, `JWT_SECRET_KEY`
  и пароль коренного модератора (`SUPERUSER_PASSWORD`) в `.env` перед тем, как
  давать ссылку кому-либо** — как только сайт доступен по интернету, значения
  по умолчанию из `.env.example` больше не безопасны.

