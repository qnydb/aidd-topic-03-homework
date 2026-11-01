# Анализ исходного кода проекта Investing Demo

**Репозиторий:** `aidd-topic-03-homework`  
**Анализируемая версия:** `362510b` (`Add homework assignment`), рабочая копия без локальных изменений  
**Дата анализа:** 2026-11-01  
**Метод:** статический анализ исходного кода, конфигурации, миграций и шаблонов. Приложение не запускалось, внешние токены и БД не использовались.  
**Связанный документ:** восстановленные требования — `REQUIREMENTS.md`.

---

## 1. Краткое резюме

`Investing Demo` — небольшое учебное веб-приложение на Python/Django для мониторинга акций. Приложение загружает справочник акций из Tinkoff Invest API, хранит его в PostgreSQL, периодически обновляет последнюю цену и показывает список инструментов с цветовой индикацией направления изменения цены. Для каждой акции доступна детальная страница с графиком дневных свечей за последние 180 дней.

Пользовательская авторизация реализована средствами стандартной модели `django.contrib.auth.User`. Авторизованные пользователи могут добавлять, редактировать и удалять акции через веб-интерфейс. Публичная часть (список и детальная страница) доступна без входа.

Дополнительно приложение предоставляет REST API на Django REST Framework: `/api/v1/shares/` с полным CRUD, фильтрацией, поиском, сортировкой и пагинацией.

Ключевые особенности текущей реализации:

- это **монолитное Django-приложение** (MVT) с двумя прикладными приложениями: `dashboard` и `user`;
- справочник акций **общий для всех пользователей**: связи между акцией и пользователем нет;
- обновление котировок выполняется **асинхронно** через Celery Beat/Worker и Redis;
- график строится **синхронно в обработчике запроса** через Tinkoff API и Plotly;
- хранится только **последняя цена**, история котировок в БД не накапливается;
- REST API не требует аутентификации и разрешает анонимные изменения данных;
- проект содержит ряд конфигурационных, эксплуатационных и проектных дефектов, существенных для дальнейшей модернизации.

---

## 2. Назначение приложения

Приложение предназначено для решения следующих задач:

1. **Ведение справочника акций.** Management-команда `add_shares` получает список акций из Tinkoff Invest API и сохраняет их в локальную БД.
2. **Отображение каталога акций.** Главная страница показывает тикер, название, ISIN и текущую цену. Цена подсвечивается зелёным, если последнее изменение было неотрицательным, и розовым — в противном случае.
3. **Просмотр детальной информации.** Для выбранной акции отображаются название, текущая цена, время обновления и интерактивный график дневных цен за 180 дней.
4. **Управление каталогом.** Авторизованный пользователь может создать, изменить или удалить запись об акции через веб-формы.
5. **Регистрация, вход и выход.** Пользователи регистрируются по e-mail и паролю; e-mail используется как логин.
6. **Программный доступ к данным.** REST API предоставляет CRUD-операции над акциями, фильтрацию по `name`/`ticker`, regex-поиск по `name`, сортировку и пагинацию.
7. **Администрирование.** Стандартная Django-админка позволяет управлять акциями и пользователями.
8. **Фоновое обновление цен.** Celery-задача периодически получает последние цены и обновляет поля `last_price` и `is_trend_high`.

Приложение **не является** торговым терминалом или брокерским клиентом: в нём нет портфелей, операций покупки/продажи, списков наблюдения, уведомлений, истории цен, стратегий и интеграции с реальными счетами. Это демонстрационный каталог акций с мониторингом котировок.

---

## 3. Технологический стек

### 3.1. Основные технологии

| Категория | Технология | Версия | Назначение |
|---|---|---|---|
| Язык | Python | не зафиксирована | Django 4.1.7 официально поддерживает Python 3.8–3.11 |
| Web-фреймворк | Django | 4.1.7 | MVT, ORM, маршрутизация, шаблоны, админка, аутентификация |
| REST API | Django REST Framework | 3.14.0 | ViewSet, сериализаторы, пагинация, browsable API |
| Фильтрация API | django-filter | 22.1 | `DjangoFilterBackend`, `FilterSet` |
| Формы | django-crispy-forms | 2.0 | Рендеринг форм |
| Тема форм | crispy-bootstrap3 | 2022.1 | Bootstrap 3-разметка для crispy-форм |
| БД | PostgreSQL | образ `postgres` без фиксации версии | Хранение данных |
| Драйвер БД | psycopg2-binary | 2.9.5 | Подключение Django к PostgreSQL |
| Очередь задач | Celery | 5.2.7 | Фоновые задачи и периодическое обновление котировок |
| Брокер | Redis | клиент 4.5.1, сервер `redis` без фиксации версии | Транспорт сообщений Celery |
| Биржевые данные | tinkoff-investments | 0.2.0b50 (beta) | gRPC-клиент Tinkoff Invest API |
| Визуализация | Plotly | 5.13.1 | Построение графика в HTML |
| Frontend | Bootstrap | 5.1.3 (локальный CSS) | Вёрстка |
| Frontend | Django Templates | — | Серверный рендеринг HTML |
| Контейнеризация | Docker Compose | — | PostgreSQL и Redis для локального запуска |

### 3.2. Вспомогательные и транзитивные зависимости

- `grpcio==1.51.3`, `protobuf==3.20.3`, `pydantic==1.10.6` — транспорт и модели Tinkoff SDK.
- `kombu==5.2.4`, `billiard==3.6.4.0`, `vine==5.0.0`, `amqp==5.1.1` — инфраструктура Celery.
- `asgiref==3.6.0`, `sqlparse==0.4.3` — зависимости Django.
- `cachetools==5.3.0`, `tenacity==8.2.2`, `python-dateutil==2.8.2`, `pytz==2022.7.1` — вспомогательные библиотеки, в том числе для Tinkoff SDK.
- `black==23.1.0`, `isort==5.12.0`, `mypy-extensions`, `pathspec`, `platformdirs`, `tomli` — инструменты разработки, включённые в общий `requirements.txt`.
- `click`, `click-didyoumean`, `click-plugins`, `click-repl`, `prompt-toolkit`, `wcwidth` — зависимости CLI Celery.
- `async-timeout`, `packaging`, `six`, `typing_extensions` — транзитивные зависимости.

### 3.3. Важные особенности стека

- **Версии не зафиксированы для Python, PostgreSQL и Redis**; зафиксированы только Python-пакеты.
- **Django 4.1.7 устарел**: ветка 4.1 не получает обновлений безопасности, официальная поддержка Python ограничена 3.11.
- **`tinkoff-investments` — beta-версия** с устаревшим `protobuf==3.20.3` и `grpcio==1.51.3`; возможны проблемы сборки на Python 3.12+.
- В `requirements.txt` одновременно присутствуют `tinkoff==0.1.1` и `tinkoff-investments==0.2.0b50`. Пакет `tinkoff` в коде не используется и потенциально дублирует/конфликтует с namespace `tinkoff`.
- `eventlet`, упомянутый в README для Windows, в `requirements.txt` отсутствует.
- CSS подключён как Bootstrap 5.1.3, а crispy-формы настроены на пакет `bootstrap3`; это рассогласование разметки и стилей.

---

## 4. Структура проекта

```text
aidd-topic-03-homework/
├── manage.py                     # Точка входа Django CLI
├── requirements.txt              # Python-зависимости
├── docker-compose.yml            # PostgreSQL + Redis
├── README.md                     # Команды запуска
├── HOMEWORK.md                   # Описание учебного задания
├── config/                       # Django-проект (настройки и корневая маршрутизация)
│   ├── settings.py               # Настройки Django, БД, DRF, Celery, логирования
│   ├── urls.py                   # Корневые URL: admin, dashboard, user
│   ├── celery.py                 # Celery-приложение и beat-расписание
│   ├── asgi.py                   # ASGI-точка входа
│   └── wsgi.py                   # WSGI-точка входа
├── dashboard/                    # Приложение каталога акций
│   ├── models.py                 # Модель Share
│   ├── forms.py                  # ModelForm для акции
│   ├── views.py                  # HTML-представления (ListView, Detail, CRUD)
│   ├── urls.py                   # Маршруты dashboard и подключение API
│   ├── admin.py                  # Регистрация Share в админке
│   ├── tasks.py                  # Celery-задача обновления цен
│   ├── tests.py                  # API-тесты
│   ├── api/
│   │   ├── urls.py               # /api/ -> /api/v1/
│   │   └── v1/
│   │       ├── router.py         # DefaultRouter, регистрация ViewSet
│   │       ├── serializer.py     # ShareSerializer
│   │       ├── views.py          # ShareViewSet и ShareFilter
│   │       └── urls.py           # Подключение router
│   ├── services/
│   │   ├── market_data.py        # Получение последней цены из Tinkoff API
│   │   └── graph.py              # Построение графика свечей через Plotly
│   ├── management/commands/
│   │   └── add_shares.py         # Импорт справочника акций из Tinkoff API
│   ├── migrations/               # 7 миграций модели Share
│   ├── static/dashboard/css/     # Bootstrap 5.1.3 и стили footer
│   └── templates/dashboard/      # base, list, detail, create, delete
└── user/                         # Приложение аутентификации
    ├── models.py                 # Реэкспорт стандартной User-модели
    ├── forms.py                  # RegistrationForm, AuthForm
    ├── views.py                  # AuthView, RegistrationView, LogoutView
    ├── urls.py                   # login/registration/logout
    ├── static/user/css/          # Bootstrap 5.1.3 и стили формы входа
    └── templates/user/           # base, auth, registration
```

---

## 5. Архитектура приложения

### 5.1. Общий характер архитектуры

Приложение построено по классической схеме **Django MVT (Model–Template–View)** и представляет собой **модульный монолит**:

- **Model** — ORM-модель `Share` и стандартные модели Django (`User`, сессии, права);
- **Template** — серверные HTML-шаблоны Django;
- **View** — class-based views для HTML и DRF ViewSet для API.

Дополнительно выделены:

- **сервисный слой** (`dashboard/services/`) для работы с Tinkoff API и Plotly;
- **асинхронный слой** (`dashboard/tasks.py`, `config/celery.py`) для фонового обновления цен;
- **management-команда** для первоначального импорта справочника;
- **REST-слой** (`dashboard/api/v1/`) для программного доступа.

При этом сервисный слой выделен не полностью: часть бизнес-логики (сравнение цен, выбор `is_trend_high`) находится непосредственно в Celery-задаче, а CRUD-логика — в представлениях и DRF ViewSet. Репозиторного слоя нет: представления работают с ORM напрямую.

### 5.2. Схема компонентов и потоков данных

```text
                        ┌─────────────────────────────────────────────┐
                        │                Браузер / клиент              │
                        │        HTML-страницы / REST-запросы          │
                        └───────────────┬─────────────────────────────┘
                                        │ HTTP
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Django-проект config                                 │
│                                                                              │
│  config/urls.py ──┬── /admin/ ──────────────► django.contrib.admin           │
│                   ├── / ────────────────────► dashboard (HTML)               │
│                   ├── /api/v1/ ─────────────► dashboard.api.v1 (DRF)         │
│                   └── /user/... ────────────► user (аутентификация)          │
│                                                                              │
│  dashboard/views.py ──► dashboard/forms.py ──► Share (ORM)                   │
│  dashboard/views.py ──► dashboard/services/graph.py ──► Tinkoff Invest API   │
│  dashboard/api/v1/views.py ──► ShareSerializer ──► Share (ORM)               │
│  user/views.py ──► user/forms.py ──► django.contrib.auth.User (ORM)          │
└───────────────┬──────────────────────────────────────────────┬───────────────┘
                │ ORM / psycopg2                               │ gRPC (синхронно)
                ▼                                              ▼
        ┌────────────────┐                          ┌──────────────────────────┐
        │   PostgreSQL   │                          │   Tinkoff Invest API     │
        │  dashboard_share│                          │  котировки, свечи,       │
        │  auth_* и др.   │                          │  справочник инструментов │
        └────────────────┘                          └──────────────────────────┘
                ▲
                │ ORM / psycopg2
┌───────────────┴──────────────────────────────────────────────────────────────┐
│ Celery Worker (dashboard.tasks.update_last_price)                            │
│   получает последнюю цену через dashboard/services/market_data.py            │
│   и обновляет last_price / is_trend_high в PostgreSQL                        │
└───────────────▲──────────────────────────────────────────────────────────────┘
                │ задачи
        ┌───────┴────────┐
        │      Redis     │
        │  broker Celery │
        └───────▲────────┘
                │ периодические задачи (каждые 10 секунд)
        ┌───────┴────────┐
        │  Celery Beat   │
        └────────────────┘
```

### 5.3. Слои и их ответственность

| Слой | Компоненты | Ответственность |
|---|---|---|
| Представление (HTML) | `dashboard/views.py`, `dashboard/templates/` | Маршрутизация запросов, выборка данных, рендеринг HTML, формы CRUD |
| Представление (API) | `dashboard/api/v1/` | Сериализация, CRUD, фильтрация, поиск, сортировка, пагинация |
| Аутентификация | `user/views.py`, `user/forms.py`, `user/templates/` | Регистрация, вход, выход, проверка e-mail и паролей |
| Прикладная логика | `dashboard/services/`, `dashboard/tasks.py`, management-команда | Работа с Tinkoff API, расчёт тренда, импорт справочника |
| Модель данных | `dashboard/models.py`, миграции | Хранение акций и их последних рыночных показателей |
| Инфраструктура | `config/settings.py`, `config/celery.py`, `docker-compose.yml` | Настройки БД, очереди, логирования, контейнеров |
| Администрирование | `dashboard/admin.py` | Управление акциями через Django Admin |

### 5.4. Взаимодействие компонентов

1. **Веб-запрос списка.** `ShareListView` через ORM получает страницу акций (100 записей), сортирует по `name`, передаёт `share_list` и `page_obj` в `dashboard/list.html`.
2. **Веб-запрос детали.** `ShareDetailView` получает `Share` по `id`, вызывает `get_graph(share)`, который синхронно обращается к Tinkoff API за свечами, строит Plotly-график и возвращает HTML-фрагмент.
3. **Создание/редактирование.** `ShareCreateView`/`ShareUpdateView` используют `ShareModelForm` (все редактируемые поля модели), после успешного сохранения перенаправляют на список.
4. **Удаление.** `ShareDeleteView` показывает подтверждение и удаляет запись после POST.
5. **Аутентификация.** `RegistrationView` создаёт пользователя через `User.objects.create_user`, `AuthView` проверяет учётные данные через `authenticate`, `LogoutView` завершает сессию.
6. **API.** `DefaultRouter` направляет запросы в `ShareViewSet`; `ShareSerializer` преобразует модель в JSON/HTML browsable API; `ShareFilter`, `SearchFilter`, `OrderingFilter` и `LimitOffsetPagination` обрабатывают параметры запроса.
7. **Фоновое обновление.** Celery Beat раз в 10 секунд ставит задачу `dashboard.tasks.update_last_price`; Worker перебирает все `Share`, получает последнюю цену через Tinkoff API и обновляет запись.
8. **Импорт справочника.** `python manage.py add_shares` открывает gRPC-клиент, получает список акций и сохраняет их через `get_or_create`.

---

## 6. Компоненты и их назначение

### 6.1. `config` — конфигурация проекта

| Файл | Назначение |
|---|---|
| `settings.py` | `INSTALLED_APPS`, middleware, шаблоны, подключение PostgreSQL, настройки DRF, crispy-forms, логирование, Celery, `TINKOFF_TOKEN` |
| `urls.py` | Корневые маршруты: `/admin/`, `dashboard` на `/`, `user` на `/user` |
| `celery.py` | Создание Celery-приложения `config`, автопоиск задач, beat-расписание |
| `wsgi.py` | WSGI-приложение для синхронного развёртывания |
| `asgi.py` | ASGI-приложение (в проекте не используется для запуска) |

### 6.2. `dashboard` — каталог акций

| Компонент | Назначение |
|---|---|
| `models.py` | Модель `Share` — единственная прикладная сущность |
| `forms.py` | `ShareModelForm` на основе `Share`, все редактируемые поля |
| `views.py` | `ShareListView`, `ShareDetailView`, `ShareCreateView`, `ShareUpdateView`, `ShareDeleteView`, `IsUserAuthenticatedMixin` |
| `urls.py` | Маршруты HTML-страниц и подключение API |
| `admin.py` | `ShareAdmin`: список, поиск и поля формы в админке |
| `tasks.py` | Периодическая задача `update_last_price` |
| `services/market_data.py` | `get_last_data(share)` — последняя цена через Tinkoff API |
| `services/graph.py` | `get_graph(share)` — свечной график за 180 дней через Plotly |
| `management/commands/add_shares.py` | Импорт всех акций из Tinkoff API |
| `api/v1/views.py` | `ShareViewSet`, `ShareFilter` |
| `api/v1/serializer.py` | `ShareSerializer` |
| `api/v1/router.py` | Регистрация `shares` в `DefaultRouter` |
| `templates/dashboard/` | `base.html`, `list.html`, `detail.html`, `create.html`, `delete.html` |
| `tests.py` | 10 тестов API и главной страницы |
| `migrations/` | История схемы `Share` (7 миграций) |

### 6.3. `user` — аутентификация

| Компонент | Назначение |
|---|---|
| `models.py` | Только `User = get_user_model()`; собственная модель пользователя отсутствует |
| `forms.py` | `RegistrationForm` (e-mail, пароль, повтор пароля), `AuthForm` (e-mail, пароль) |
| `views.py` | `AuthView`, `RegistrationView`, `LogoutView` |
| `urls.py` | `/user/login/`, `/user/registration/`, `/user/logout/` |
| `templates/user/` | Отдельный `base.html`, формы входа и регистрации |

### 6.4. Статические ресурсы и шаблоны

- `dashboard/static/dashboard/css/bootstrap.min.css` — Bootstrap 5.1.3.
- `dashboard/static/dashboard/css/sticky-footer-navbar.css` — стили прижатого footer.
- `user/static/user/css/bootstrap.min.css` — вторая копия Bootstrap 5.1.3.
- `user/static/user/css/signin.css` — стили центрированной формы входа.
- В `dashboard/base.html` подключены Plotly CDN (`plotly-latest.min.js`), jQuery slim, Popper, Font Awesome, Moment.js и Tempus Dominus. Большая часть этих библиотек в коде шаблонов не используется.
- `user/base.html` — отдельный минимальный layout без общей навигации.

---

## 7. Маршруты и точки входа

### 7.1. HTML-маршруты

| Метод | URL | Обработчик | Доступ | Результат |
|---|---|---|---|---|
| GET | `/` | `ShareListView` | Публичный | Список акций с пагинацией по 100 |
| GET | `/<int:pk>` | `ShareDetailView` | Публичный | Детальная страница и график |
| GET | `/create` | `ShareCreateView` | Только авторизованные | Форма создания |
| POST | `/create` | `ShareCreateView` | Только авторизованные | Создание и редирект на `/` |
| GET | `/update/<int:pk>` | `ShareUpdateView` | Только авторизованные | Форма редактирования |
| POST | `/update/<int:pk>` | `ShareUpdateView` | Только авторизованные | Сохранение и редирект на `/` |
| GET | `/delete/<int:pk>` | `ShareDeleteView` | Только авторизованные | Подтверждение удаления |
| POST | `/delete/<int:pk>` | `ShareDeleteView` | Только авторизованные | Удаление и редирект на `/` |
| GET/POST | `/user/login/` | `AuthView` | Публичный | Вход |
| GET/POST | `/user/registration/` | `RegistrationView` | Публичный | Регистрация |
| GET | `/user/logout/` | `LogoutView` | Публичный | Выход и редирект на `/` |
| GET/POST | `/admin/` | Django Admin | Персонал | Управление моделями |

Маршруты `dashboard` не имеют завершающего слэша (кроме корневого), маршруты API — имеют.

### 7.2. REST API

| Метод | URL | Назначение |
|---|---|---|
| GET | `/api/v1/` | Корень DefaultRouter, список доступных ресурсов |
| GET | `/api/v1/shares/` | Список акций (пагинация `limit`/`offset`) |
| POST | `/api/v1/shares/` | Создание акции |
| GET | `/api/v1/shares/{id}/` | Получение акции |
| PUT | `/api/v1/shares/{id}/` | Полное обновление |
| PATCH | `/api/v1/shares/{id}/` | Частичное обновление |
| DELETE | `/api/v1/shares/{id}/` | Удаление |

Параметры списка:

- `?name=...`, `?ticker=...` — точная фильтрация через `django-filter`;
- `?search=...` — regex-поиск по `name` (в DRF префикс `$` означает regex search);
- `?ordering=name`, `?ordering=-ticker` — сортировка по `name` или `ticker`;
- `?limit=...&offset=...` — limit-offset пагинация, размер страницы по умолчанию 100.

Аутентификация и разрешения для API не настроены: действуют значения DRF по умолчанию (`AllowAny`), поэтому анонимный клиент может выполнять любые CRUD-операции.

---

## 8. Модель данных

### 8.1. ER-представление

```text
┌──────────────────────────────┐          ┌──────────────────────────────┐
│ django.contrib.auth.User     │          │ dashboard.Share              │
│ (стандартная модель)         │          │ (единственная прикладная     │
│                              │          │  модель)                     │
│ id            BigAutoField   │          │ id            BigAutoField   │
│ username      CharField      │          │ figi          CharField(255) │
│ email         EmailField     │          │ isin          CharField(255) │
│ password      CharField      │          │ name          CharField(255) │
│ is_active     BooleanField   │          │ ticker        CharField(10)  │
│ is_staff      BooleanField   │          │ currency      CharField(20)  │
│ is_superuser  BooleanField   │          │ uid           CharField(255) │
│ date_joined   DateTimeField  │          │ last_price    FloatField     │
│ ...                          │          │ is_trend_high BooleanField   │
└──────────────────────────────┘          │ updated_at    DateTimeField  │
                                          └──────────────────────────────┘
             связей между User и Share нет: каталог акций общий
```

### 8.2. Сущность `Share` (таблица `dashboard_share`)

| Поле | Тип | Ограничения | Назначение |
|---|---|---|---|
| `id` | `BigAutoField` | PK, auto | Идентификатор |
| `figi` | `CharField(max_length=255)` | `unique=True`, обязательное | FIGI — идентификатор инструмента в Tinkoff API |
| `isin` | `CharField(max_length=255)` | обязательное, **не уникальное** | ISIN — международный идентификатор ценной бумаги |
| `name` | `CharField(max_length=255)` | обязательное | Наименование компании |
| `ticker` | `CharField(max_length=10)` | `unique=True`, обязательное | Биржевой тикер |
| `currency` | `CharField(max_length=20)` | обязательное | Валюта цены |
| `uid` | `CharField(max_length=255)` | `unique=True`, обязательное | UID инструмента в Tinkoff API |
| `last_price` | `FloatField` | `null=True`, `blank=True` | Последняя полученная цена |
| `is_trend_high` | `BooleanField` | `null=True`, `blank=True` | Признак «цена не снизилась»; допускает три состояния |
| `updated_at` | `DateTimeField` | `auto_now=True`, `null=True`, `blank=True` | Время последнего сохранения через `save()` |

`Meta`:

- `verbose_name = "Акция"`;
- `verbose_name_plural = "Акции"`;
- сортировка по умолчанию не задана (список сортируется во view);
- индексы явно не заданы; уникальные ограничения создают индексы для `figi`, `ticker`, `uid`.

### 8.3. Особенности модели

1. **Нет связи с пользователем.** Любой авторизованный пользователь может редактировать и удалять любую акцию. Персональных списков, портфелей и владения записями нет.
2. **`isin` не уникален**, хотя логически ISIN должен быть уникальным идентификатором бумаги.
3. **`ticker` ограничен 10 символами**; для части инструментов этого может быть недостаточно.
4. **Цена хранится во `FloatField`**, что не подходит для денежных значений; в Tinkoff API цена — это `Quotation` с полями `units` и `nano`.
5. **`is_trend_high` — nullable Boolean**: `True` — цена выросла или не изменилась, `False` — снизилась, `NULL` — значение ещё не рассчитано. В шаблоне `NULL` отображается как «розовый» (falsy).
6. **`updated_at` обновляется только при вызове `Model.save()`**. Celery-задача использует `QuerySet.update()`, поэтому время фактического обновления цены в этом поле не меняется.
7. **`uid` обязателен, но не включён в API-сериализатор.** При создании через API Django подставляет пустую строку; из-за `unique=True` вторая такая запись приведёт к ошибке целостности.
8. **История цен не хранится.** Есть только одно поле `last_price`; для истории потребуется отдельная сущность.

### 8.4. Стандартные таблицы Django

Помимо `dashboard_share`, в БД создаются стандартные таблицы:

- `auth_user` — пользователи;
- `auth_group`, `auth_permission`, `auth_user_groups`, `auth_user_user_permissions` — права и группы;
- `django_admin_log` — журнал админки;
- `django_content_type` — типы контента;
- `django_migrations` — применённые миграции;
- `django_session` — сессии.

Собственная модель пользователя не создаётся: `user/models.py` только получает стандартную модель через `get_user_model()`. Поэтому e-mail в `auth_user` не имеет уникального ограничения на уровне БД.

### 8.5. История миграций

| Миграция | Изменения |
|---|---|
| `0001_add_share` | Создание `Share`: `figi`, `isin`, `name`, `ticker`, `currency` |
| `0002_update_share` | `figi` и `ticker` становятся уникальными |
| `0003_add_uuid` | Добавление `uid` с `unique=True` |
| `0004_add_last_price` | Добавление `last_price`, `is_trend_high`, уточнение `verbose_name` |
| `0005_add_verbose_name` | Исправление `verbose_name` для `name` |
| `0006_add_verbose_name` | Исправление `verbose_name` для `ticker` |
| `0007_add_updated_at` | Добавление `updated_at` с `auto_now=True` |

Миграции отражают постепенное развитие модели; в `0004` заметны следы копирования (`verbose_name="Валюта"` у `name`, `"Ти"` у `ticker`), исправленные следующими миграциями.

### 8.6. Жизненный цикл данных

1. **Первичное наполнение.** `add_shares` получает справочник из Tinkoff API и создаёт записи `Share`.
2. **Обновление котировок.** Celery-задача получает последнюю цену и обновляет `last_price` и `is_trend_high`.
3. **Ручное управление.** Пользователь или API создаёт/изменяет/удаляет записи; форма позволяет вручную менять в том числе `last_price`, `is_trend_high` и `uid`.
4. **Просмотр.** Список читает сохранённые значения; детальная страница дополнительно запрашивает исторические свечи напрямую из Tinkoff API, не сохраняя их.
5. **Пользователи.** Хранятся в `auth_user`; с акциями не связаны.

---

## 9. Механизмы обработки пользовательских запросов

### 9.1. Обработка HTML-запроса

1. Запрос приходит на WSGI/ASGI-приложение `config`.
2. `config/urls.py` выбирает пространство имён: `admin`, `dashboard` или `user`.
3. Проходят middleware: security, session, common, CSRF, authentication, messages, clickjacking.
4. Вызывается class-based view:
   - `ListView` формирует queryset `Share.objects.all()`, применяет `ordering=["name"]` и пагинацию `paginate_by=100`;
   - `DetailView` (фактически обычный `View`) делает `Share.objects.get(id=pk)` и вызывает сервис построения графика;
   - `CreateView`/`UpdateView` создают `ShareModelForm`, валидируют данные и сохраняют модель;
   - `DeleteView` показывает объект и удаляет его после POST.
5. Контекст дополняется context processors (`request`, `auth`, `messages`, `debug`).
6. Django рендерит шаблон и возвращает HTML.
7. При успешном создании/редактировании/удалении выполняется редирект на `dashboard:index`.

### 9.2. Аутентификация и авторизация

**Регистрация:**

- `RegistrationForm` принимает `email`, `password1`, `password2`;
- проверяет совпадение паролей;
- проверяет отсутствие пользователя с таким e-mail через `User.objects.filter(email=email)`;
- создаёт пользователя `User.objects.create_user(email, email, password)`, то есть e-mail используется и как `username`, и как `email`;
- выполняет `authenticate` и `login`, затем редирект на главную.

**Вход:**

- `AuthView.post` создаёт `AuthForm`, но **не вызывает `form.is_valid()`**;
- берёт `form.data["email"]` и `form.data["password"]` напрямую;
- выполняет `authenticate(username=email.lower(), password=...)`;
- при успехе — `login` и редирект, при ошибке — повторный рендер с сообщением.

**Выход:**

- `LogoutView.get` вызывает `logout(request)` и перенаправляет на главную.

**Ограничение доступа к CRUD:**

- `IsUserAuthenticatedMixin` наследует `UserPassesTestMixin`;
- `test_func` проверяет `request.user.is_authenticated`;
- при неуспехе `handle_no_permission` перенаправляет на главную, а не на страницу входа;
- проверка действует только на create/update/delete; список, детальная страница и API публичны;
- проверка не привязана к владельцу объекта, поскольку владельца у `Share` нет.

### 9.3. Обработка REST-запроса

1. `config/urls.py` → `dashboard/urls.py` → `dashboard/api/urls.py` → `dashboard/api/v1/urls.py`.
2. `DefaultRouter` сопоставляет `shares` с `ShareViewSet`.
3. `ModelViewSet` автоматически предоставляет `list`, `create`, `retrieve`, `update`, `partial_update`, `destroy`.
4. `ShareSerializer` преобразует поля модели; `updated_at` доступен только для чтения, `uid` отсутствует.
5. `ShareFilter` фильтрует по `name` и `ticker`; `SearchFilter` выполняет regex-поиск по `name`; `OrderingFilter` сортирует по `name`/`ticker`.
6. `LimitOffsetPagination` ограничивает выдачу 100 записями на страницу.
7. Ответ формируется в JSON или в HTML browsable API в зависимости от `Accept`.

### 9.4. Фоновое обновление котировок

1. `config/celery.py` регистрирует beat-расписание `add-every-5-min` с интервалом **10 секунд** (название не соответствует интервалу).
2. Beat отправляет задачу `dashboard.tasks.update_last_price` в Redis.
3. Worker выбирает задачу и получает `Share.objects.all()`.
4. Для каждой акции вызывается `get_last_data(share)`:
   - открывается gRPC-клиент `Client(TINKOFF_TOKEN)`;
   - вызывается `client.market_data.get_last_prices(figi=[share.figi])`;
   - возвращается `[price.units, price.time]`.
5. Задача сравнивает новую цену с сохранённой:
   - если старая цена больше новой — `is_trend_high = False`;
   - иначе — `is_trend_high = True`;
   - если старая цена `NULL` и сравнение невозможно — исключение логируется, `is_trend_high = True`.
6. Обновление выполняется через `Share.objects.filter(id=share.id).update(...)`.
7. Ошибки по каждой акции логируются, цикл продолжается; задача не прерывается целиком.

### 9.5. Построение графика

1. `ShareDetailView.get` получает акцию по `id`.
2. `get_graph(share)` открывает Tinkoff-клиент и запрашивает дневные свечи за 180 дней:
   - `client.get_all_candles(figi=share.figi, from_=now() - timedelta(days=180), interval=CANDLE_INTERVAL_DAY)`.
3. Из каждой свечи берётся `close.units` (без `nano`) и дата.
4. Plotly формирует `Scatter`-график и возвращает HTML-фрагмент `div`.
5. Шаблон вставляет фрагмент с `{% autoescape off %}`.
6. График не кэшируется и не сохраняется в БД; при каждом открытии страницы выполняется новый запрос к Tinkoff API.

### 9.6. Обработка ошибок

| Ситуация | Фактическое поведение |
|---|---|
| Неверный логин/пароль | Рендер формы с сообщением об ошибке, HTTP 200 |
| Ошибка валидации формы регистрации | Рендер формы; извлечение текста ошибки из `form.errors["__all__"]` |
| Ошибка валидации формы акции | Стандартный рендер формы с ошибками полей |
| Несуществующая акция на детальной странице | `Share.DoesNotExist` не обрабатывается → HTTP 500 |
| Несуществующая акция в API | DRF возвращает HTTP 404 |
| Ошибка Tinkoff API в задаче | Логируется, обработка следующей акции продолжается |
| Ошибка Tinkoff API при построении графика | Не обрабатывается → HTTP 500 |
| Ошибка Tinkoff API при импорте | Не обрабатывается → команда завершается исключением |
| Дублирование `figi`/`ticker`/`uid` | Ошибка целостности БД или ошибка формы |

---

## 10. Взаимодействие с базой данных

### 10.1. Подключение

В `config/settings.py` настроено подключение к PostgreSQL:

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "investing",
        "USER": "postgres",
        "PASSWORD": "postgres",
        "HOST": "localhost",
        "PORT": "5432",
    }
}
```

Параметры захардкожены; переменные окружения не используются. Закомментированный вариант SQLite присутствует, но не активен.

### 10.2. Используемые механизмы ORM

| Операция | Где используется | Особенности |
|---|---|---|
| `Share.objects.all()` | `ShareListView`, Celery-задача | Полная выборка; во view ограничивается пагинацией, в задаче — нет |
| `Share.objects.get(id=pk)` | `ShareDetailView` | Без `get_object_or_404`, без обработки `DoesNotExist` |
| `Share.objects.filter(id=...).update(...)` | `update_last_price` | Не вызывает `save()`, не обновляет `auto_now` |
| `Share.objects.get_or_create(...)` | `add_shares` | Все переданные поля входят в lookup |
| `ModelForm.save()` | create/update views | Стандартная валидация и сохранение |
| `ModelSerializer.save()` | DRF ViewSet | Стандартное создание/обновление |
| `QuerySet` filtering/ordering/pagination | DRF | Выполняется на стороне БД (LIMIT/OFFSET, WHERE, ORDER BY) |
| `QuerySet.delete()` | `DeleteView`, DRF destroy | Каскадных связей у `Share` нет |

### 10.3. Транзакции и целостность

- Явные транзакции (`transaction.atomic`) в прикладном коде не используются.
- Django работает в режиме autocommit: каждое изменение фиксируется отдельно.
- `get_or_create` внутри использует транзакцию, но логика импорта в целом неатомарна.
- Celery-задача обновляет акции по одной; при сбое посередине часть записей уже обновлена, часть — нет.
- Конкурентные изменения из веб-формы, API, админки и Celery-задачи не синхронизируются; действует принцип «последняя запись побеждает».
- Уникальные ограничения на `figi`, `ticker`, `uid` обеспечивают часть целостности на уровне БД, но `isin` и `name` не защищены от дублей.
- Внешние ключи и каскадные удаления отсутствуют.

### 10.4. Производительность запросов

- Список акций пагинируется на уровне БД (100 записей), что ограничивает объём передачи.
- API также использует `LimitOffsetPagination` с размером страницы 100.
- Celery-задача делает **отдельный сетевой вызов Tinkoff API на каждую акцию** и отдельный `UPDATE` на каждую акцию; пакетных операций нет.
- Детальная страница делает синхронный сетевой вызов Tinkoff API и строит график в процессе обработки HTTP-запроса.
- `select_related`/`prefetch_related` не требуются из-за отсутствия связей, но и кэширования данных нет.
- Денормализованное поле `last_price` избавляет список от обращения к Tinkoff API, но детальная страница всё равно обращается к внешнему сервису.

### 10.5. Миграции и схема

Схема управляется стандартными миграциями Django. Применение: `python manage.py migrate`. Отдельных миграций для пользователей нет, так как используется встроенная модель `auth.User`. Тестовая БД создаётся Django на основе тех же миграций и требует доступного PostgreSQL.

---

## 11. Внешние интеграции

### 11.1. Tinkoff Invest API

- Подключение через `tinkoff-investments` (gRPC).
- Токен задаётся в `config/settings.py` (`TINKOFF_TOKEN`), сейчас пустой.
- Используемые методы:
  - `client.instruments.shares()` — справочник акций;
  - `client.market_data.get_last_prices(figi=[...])` — последняя цена;
  - `client.get_all_candles(figi=..., from_=..., interval=CANDLE_INTERVAL_DAY)` — дневные свечи.
- Идентификация инструмента преимущественно по `figi`; `uid` хранится, но в запросах не используется. FIGI считается устаревающим идентификатором, Tinkoff рекомендует `instrument_id`/`uid`.
- Клиент создаётся заново на каждый вызов (на каждую акцию в задаче и на каждый просмотр детальной страницы).
- Обработка ошибок/повторов на уровне SDK не настроена; `tenacity` присутствует как транзитивная зависимость, но в коде не используется.

### 11.2. Redis и Celery

- Redis используется как broker: `CELERY_BROKER_URL = "redis://127.0.0.1:6379/0"`.
- Result backend не настроен; результаты задач не сохраняются.
- Beat-расписание задано в коде, а не в БД; `django-celery-beat` не используется.
- `CELERY_TASK_TIME_LIMIT = 30 * 60` (30 минут), `CELERY_TASK_TRACK_STARTED = True`, `CELERY_TASK_ALWAYS_EAGER = False`.
- Часовой пояс Celery — `Europe/Moscow`, Django — `UTC`.

### 11.3. Frontend-ресурсы

- Plotly загружается с CDN `plotly-latest.min.js` (без фиксации версии).
- Дополнительно Plotly-график, вероятно, встраивает собственную копию JS-библиотеки в HTML (поведение `plotly.offline.plot` по умолчанию), что увеличивает размер страницы.
- jQuery slim, Popper, Font Awesome, Moment.js, Tempus Dominus подключены в `dashboard/base.html`, но в шаблонах не используются.
- Bootstrap 5.1.3 хранится локально в двух копиях — для `dashboard` и `user`.

### 11.4. Docker Compose

`docker-compose.yml` поднимает только два сервиса:

- `db` — PostgreSQL с БД `investing`, пользователем/паролем `postgres`, портом `5432`;
- `redis` — Redis с портом `6379`.

Django-приложение, Celery Worker и Celery Beat в compose не описаны; они запускаются на хосте. Каталог проекта монтируется в контейнер БД как `./:/app`, что не относится к хранению данных PostgreSQL. Именованный volume для `/var/lib/postgresql/data` не задан, поэтому данные БД не гарантированно сохраняются при пересоздании контейнера.

---

## 12. Зависимости: назначение и риски

| Зависимость | Версия | Назначение | Риск/замечание |
|---|---|---|---|
| Django | 4.1.7 | Web-фреймворк | Устаревшая ветка, нет обновлений безопасности |
| djangorestframework | 3.14.0 | REST API | Совместим с Django 4.1, но версия старая |
| django-filter | 22.1 | Фильтрация API | Старая версия |
| django-crispy-forms | 2.0 | Формы | Совместим с Django 4.1 |
| crispy-bootstrap3 | 2022.1 | Bootstrap 3-разметка | Не соответствует Bootstrap 5 в статике |
| psycopg2-binary | 2.9.5 | Драйвер PostgreSQL | Старый бинарный драйвер |
| celery | 5.2.7 | Очередь задач | Старая версия |
| redis | 4.5.1 | Клиент Redis | Сервер не зафиксирован |
| tinkoff-investments | 0.2.0b50 | Tinkoff API | Beta, старые protobuf/grpcio |
| tinkoff | 0.1.1 | Не используется | Потенциальный конфликт/лишняя зависимость |
| grpcio | 1.51.3 | gRPC | Может не собираться на Python 3.12+ |
| protobuf | 3.20.3 | Protobuf | Сильно устарел |
| pydantic | 1.10.6 | Модели Tinkoff SDK | Ветка 1.x |
| plotly | 5.13.1 | Графики | `plotly.offline` — устаревающий API |
| black | 23.1.0 | Форматирование | Инструмент разработки в runtime-зависимостях |
| isort | 5.12.0 | Сортировка импортов | Инструмент разработки в runtime-зависимостях |
| eventlet | не указан | Worker для Windows | Упомянут в README, но отсутствует в requirements |

---

## 13. Конфигурация и запуск

### 13.1. Предполагаемый порядок запуска

```bash
# 1. Поднять PostgreSQL и Redis
docker-compose -f docker-compose.yml up

# 2. Применить миграции
python manage.py migrate

# 3. Загрузить справочник акций
python manage.py add_shares

# 4. Запустить Celery Beat
celery -A config beat

# 5. Запустить Celery Worker
celery -A config worker -l info

# 6. Запустить Django
python manage.py runserver
```

Для Windows README дополнительно предлагает `celery -A config worker -l info -P eventlet` и установку `eventlet`/`cryptography` вручную.

### 13.2. Обязательные настройки

Перед реальным запуском необходимо заполнить:

- `SECRET_KEY` — сейчас пустая строка;
- `TINKOFF_TOKEN` — сейчас пустая строка;
- параметры PostgreSQL (сейчас `postgres/postgres`, `localhost:5432`);
- при необходимости — `ALLOWED_HOSTS` (сейчас пустой список);
- при необходимости — адрес Redis (сейчас `127.0.0.1:6379`).

`DEBUG = True` включён постоянно; конфигурации для production (разделение настроек, `.env`, `STATIC_ROOT`, `CSRF_TRUSTED_ORIGINS`, запуск через WSGI/ASGI-сервер) отсутствуют.

### 13.3. Ограничения текущей конфигурации

- Нет `.env`/`.env.example` и чтения переменных окружения.
- Нет Dockerfile и сервисов приложения/Celery в compose.
- Нет фиксации версий PostgreSQL, Redis и Python.
- Нет инструкции по созданию суперпользователя и запуску тестов.
- `app.logs` пишется в текущую рабочую директорию; ротация логов не настроена.
- Язык интерфейса — русский, но `LANGUAGE_CODE = "en-us"`; Django использует UTC, Celery — Europe/Moscow.

---

## 14. Тесты

В `dashboard/tests.py` находятся 10 тестов на базе `rest_framework.test.APITestCase`:

| Тест | Проверяет |
|---|---|
| `test_share_list_html` | HTTP 200 для главной HTML-страницы |
| `test_share_list` | Список API, пагинацию и поля первой акции |
| `test_share_detail` | Получение акции по ID через API |
| `test_share_detail_invalid` | HTTP 404 для несуществующего ID |
| `test_share_update` | PUT-обновление названия |
| `test_share_update_invalid` | HTTP 400 при неполных данных |
| `test_share_create` | POST-создание акции |
| `test_share_create_invalid` | HTTP 400 при отсутствии обязательных полей |
| `test_share_delete` | DELETE и проверка пустого списка |
| `test_share_delete_invalid` | HTTP 404 при удалении несуществующей акции |

Что **не покрыто** тестами:

- регистрация, вход и выход;
- ограничение доступа к create/update/delete;
- фильтрация, поиск и сортировка API;
- пагинация с `limit`/`offset`;
- Celery-задача и сервисы Tinkoff API;
- management-команда `add_shares`;
- обработка ошибок Tinkoff API;
- модели и миграции отдельно от API-сценариев.

Тесты используют реальную PostgreSQL-тестовую БД, поэтому для их запуска требуется доступный сервер БД. Тесты не проверяют аутентификацию и потому не обнаруживают, что API полностью открыт.

---

## 15. Выявленные дефекты, ограничения и риски

Ниже перечислены особенности, важные для понимания текущей архитектуры и планирования модернизации. Это **не функциональные требования**, а описание фактического состояния кода.

### 15.1. Безопасность

1. **REST API полностью открыт.** В `REST_FRAMEWORK` не заданы `DEFAULT_PERMISSION_CLASSES`, а в `ShareViewSet` — `permission_classes`; действует `AllowAny`. Анонимный клиент может создавать, изменять и удалять акции.
2. **Секреты и режим разработки захардкожены.** `SECRET_KEY = ""`, `DEBUG = True`, `ALLOWED_HOSTS = []`, пароль БД `postgres`, токен Tinkoff в `settings.py`. Это небезопасно и неудобно для развёртывания.
3. **Нет разграничения прав на уровне объектов.** Любой авторизованный пользователь может редактировать/удалять любую акцию; владелец у `Share` отсутствует.
4. **Выход через GET.** `LogoutView` выполняет выход по GET-запросу, что позволяет инициировать его сторонним запросом.
5. **Парольные валидаторы Django не применяются.** `RegistrationForm` — обычная `forms.Form`, а не `UserCreationForm`; `AUTH_PASSWORD_VALIDATORS` из настроек не задействованы.
6. **E-mail не уникален на уровне БД.** Проверка дубля выполняется только в форме; стандартная `auth_user.email` не имеет `unique=True`. Регистрация в обход формы (например, через ORM/админку) может создать дубли.

### 15.2. Целостность данных

7. **`uid` отсутствует в API-сериализаторе.** При создании через API поле не передаётся; Django подставляет пустую строку, а `unique=True` не позволяет создать более одной такой записи. Первый API-created объект получит `uid=""`, второй — ошибку целостности.
8. **`get_or_create` в `add_shares` использует все поля как lookup.** Если у существующей акции изменилось название или другой атрибут, `get_or_create` попытается создать новую запись и столкнётся с уникальностью `figi`/`ticker`/`uid`. Команда не обновляет существующие записи.
9. **`updated_at` не обновляется Celery-задачей.** `QuerySet.update()` обходит `Model.save()` и `auto_now`, поэтому на детальной странице отображается время последнего изменения через форму/API, а не время обновления котировки.
10. **Денежные значения теряют точность.** Используются `FloatField` и только `Quotation.units`, без `nano`; дробная часть цены (копейки/центы) отбрасывается.
11. **`isin` не уникален, `ticker` ограничен 10 символами.** Это не соответствует реальным идентификаторам и может приводить к дублям или ошибкам импорта.
12. **Рыночные поля редактируются вручную.** `ShareModelForm` с `fields = "__all__"` позволяет пользователю менять `last_price`, `is_trend_high` и `uid`, хотя эти значения должны поступать из Tinkoff API.

### 15.3. Функциональные дефекты

13. **`AuthView` не валидирует форму.** Используются `form.data["email"]` и `form.data["password"]`; при отсутствии поля возникает `KeyError` и HTTP 500. `form.is_valid()` не вызывается.
14. **Регистрация не нормализует e-mail.** Пользователь создаётся с `username=email` без приведения к нижнему регистру, а вход выполняет `email.lower()`. E-mail с заглавными буквами может быть зарегистрирован, но не позволит войти.
15. **`RegistrationForm.clean` может упасть при ошибках полей.** После `super().clean()` код обращается к `cleaned_data["email"]`, `["password1"]`, `["password2"]` без проверки; при ошибке валидации поля это `KeyError`.
16. **`RegistrationView` извлекает только `form.errors["__all__"]`.** При ошибках конкретных полей ключа `__all__` может не быть, что приведёт к `KeyError`.
17. **Детальная страница не обрабатывает отсутствие акции.** `Share.objects.get(id=pk)` без `get_object_or_404`; несуществующий ID даёт HTTP 500 вместо 404.
18. **Детальная страница не обрабатывает ошибки Tinkoff API.** Любая сетевая/авторизационная ошибка при построении графика приводит к HTTP 500.
19. **Неоднозначность `is_trend_high`.** При первой успешной загрузке цены сравнение с `NULL` вызывает исключение, после чего флаг принудительно становится `True`; равенство цен также трактуется как `True`. Семантика «цена растёт» фактически означает «цена не снизилась».
20. **`NULL` в `is_trend_high` отображается как снижение.** В шаблоне `{% if share.is_trend_high %}` для `NULL` выбирается розовый цвет, хотя значение ещё не рассчитано.

### 15.4. Производительность и надёжность

21. **Celery-задача запускается каждые 10 секунд, хотя называется `add-every-5-min`.** Для каждой акции выполняется отдельный gRPC-вызов и отдельный `UPDATE`; при большом справочнике это создаёт высокую нагрузку и может исчерпать лимиты Tinkoff API.
22. **Нет повторных попыток и обработки rate limit.** Ошибки логируются, но задача не использует retry/backoff; `tenacity` не задействован.
23. **Нет result backend.** Результаты и статусы задач не сохраняются, мониторинг выполнения ограничен логами.
24. **График строится синхронно на каждый запрос.** Запрос 180 дней свечей выполняется в обработчике HTTP; данные не кэшируются и не сохраняются.
25. **Plotly-график может дублировать библиотеку.** В `base.html` уже подключён Plotly CDN, а `plotly.offline.plot` по умолчанию встраивает JS в HTML-фрагмент.
26. **Нет атомарности массового обновления.** При сбое посередине Celery-цикла часть акций обновлена, часть — нет.

### 15.5. Сопровождение и совместимость

27. **Устаревшие зависимости.** Django 4.1.7, Celery 5.2.7, `protobuf` 3.20.3, `grpcio` 1.51.3, `tinkoff-investments` beta; возможны проблемы на современном Python.
28. **Неполный `docker-compose.yml`.** Нет сервисов приложения, worker и beat; данные PostgreSQL не защищены именованным volume; образы `postgres` и `redis` не зафиксированы по версии.
29. **Рассогласование frontend-стека.** Bootstrap 5.1.3 + crispy-bootstrap3; подключены неиспользуемые jQuery, Popper, Font Awesome, Moment.js, Tempus Dominus; две копии Bootstrap и два независимых base-шаблона.
30. **Настройки логирования и локализации непоследовательны.** Лог пишется в `app.logs` без ротации; язык `en-us` при русском интерфейсе; Django в UTC, Celery — в Europe/Moscow.
31. **Нет production-конфигурации.** Отсутствуют `STATIC_ROOT`, настройки безопасности, WSGI/ASGI-сервер в зависимостях, разделение dev/prod-настроек.
32. **Тестовое покрытие узкое.** Нет тестов аутентификации, прав доступа, Celery, сервисов и management-команд; тесты API не обнаруживают отсутствие авторизации.

---

## 16. Выводы

`Investing Demo` — компактный учебный монолит на Django, который демонстрирует полный цикл работы с внешним биржевым API: импорт справочника, фоновое обновление котировок, отображение каталога, построение графиков, пользовательскую аутентификацию и REST API.

**Сильные стороны текущей структуры:**

- чёткое разделение на проект `config` и приложения `dashboard`/`user`;
- наличие сервисного слоя для Tinkoff API и построения графиков;
- использование стандартных механизмов Django (ORM, формы, CBV, админка, миграции);
- работоспособная схема асинхронного обновления через Celery Beat/Worker;
- покрытие основных CRUD-сценариев API тестами;
- простая и понятная модель данных, удобная как отправная точка для расширения.

**Ограничения, которые важно учитывать при развитии:**

- каталог акций глобальный, без связи с пользователем, поэтому текущая модель не поддерживает персональные списки наблюдения, портфели и уведомления;
- хранится только последняя цена, истории котировок нет;
- API не защищён, а веб-права не разграничены по объектам;
- денежные значения и рыночные поля реализованы упрощённо;
- конфигурация не готова к production, зависимости устарели;
- значительная часть бизнес-логики распределена между представлениями, задачей и сервисами без явного прикладного слоя.

Для перехода к персональному мониторингу, виртуальному портфелю, истории цен или симулятору стратегий потребуется прежде всего:

1. добавить связь `Share` с `User` (или отдельные сущности watchlist/portfolio/position);
2. ввести модель истории цен и накапливать котировки в БД;
3. вынести расчёт тренда и работу с Tinkoff API в единый сервисный слой;
4. закрыть API аутентификацией и разграничением прав;
5. заменить `FloatField` на точный денежный тип и учитывать `units` + `nano`;
6. обновить зависимости и конфигурацию, добавить переменные окружения и production-настройки;
7. расширить тестовое покрытие на аутентификацию, права, фоновые задачи и внешние интеграции.

Таким образом, проект представляет собой рабочую демонстрационную основу с понятной архитектурой, но с существенным техническим долгом, который необходимо устранить перед добавлением новой функциональности.
