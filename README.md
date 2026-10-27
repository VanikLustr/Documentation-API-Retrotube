# RetroTube API v1

> **Неофициальная документация.** Публичного описания API у retrotube.net не найдено. Этот справочник составлен по результатам анализа живого API 27 октября 2026 года и открытой OpenAPI-спецификации сообщества из репозитория [retrotube-cli](https://github.com/RootGuardian10/retrotube-cli). Сайт находится в статусе **Beta**, поэтому поведение и набор полей могут меняться без предупреждения.

| Параметр | Значение |
|---|---|
| Базовый URL | `https://retrotube.net/api/v1` |
| Версия | v1 |
| Формат | JSON (UTF-8); вход и регистрация — `application/x-www-form-urlencoded`; загрузка файлов — `multipart/form-data` |
| Авторизация | cookie `accessToken` + `refreshToken` (JWT) |
| Лимит запросов | 1000 запросов / ~15 минут на IP; 10–15 запросов на отдельный маршрут |
| Пагинация | `page` + `limit` (1–25) |
| Загрузка видео | до 200 МБ, не более 10 видео за 24 часа |
| CORS | не настроен: API рассчитан на same-origin или серверные клиенты |

---

## Содержание

1. [Обзор](#1-обзор)
2. [Быстрый старт](#2-быстрый-старт)
3. [Идентификаторы](#3-идентификаторы)
4. [Аутентификация и сессия](#4-аутентификация-и-сессия)
5. [Общие правила](#5-общие-правила)
6. [Справочник эндпоинтов](#6-справочник-эндпоинтов)
7. [Загрузка видео: полный цикл](#7-загрузка-видео-полный-цикл)
8. [Live и Socket.IO](#8-live-и-socketio)
9. [Модели данных](#9-модели-данных)
10. [Коды ошибок](#10-коды-ошибок)
11. [Подводные камни](#11-подводные-камни)
12. [Примеры клиентов](#12-примеры-клиентов)
13. [Источники и методика](#13-источники-и-методика)

---

## 1. Обзор

RetroTube — видеохостинг, стилизованный под начало 2010-х. API повторяет основные функции сайта:

- каталог видео, категории, рекомендуемые видео, поиск;
- загрузка, кодирование, публикация и редактирование видео;
- профили пользователей, подписки, уведомления, статистика канала;
- комментарии, ответы, закрепление, лайки и дизлайки;
- плейлисты;
- посты сообщества и опросы;
- live-стримы и модерация (REST + Socket.IO);
- жалобы на контент.

### 1.1. Кратко

- **REST**, JSON, версия в пути: `/api/v1`.
- **Авторизация — cookie.** Токены `accessToken` (короткоживущий) и `refreshToken` (1 год) выставляет сервер; вручную обновлять access-токен не нужно.
- **Почти все списки требуют `page` и `limit`.** Максимум `limit` — 25.
- **Три вида идентификаторов:** ObjectId, URL-ид видео и username. Их нельзя путать.
- **Ошибки имеют машиночитаемый `code`** и локализуемый текст `errors[].msg`.
- **Загрузка видео — четырёхшаговый процесс:** черновик → файл → кодирование → публикация.

### 1.2. Чего в API нет

- Официальной публичной документации и OpenAPI-файла на самом сайте.
- CORS-заголовков: браузерный JavaScript с другого origin не сможет читать ответы.
- REST-методов добавления/снятия модераторов live-стрима — только Socket.IO.
- Отдельных методов для ленты «рекомендации», трендов, тегов, черновиков и административной панели. Некоторые похожие пути существуют, но возвращают ошибку (см. раздел 11).

---

## 2. Быстрый старт

### 2.1. Публичный запрос

```bash
curl -s "https://retrotube.net/api/v1/videos?page=1&limit=2&sort=newest"
```

Ответ:

```json
{
  "items": [
    {
      "_id": "6ae02ad490aa27b3ebcd3cd1",
      "url": "VQcp3UEdiu7dz",
      "title": "БЫСТРЫЙ Обзор на RetroTube CLI",
      "views": 13,
      "likes": 2,
      "category": "Technology",
      "formattedLength": "3:13",
      "status": "published",
      "deletion": "none"
    }
  ],
  "totalPages": 1984,
  "isLastPage": false
}
```

Поля в примере сокращены; полная модель — в разделе 9.

### 2.2. Вход и сохранение cookie

```bash
curl -s -c cookies.txt -X POST "https://retrotube.net/api/v1/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "username=yourlogin" \
  --data-urlencode "password=YourPassword1"

curl -s -b cookies.txt "https://retrotube.net/api/v1/auth/session"
```

`curl -c` сохраняет `Set-Cookie`, `-b` отправляет их обратно. Успешный вход возвращает HTML, а не JSON, поэтому не пытайтесь разобрать тело как JSON — ориентируйтесь на статус `200` и cookie.

### 2.3. Загрузка видео (кратко)

```bash
# 1. Черновик
curl -s -b cookies.txt -X POST "https://retrotube.net/api/v1/videos" \
  -H "Content-Type: application/json" \
  -d '{"title":"Мой ролик"}'

# 2. Файл (подставьте _id из ответа)
curl -s -b cookies.txt -X POST "https://retrotube.net/api/v1/videos/$VIDEO_ID/upload" \
  -F "video=@movie.mp4;type=video/mp4"

# 3. Статус кодирования
curl -s -b cookies.txt "https://retrotube.net/api/v1/videos/$VIDEO_ID/status"

# 4. Публикация
curl -s -b cookies.txt -X POST "https://retrotube.net/api/v1/videos/$VIDEO_ID/publish"
```

Подробнее — в разделе 7.

---

## 3. Идентификаторы

В API одновременно используются три вида идентификаторов. Это главный источник ошибок `400` и `404`.

| Вид | Формат | Где используется | Пример |
|---|---|---|---|
| **ObjectId** | 24 hex-символа | `ref`, `commentId`, `postId`, `playlistId`, `channelId`, `liveChannelId`, а также `upload`, `status`, `publish` | `6ad40af79010a953622cfa46` |
| **URL-ид видео** | 10–20 символов base64url, обычно 11 | `/videos/{videoId}`, `/videos/{videoId}/thumbnail`, `/playlists/{p}/videos/{videoId}` | `JS8iyxhxktlY-` |
| **Username** | `[a-zA-Z0-9]{3,16}` | `/users/{username}`, `/community/{username}/posts` | `klodska` |

> ⚠️ **Важно:** в `ref` передаётся **ObjectId**, а не URL-ид видео. `ref=JS8iyxhxktlY-` вернёт `400 ID_INVALID`. URL-ид видео — это значение из ссылки `/watch?v=…`; ObjectId видео — поле `_id` в ответе API.

Дополнительно:

- `videoId` в `POST /videos/{videoId}/upload`, `GET /videos/{videoId}/status` и `POST /videos/{videoId}/publish` — это **ObjectId**, а не URL-ид.
- `channelId` в `/engagement/...` — ObjectId пользователя-канала (его `_id`), а не username.
- `playlistId`, `postId`, `commentId`, `liveChannelId` — всегда ObjectId.
- `index` в `/videos/{videoId}/thumbnail/{index}` — целое число, индекс в массиве `agThumbnails`, **начиная с 0**.

---

## 4. Аутентификация и сессия

### 4.1. Cookie-модель

| Cookie | Срок жизни | Содержимое | Назначение |
|---|---|---|---|
| `accessToken` | по спецификации **5 секунд** | JWT с данными пользователя | Авторизация запросов |
| `refreshToken` | **1 год** | JWT только с `sessionId` | Основной ключ сессии; по нему сервер выдаёт новый access-токен |
| `i18next` | 1 год | код языка (`en`, `ru`, `be`, `pl`) | Локализация интерфейса и сообщений об ошибках |

Сервер **сам продлевает** `accessToken` на любом запросе с валидным `refreshToken` и присылает новый `Set-Cookie`. Вручную обновлять токен не нужно. Именно поэтому `GET /auth/session` может вернуть `200` даже после истечения 5-секундного access-токена.

Клиент должен:

1. Сохранять cookie, которые присылает сервер (например, cookie-jar в `curl` или `requests.Session`).
2. Отправлять их с каждым запросом.
3. Не пытаться разбирать JWT самостоятельно — формат и состав полей могут меняться.

### 4.2. Вход

#### `POST /auth/login`

**Доступ:** публичный.

Аутентифицирует пользователя и выставляет cookie `accessToken`, `refreshToken` и `i18next`.

| Поле (form-urlencoded) | Тип | Обяз. | Описание |
|---|---|---|---|
| `username` | string | да | Логин: 3–16 символов, латиница и цифры. |
| `password` | string | да | Пароль: 8–32 символа, минимум одна цифра, одна строчная и одна заглавная буква. |

**Успех:** `200 text/html` — обычный form-POST с редиректом; тело — HTML-страница, а не JSON. Cookie установлены.

**Ошибки:** `400` (`USERNAME_REQUIRED`, `USERNAME_INVALID`, `PASSWORD_REQUIRED`, `WRONG_PASSWORD`, `RESOURCE_NOT_FOUND` — пользователя нет), `429 TOO_MANY_REQUESTS`, `500`.

```bash
curl -i -c cookies.txt -X POST "https://retrotube.net/api/v1/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "username=yourlogin" \
  --data-urlencode "password=YourPassword1"
```

> На `/auth/login` действует отдельный лимит — около 15 запросов на маршрут.

### 4.3. Регистрация

#### `POST /auth/signup`

**Доступ:** публичный.

Создаёт аккаунт. Тело — `application/x-www-form-urlencoded`.

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `username` | string | да | 3–16 символов, латиница и цифры. |
| `email` | string | да | E-mail. |
| `password` | string | да | 8–32 символа, цифра + строчная + заглавная. |
| `retype-password` | string | нет | Повтор пароля (используется формой сайта). |
| `terms` | boolean | да | Подтверждение возраста ≥ 13 лет и согласия с Terms/Privacy. |
| `g-recaptcha-response` | string | нет | Токен reCAPTCHA, если требуется. |

**Ответ:** `201` (тело не описано). После регистрации на e-mail приходит ссылка подтверждения, действующая 24 часа.

**Ошибки:** `CAPTCHA_REQUIRED`, `CAPTCHA_FAILED`, `USERNAME_TAKEN`, `EMAIL_TAKEN`, `EMAIL_INVALID`, `PASSWORD_INVALID`, `PASSWORDS_NOT_MATCH`, `TERMS_REQUIRED`, `TERMS_INVALID`, `TOO_MANY_REQUESTS`, `500`.

### 4.4. Текущая сессия

#### `GET /auth/session`

**Доступ:** требуется авторизация.

Возвращает данные авторизованного пользователя. Ответ обёрнут в `user`:

```json
{
  "user": {
    "_id": "69f87d6c309a8da85e884c61",
    "username": "delovoy24",
    "displayName": "GrabberVIII",
    "roles": ["user"],
    "status": "active",
    "isVerified": true,
    "hasPremium": false,
    "sessionId": "…"
  }
}
```

Иногда внутри дополнительно присутствуют поля access-JWT (`type`, `iat`, `exp`) — набор нестабилен. `401` — нет действующей сессии.

### 4.5. Выход

#### `POST /auth/logout`

**Доступ:** требуется авторизация.

Удаляет cookie `accessToken` и `refreshToken`. Ответ — `{"success": true}`. Последующий `GET /auth/session` вернёт `401`.

### 4.6. Требования к паролю

Пароль должен содержать:

- 8–32 символа;
- хотя бы одну цифру;
- хотя бы одну строчную латинскую букву;
- хотя бы одну заглавную латинскую букву.

Регулярное выражение из спецификации: `^(?=.*\d)(?=.*[a-z])(?=.*[A-Z]).{8,32}$`.

---

## 5. Общие правила

### 5.1. Форматы запросов

| Тип операции | Content-Type |
|---|---|
| Обычные запросы и JSON-тела | `application/json` |
| Вход и регистрация | `application/x-www-form-urlencoded` |
| Загрузка видео, обложки, аватара, вложения комментария, создание плейлиста | `multipart/form-data` |

Query-параметры (`page`, `limit`, `ref`, `refType`, `query`, `type`, `sort`, `category`, `channelId`) передаются в строке запроса. Для лайков, дизлайков, снятия оценки, комментариев и жалоб `ref`/`refType` — **query-параметры**, а не поля JSON-тела (кроме `POST /reports`, где они в JSON).

Заголовок `Accept: application/json` желателен, но не обязателен. Ответы — UTF-8.

### 5.2. Пагинация

Почти все списки используют одну схему:

```json
{
  "items": [],
  "totalPages": 1984,
  "isLastPage": false
}
```

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Номер страницы, начиная с 1. |
| `limit` | integer 1–25 | да | Количество элементов на странице. |

Правила:

- `page` и `limit` **обязательны** для большинства списков. Если не передать оба, сервер вернёт `400 PAGE_INVALID` и два сообщения — про страницу и про лимит.
- `limit=26` и больше → `400 LIMIT_INVALID`.
- `page=0` или отрицательный → `400 PAGE_INVALID`.
- `totalPages` — общее число страниц; `isLastPage` — признак последней страницы.
- У некоторых списков (например, пустые плейлисты) наблюдалось `totalPages: 0` при `isLastPage: false` — не полагайтесь на согласованность этих полей, проверяйте `items.length`.
- `GET /videos/featured`, `GET /comments/pinned`, `GET /videos/categories` и `GET /users/{u}/notifications/unread` возвращают **не** пагинированный объект, а массив/объект/boolean.

### 5.3. Сортировка, фильтры и категории

| Параметр | Значения | Где |
|---|---|---|
| `sort` | `popular`, `newest`, `oldest` | каталог, комментарии |
| `timeRange` | `all`, `hour`, `day`, `week`, `month` | каталог |
| `restrictionSort` | `all-ages`, `all` | каталог (чувствительный контент) |
| `category` | одна из 12 категорий | каталог, поиск, редактирование видео |
| `type` | `videos`, `users` | поиск |
| `hidden` | boolean | видео/плейлисты пользователя (только владелец) |

Категории (фиксированный набор):

`Gaming`, `Music`, `Art`, `Comedy`, `Entertainment`, `News`, `Sports`, `Technology`, `Travel`, `Tutorial`, `Animals`, `Science`.

### 5.4. Лимиты запросов

Действуют два независимых счётчика:

| Счётчик | Лимит | Область |
|---|---|---|
| Глобальный | 1000 запросов / ~15 минут | по IP |
| По маршруту | 10 запросов / окно (для `/auth/login` — около 15) | на каждый маршрут отдельно |

При превышении — `429 Too Many Requests`. Заголовки ответа:

- `X-RateLimit-Limit` — лимит;
- `X-RateLimit-Remaining` — остаток;
- `X-RateLimit-Reset` — Unix-время сброса (секунды);
- `Retry-After` — сколько ждать (на `429`).

Пример: 11-й запрос к `GET /search` в одном окне может вернуть `429`, даже если глобальный лимит ещё не исчерпан. Учитывайте это в циклах и при опросе статуса кодирования.

### 5.5. Ошибки

Формат ошибки:

```json
{
  "code": "PAGE_INVALID",
  "errors": [
    { "msg": "Invalid page number." },
    { "msg": "Invalid limit." }
  ]
}
```

| HTTP | Значение |
|---|---|
| `400` | Ошибка валидации, неверный идентификатор, неверное тело |
| `401` | Нет действующей сессии |
| `403` | Недостаточно прав (не владелец, не админ/модератор, кулдаун) |
| `404` | Ресурс не найден **или** неверная структура пути |
| `429` | Превышен лимит запросов |
| `500` | Внутренняя ошибка сервера |

Два разных `404`:

1. **Ресурс не существует:** `{"code":"RESOURCE_NOT_FOUND","errors":[{"msg":"Resource doesn't exist."}]}`.
2. **Маршрут не существует:** `{"errors":[{"msg":"API route not found"}]}` — **без поля `code`**. Так отвечает, например, `GET /users/{u}/subscribers`.

Локализация: текст `errors[].msg` зависит от cookie `i18next` (или языка сайта). Например, с `i18next=ru` приходит «Укажите правильный номер страницы.». **Ориентируйтесь на `code`, а не на текст.**

Исключение: `PATCH /users/{username}` при отсутствии изменений возвращает `200` с `{"code":"NO_CHANGES_MADE"}` — это не ошибка HTTP, но тело выглядит как ошибка.

### 5.6. CORS

По состоянию на 27.10.2026 API **не отдаёт** `Access-Control-Allow-Origin`. Preflight `OPTIONS` возвращает `200` и заголовок `Allow`, но без CORS-разрешений. Следствия:

- вызывать API из браузера можно только с того же origin (`https://retrotube.net`);
- сторонний фронтенд должен ходить через свой backend-прокси;
- `fetch` с `credentials: "include"` работает только в same-origin сценарии.

### 5.7. Файлы, URL и текст

API возвращает не полные URL файлов, а имена/пути. Собирайте URL так:

| Поле | URL |
|---|---|
| `avatarPath` | `https://retrotube.net/uploads/avatars/{avatarPath}` |
| `bannerPath` | `https://retrotube.net/uploads/banners/{bannerPath}` (по аналогии с сайтом) |
| `backgroundPath` | `https://retrotube.net/uploads/backgrounds/{backgroundPath}` (по аналогии с сайтом) |
| `thumbnailPath` | `https://retrotube.net/uploads/thumbnails/{thumbnailPath}` |
| `filePath` | `https://retrotube.net/uploads/videos/{filePath}` |
| `badgePath` | `https://retrotube.net/images/icons/badges/{badgePath}` |

Если `avatarPath` равен `default.png`, используйте `/uploads/avatars/default.png`.

Тексты (`bio`, комментарии, описания) могут приходить **HTML-экранированными**: например, `https:&#x2F;&#x2F;example.com`. Декодируйте HTML-сущности для показа, но не вставляйте пользовательский текст как HTML без санитизации.

Время — ISO 8601 в UTC (`2026-10-27T04:38:44.612Z`). Длительность видео — секунды (число с плавающей точкой).

---

## 6. Справочник эндпоинтов

Ниже описаны **69 операций на 49 путях**. Если у эндпоинта не указано иное, действуют стандартные ответы: `401` — нет сессии, `403` — нет прав, `404` — ресурс не найден, `429` — лимит запросов, `500` — внутренняя ошибка. Специфические коды `400` перечислены в каждом разделе.

### 6.1. Auth

| Метод | Путь | Назначение | Доступ |
|---|---|---|---|
| `POST` | `/auth/login` | Вход | публичный |
| `POST` | `/auth/signup` | Регистрация | публичный |
| `GET` | `/auth/session` | Текущая сессия | auth |
| `POST` | `/auth/logout` | Выход | auth |

Все четыре операции подробно описаны в разделе 4: cookie-модель, тела запросов, ответы и ошибки.

---

### 6.2. Videos

| Метод | Путь | Назначение | Доступ |
|---|---|---|---|
| `GET` | `/videos` | Каталог видео | публичный |
| `POST` | `/videos` | Создать черновик | auth |
| `GET` | `/videos/categories` | Категории | публичный |
| `GET` | `/videos/featured` | Рекомендуемые | публичный |
| `GET` | `/videos/{videoId}` | Видео по URL-иду | публичный |
| `PATCH` | `/videos/{videoId}` | Редактировать видео | auth, владелец |
| `DELETE` | `/videos/{videoId}` | Удалить видео | auth, владелец |
| `POST` | `/videos/{videoId}/upload` | Загрузить файл | auth, владелец |
| `GET` | `/videos/{videoId}/status` | Статус кодирования | auth, владелец |
| `POST` | `/videos/{videoId}/publish` | Опубликовать | auth, владелец |
| `PATCH` | `/videos/{videoId}/thumbnail` | Загрузить обложку | auth, владелец |
| `PATCH` | `/videos/{videoId}/thumbnail/{index}` | Выбрать авто-обложку | auth, владелец |
| `GET` | `/videos/{videoId}/comments` | Комментарии к видео | публичный |

#### `GET /videos`

Каталог видео с фильтрами и пагинацией.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Номер страницы. |
| `limit` | integer 1–25 | да | Элементов на странице. |
| `sort` | `popular` / `newest` / `oldest` | нет | Сортировка. |
| `timeRange` | `all` / `hour` / `day` / `week` / `month` | нет | Период выборки. |
| `restrictionSort` | `all-ages` / `all` | нет | Показывать ли чувствительный контент. |
| `category` | одна из 12 категорий | нет | Фильтр по категории. |

**Ответ `200`:** `VideoList` — `{ items: Video[], totalPages, isLastPage }`. В списке присутствуют `status`, `deletion` и `formattedLength`; в детальном ответе этих полей нет.

**Ошибки:** `400 PAGE_INVALID` / `LIMIT_INVALID`, `429`, `500`.

```bash
curl -s "https://retrotube.net/api/v1/videos?page=1&limit=10&sort=popular&timeRange=week&category=Gaming"
```

#### `POST /videos`

Создаёт черновик и резервирует URL-ид. Первый шаг загрузки.

| Поле JSON | Тип | Обяз. | Описание |
|---|---|---|---|
| `title` | string | да | Рабочее название, 3–50 символов. |

**Ответ `200`:**

```json
{
  "_id": "6ae02ad490aa27b3ebcd3cd1",
  "url": "VQcp3UEdiu7dz",
  "title": "Мой ролик",
  "uploadedBy": "69f87d6c309a8da85e884c61"
}
```

`uploadedBy` здесь — **строка ObjectId**, а не объект.

**Ограничения:** не более 10 видео за 24 часа (`TOO_MANY_VIDEOS`).

**Ошибки:** `400 TITLE_INVALID`, `401`, `429`, `500`.

#### `GET /videos/categories`

Возвращает фиксированный список категорий:

```json
{
  "categories": ["Gaming", "Music", "Art", "Comedy", "Entertainment", "News",
                 "Sports", "Technology", "Travel", "Tutorial", "Animals", "Science"]
}
```

**Ошибки:** `429`.

#### `GET /videos/featured`

Возвращает **голый массив** видео, без пагинации:

```json
[
  { "_id": "…", "url": "…", "title": "…", "views": 44, "likes": 13 }
]
```

**Ошибки:** `429`.

#### `GET /videos/{videoId}`

Одно видео. `videoId` — **URL-ид** из ссылки `/watch?v=…`, не ObjectId.

**Ответ `200`:** объект `Video` (детальная форма). В нём нет `status`, `deletion` и `formattedLength`; `uploadedBy` — объект `VideoAuthor`.

**Ошибки:** `400 URL_INVALID` (неверный формат URL-ида), `404 RESOURCE_NOT_FOUND`, `429`.

```bash
curl -s "https://retrotube.net/api/v1/videos/VQcp3UEdiu7dz"
```

#### `PATCH /videos/{videoId}`

Изменяет метаданные видео. `videoId` — URL-ид. Тело — JSON. Все поля опциональны.

| Поле | Тип | Описание |
|---|---|---|
| `title` | string | 3–50 символов. |
| `description` | string | Описание. |
| `tags` | string | Теги одной строкой через запятую. |
| `category` | string | Одна из 12 категорий. |
| `visibility` | `public` / `unlisted` / `private` | Видимость. |
| `isSensitive` | boolean | Чувствительный контент. |

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `TITLE_INVALID`, `DESCRIPTION_INVALID`, `DESCRIPTION_TOO_LONG`, `TAGS_TOO_LONG`, `CATEGORY_INVALID`, `VISIBILITY_REQUIRED`, `VISIBILITY_INVALID`, `NO_CHANGES_MADE`, `403`, `404`, `429`.

#### `DELETE /videos/{videoId}`

Удаляет видео. `videoId` — URL-ид. Ответ `{ "success": true }`. После удаления `GET /videos/{url}` вернёт `404`.

> Счётчик `videos` в `GET /users/{u}/stats` при удалении **не уменьшается** — это денормализованное значение.

**Ошибки:** `401`, `403`, `404`, `429`.

#### `POST /videos/{videoId}/upload`

Загружает видеофайл в черновик. `videoId` — **ObjectId** из ответа `POST /videos`.

| Поле multipart | Тип | Обяз. | Описание |
|---|---|---|---|
| `video` | file | да | Видеофайл. Имя поля — именно `video`, не `file`. |

**Ответ `200`:** `{ "success": true }`.

**Ограничения:** файл ≤ 200 МБ; не более 10 видео за 24 часа. Проверено на `video/webm` размером 8,9 МБ.

**Ошибки:** `VIDEO_FILE_REQUIRED`, `INVALID_FILE_TYPE`, `FILE_TOO_LARGE`, `VIDEO_ALREADY_UPLOADED`, `401`, `404`, `429`, `500`.

#### `GET /videos/{videoId}/status`

Статус обработки после загрузки. `videoId` — **ObjectId**.

**Ответ `200`:**

```json
{
  "success": true,
  "status": "encoding",
  "progress": 57
}
```

| Поле | Значения |
|---|---|
| `status` | `encoding`, `encoding_finished`, `encoding_failed` |
| `progress` | 0–100 |

Опрашивайте, пока `status` не станет `encoding_finished` (или `encoding_failed`). Учитывайте лимит на маршрут: не чаще одного запроса в несколько секунд.

**Ошибки:** `401`, `404`, `429`.

#### `POST /videos/{videoId}/publish`

Публикует видео после успешного кодирования. `videoId` — **ObjectId**. Ответ `{ "success": true }`.

**Ошибки:** `400` (например, `VIDEO_STATUS_MISSING`, `VIDEO_NOT_PUBLISHED`), `403`, `404`, `429`.

#### `PATCH /videos/{videoId}/thumbnail`

Загружает собственную обложку. `videoId` — URL-ид.

| Поле multipart | Тип | Обяз. | Описание |
|---|---|---|---|
| `thumbnail` | file | да | `image/jpeg` или `image/png`. |

**Ответ `200`:**

```json
{
  "success": true,
  "thumbnailPath": "17930775003724f8de585-20e8-40a2-80f3-24b842d36582.jpeg"
}
```

**Ошибки:** `THUMBNAIL_FILE_REQUIRED`, `INVALID_FILE_TYPE`, `FILE_TOO_LARGE`, `403`, `404`, `429`.

#### `PATCH /videos/{videoId}/thumbnail/{index}`

Выбирает одну из автоматически сгенерированных обложек. `videoId` — URL-ид; `index` — целое число **с 0** (позиция в массиве `agThumbnails`).

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `AG_THUMBNAILS_NOT_AVAILABLE`, `AG_INDEX_INVALID`, `403`, `404`, `429`.

#### `GET /videos/{videoId}/comments`

Алиас `GET /comments?refType=Video&ref=<ObjectId>`. `ref` подставляется сервером.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |
| `sort` | `popular` / `newest` / `oldest` | нет | Сортировка. |

**Ответ `200`:** `CommentList`.

**Ошибки:** `400 PAGE_INVALID` / `LIMIT_INVALID`, `404`, `429`.

---

### 6.3. Users

| Метод | Путь | Назначение | Доступ |
|---|---|---|---|
| `GET` | `/users` | Список пользователей | админ/модератор |
| `GET` | `/users/{username}` | Профиль | публичный |
| `PATCH` | `/users/{username}` | Изменить профиль | auth, владелец |
| `DELETE` | `/users/{username}` | Удалить аккаунт | auth, владелец |
| `GET` | `/users/{username}/videos` | Видео пользователя | публичный |
| `GET` | `/users/{username}/playlists` | Плейлисты пользователя | публичный |
| `GET` | `/users/{username}/comments` | Комментарии пользователя | админ/модератор |
| `GET` | `/users/{username}/subscriptions` | Подписки пользователя | админ/модератор |
| `GET` | `/users/{username}/subscriptions/videos` | Лента подписок | auth, владелец |
| `GET` | `/users/{username}/notifications` | Уведомления | auth, владелец |
| `POST` | `/users/{username}/notifications` | Отметить прочитанными | auth, владелец |
| `GET` | `/users/{username}/notifications/unread` | Есть непрочитанные | auth, владелец |
| `GET` | `/users/{username}/stats` | Статистика канала | auth, владелец |

#### `GET /users`

**Только для админа/модератора.** Обычный авторизованный пользователь получает `403 FORBIDDEN`.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |

**Ответ `200`:** `UserSearchResultList`.

**Ошибки:** `400`, `401`, `403`, `429`.

#### `GET /users/{username}`

Публичный профиль. `username` — логин, не ObjectId (с ObjectId придёт `400 USERNAME_INVALID`).

**Ответ `200`:** объект `User` (см. раздел 9). Поле `channel` содержит оформление канала: `profileTheme`, `featuredChannels`, `featuredVideo`.

**Ошибки:** `400 USERNAME_INVALID`, `404 RESOURCE_NOT_FOUND`, `429`.

```bash
curl -s "https://retrotube.net/api/v1/users/delovoy24"
```

#### `PATCH /users/{username}`

Изменяет профиль текущего пользователя. `username` должен совпадать с владельцем сессии.

| Поле JSON | Тип | Описание |
|---|---|---|
| `displayName` | string | Отображаемое имя. Может быть кулдаун на смену. |
| `bio` | string | Описание. **Пустая строка `""` игнорируется** — очистить bio через API нельзя. |
| `email` | string | Новый e-mail. |
| `currentPassword` | string | Текущий пароль (при смене пароля). |
| `newPassword` | string | Новый пароль. |
| `confirmPassword` | string | Подтверждение нового пароля. |
| `profileTheme` | object | Оформление профиля: `{ presetId, custom }`. `presetId` — `halloween`, `terminal`, `golden`, `lava`, `mariana_trench`, `bluedrop`, `midnight`; `custom` — `{ color }`. Часть пресетов требует Premium (`PROFILE_THEME_PREMIUM_REQUIRED`). |

**Ответ `200`:** `{ "success": true }`. Если изменений нет — `200` с `{ "code": "NO_CHANGES_MADE" }`.

**Ошибки:** `DISPLAY_NAME_REQUIRED`, `DISPLAY_NAME_INVALID`, `DISPLAY_NAME_INVALID_LENGTH`, `DISPLAY_NAME_CHANGE_NOT_ALLOWED`, `BIO_REQUIRED`, `BIO_TOO_LONG`, `EMAIL_INVALID`, `EMAIL_TAKEN`, `PROFILE_THEME_PRESET_INVALID`, `PROFILE_THEME_CUSTOM_INVALID`, `PROFILE_THEME_PREMIUM_REQUIRED`, `403`, `429`.

#### `DELETE /users/{username}`

Удаляет аккаунт. Маршрут существует (`Allow: DELETE, GET, HEAD, PATCH`), но **не проверялся** во избежание удаления тестового аккаунта. Ответ — `{ "success": true }`.

**Ошибки:** `401`, `403`, `429`.

#### `GET /users/{username}/videos`

Публичный список видео пользователя.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |
| `hidden` | boolean | нет | Показать скрытые (черновики/приватные). Только владелец. |

**Ответ `200`:** `VideoList`.

**Ошибки:** `400`, `429`.

#### `GET /users/{username}/playlists`

Публичный список плейлистов пользователя.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |
| `hidden` | boolean | нет | Показать скрытые. Только владелец. |

**Ответ `200`:** `PlaylistList`. В списке `owner` — **строка ObjectId** (в детальном `GET /playlists/{id}` — объект).

**Ошибки:** `400`, `429`.

#### `GET /users/{username}/comments`

**Только для админа/модератора**: даже на свой профиль обычный пользователь получает `403 FORBIDDEN`.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |

**Ответ `200`:** `CommentList`.

**Ошибки:** `400`, `401`, `403`, `429`.

#### `GET /users/{username}/subscriptions`

**Только для админа/модератора** (`403 FORBIDDEN` для обычного пользователя).

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |

**Ответ `200`:** `UserSearchResultList`.

**Ошибки:** `400`, `401`, `403`, `429`.

#### `GET /users/{username}/subscriptions/videos`

Лента видео из каналов, на которые подписан пользователь. Требует авторизации; `username` — свой.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |

**Ответ `200`:** `VideoList` + дополнительное поле `success` (по данным спецификации).

**Ошибки:** `401`, `429`.

#### `GET /users/{username}/notifications`

Список уведомлений. Требует авторизации; `username` — свой.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | фактически да | Если не передать, сервер может вернуть `400 PAGE_INVALID`, хотя в спецификации параметр помечен как необязательный. Передавайте `page=1`. |

**Ответ `200`:** `NotificationList`.

**Ошибки:** `400`, `401`, `429`.

#### `POST /users/{username}/notifications`

Помечает уведомления прочитанными. Требует авторизации; `username` — свой.

| Параметр query | Тип | Обяз. | Описание |
|---|---|---|---|
| `read` | boolean | да | `true` — пометить прочитанными. |

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 BOOLEAN_INVALID`, `401`, `429`.

#### `GET /users/{username}/notifications/unread`

Возвращает **голый boolean**:

```json
true
```

`true` — есть непрочитанные, `false` — нет. Требует авторизации.

**Ошибки:** `401`, `429`.

#### `GET /users/{username}/stats`

Сводная статистика **своего** канала. На чужой профиль — `403 FORBIDDEN`.

**Ответ `200`:** `UserStats` — объект с полями:

```json
{
  "videos": 12,
  "views": 345,
  "likes": 67,
  "dislikes": 2,
  "comments": 18,
  "subscribers": 4
}
```

Набор полей может расширяться. Счётчик `videos` не уменьшается при удалении видео.

**Ошибки:** `401`, `403`, `429`.

---

### 6.4. Comments

| Метод | Путь | Назначение | Доступ |
|---|---|---|---|
| `GET` | `/comments` | Комментарии к ресурсу | публичный |
| `POST` | `/comments` | Создать комментарий | auth |
| `GET` | `/comments/pinned` | Закреплённые | публичный |
| `PATCH` | `/comments/{commentId}` | Редактировать | auth, автор |
| `DELETE` | `/comments/{commentId}` | Удалить | auth, автор |
| `GET` | `/comments/{commentId}/replies` | Ответы | публичный |
| `POST` | `/comments/{commentId}/replies` | Ответить | auth |
| `POST` | `/comments/{commentId}/pin` | Закрепить | auth, владелец контента |
| `DELETE` | `/comments/{commentId}/pin` | Открепить | auth, владелец контента |

#### `GET /comments`

Комментарии к произвольному ресурсу. `ref` — **ObjectId** ресурса, `refType` — `Video` или `CommunityPost`.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `ref` | ObjectId (24 hex) | да | ObjectId видео или поста. |
| `refType` | `Video` / `CommunityPost` | да | Тип ресурса. |
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |
| `sort` | `popular` / `newest` / `oldest` | нет | Сортировка. |

**Ответ `200`:** `CommentList`.

**Ошибки:** `400 REF_REQUIRED`, `REF_TYPE_REQUIRED`, `REF_TYPE_INVALID`, `ID_INVALID`, `PAGE_INVALID`, `LIMIT_INVALID`, `429`.

```bash
curl -s "https://retrotube.net/api/v1/comments?ref=6ae02ad490aa27b3ebcd3cd1&refType=Video&page=1&limit=10&sort=newest"
```

#### `POST /comments`

Создаёт комментарий. `ref` и `refType` передаются **в query**, тело — `multipart/form-data`.

| Query | Тип | Обяз. | Описание |
|---|---|---|---|
| `ref` | ObjectId | да | ObjectId ресурса. |
| `refType` | `Video` / `CommunityPost` | да | Тип ресурса. |

| Поле multipart | Тип | Обяз. | Описание |
|---|---|---|---|
| `text` | string | да | Текст комментария. |
| `media` | file | нет | Вложение. |

**Ответ `200`:** `{ "success": true }` — **без `_id`** созданного комментария. Чтобы узнать id, перечитайте список с `sort=newest`.

**Ошибки:** `400 TEXT_REQUIRED`, `TEXT_TOO_LONG`, `TEXT_LENGTH_INVALID`, `401`, `429`, `500`.

```bash
curl -s -b cookies.txt -X POST \
  "https://retrotube.net/api/v1/comments?ref=6ae02ad490aa27b3ebcd3cd1&refType=Video" \
  -F "text=Отличное видео!"
```

#### `GET /comments/pinned`

Закреплённые комментарии. Возвращает **голый массив** (пустой `[]`, если закреплённых нет).

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `ref` | ObjectId | да | ObjectId ресурса. |
| `refType` | `Video` / `CommunityPost` | да | Тип ресурса. |

**Ответ `200`:** `Comment[]`.

**Ошибки:** `400`, `429`.

#### `PATCH /comments/{commentId}`

Редактирует комментарий. `commentId` — ObjectId. Тело — JSON.

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `text` | string | да | Новый текст. |

**Ответ `200`:** `{ "success": true }`. После успеха у комментария выставляется `isEdited: true`.

**Ошибки:** `400 TEXT_REQUIRED`, `TEXT_TOO_LONG`, `401`, `404`, `429`.

#### `DELETE /comments/{commentId}`

Удаляет комментарий или ответ. `commentId` — ObjectId.

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `401`, `404`, `429`.

#### `GET /comments/{commentId}/replies`

Список ответов на комментарий.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | фактически да | В спецификации необязателен, но без него приходит `400 LIMIT_INVALID`. Передавайте оба. |

**Ответ `200`:** `CommentList`.

**Ошибки:** `400`, `404`, `429`.

#### `POST /comments/{commentId}/replies`

Создаёт ответ. Тело — `multipart/form-data`.

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `text` | string | да | Текст ответа. |
| `media` | file | нет | Вложение. |

**Ответ `200`:** `{ "success": true }` — без `_id`.

**Ошибки:** `400`, `401`, `404`, `429`.

#### `POST /comments/{commentId}/pin`

Закрепляет комментарий. Право есть **только у владельца контента** (видео или поста); для чужого контента — `403 FORBIDDEN`. Ответы закреплять нельзя — `400 CANT_PIN_REPLY`.

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 CANT_PIN_REPLY`, `PINNED_INVALID`, `401`, `403`, `404`, `429`.

#### `DELETE /comments/{commentId}/pin`

Снимает закрепление. Те же права, что и у `POST` (иначе `403`).

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400`, `401`, `403`, `404`, `429`.

---

### 6.5. Engagement

| Метод | Путь | Назначение | Доступ |
|---|---|---|---|
| `POST` | `/engagement/like` | Лайк | auth |
| `POST` | `/engagement/dislike` | Дизлайк | auth |
| `DELETE` | `/engagement/rating` | Снять оценку | auth |
| `GET` | `/engagement/subscribers` | Подписчики канала | публичный |
| `GET` | `/engagement/{channelId}/subscription` | Состояние подписки | auth |
| `POST` | `/engagement/{channelId}/subscribers` | Подписаться | auth |
| `DELETE` | `/engagement/{channelId}/subscribers` | Отписаться | auth |
| `PATCH` | `/engagement/{channelId}/subscribers/notifications` | Уведомления канала | auth |

#### `POST /engagement/like`

Ставит лайк ресурсу. `ref` — ObjectId, `refType` — `Video` или `CommunityPost`.

| Query | Тип | Обяз. | Описание |
|---|---|---|---|
| `ref` | ObjectId | да | ObjectId ресурса. |
| `refType` | `Video` / `CommunityPost` | да | Тип ресурса. |

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 REF_REQUIRED`, `REF_TYPE_REQUIRED`, `ID_INVALID`, `401`, `429`.

```bash
curl -s -b cookies.txt -X POST \
  "https://retrotube.net/api/v1/engagement/like?ref=6ae02ad490aa27b3ebcd3cd1&refType=Video"
```

#### `POST /engagement/dislike`

Ставит дизлайк. Параметры и ответ аналогичны лайку.

**Ошибки:** `400`, `401`, `429`.

#### `DELETE /engagement/rating`

Снимает ранее поставленный лайк или дизлайк. Параметры те же: `ref`, `refType` в query.

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 RATING_NOT_UPDATED` (оценки не было), `401`, `429`.

#### `GET /engagement/subscribers`

Список подписчиков канала. `channelId` — ObjectId канала.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `channelId` | ObjectId (24 hex) | да | ObjectId канала (пользователя). |
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |

**Ответ `200`:** `SubscriberList`.

**Ошибки:** `400`, `429`.

> `GET /users/{username}/subscribers` **не существует** — используйте именно этот маршрут.

#### `GET /engagement/{channelId}/subscription`

Проверяет, подписан ли текущий пользователь на канал. Требует авторизации.

**Ответ `200`:**

```json
{
  "subscribed": true,
  "notificationsEnabled": true
}
```

> ⚠️ Если подписки **нет**, возвращается `404 RESOURCE_NOT_FOUND`, а не `{"subscribed": false}`. Так же отвечает несуществующий канал.

**Ошибки:** `401`, `404`, `429`.

#### `POST /engagement/{channelId}/subscribers`

Подписывает текущего пользователя на канал. `channelId` — ObjectId канала.

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 SELF_SUBSCRIPTION_NOT_ALLOWED` (нельзя подписаться на себя), `401`, `404`, `429`.

#### `DELETE /engagement/{channelId}/subscribers`

Отписывает текущего пользователя от канала.

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 SELF_UNSUBSCRIPTION_NOT_ALLOWED`, `401`, `404`, `429`.

#### `PATCH /engagement/{channelId}/subscribers/notifications`

Включает или выключает уведомления о новых видео канала. Тело — JSON.

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `notificationsEnabled` | boolean | да | `true` — включить, `false` — выключить. |

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 BOOLEAN_INVALID`, `401`, `404`, `429`.

---

### 6.6. Playlists

| Метод | Путь | Назначение | Доступ |
|---|---|---|---|
| `GET` | `/playlists` | Список всех плейлистов | админ/модератор |
| `POST` | `/playlists` | Создать плейлист | auth |
| `GET` | `/playlists/{playlistId}` | Плейлист | публичный |
| `PATCH` | `/playlists/{playlistId}` | Редактировать | auth, владелец |
| `DELETE` | `/playlists/{playlistId}` | Удалить | auth, владелец |
| `POST` | `/playlists/{playlistId}/videos/{videoId}` | Добавить видео | auth, владелец |
| `PUT` | `/playlists/{playlistId}/videos/{videoId}` | Обновить видео | auth, владелец |
| `DELETE` | `/playlists/{playlistId}/videos/{videoId}` | Убрать видео | auth, владелец |

#### `GET /playlists`

**Только для админа/модератора**: обычный авторизованный пользователь получает `403 FORBIDDEN`. Свои плейлисты — через `GET /users/{username}/playlists`.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |

**Ответ `200`:** `PlaylistList`.

**Ошибки:** `400`, `401`, `403`, `429`.

#### `POST /playlists`

Создаёт плейлист. Тело — `multipart/form-data`.

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `name` | string | да | Название. |
| `description` | string | нет | Описание. |
| `visibility` | `public` / `unlisted` / `private` | нет | Видимость. |
| `thumbnail` | file | нет | Обложка. |

**Ответ `200`:** `{ "success": true }` — **без `_id`**. Созданный плейлист ищите в `GET /users/{username}/playlists`.

**Ошибки:** `REACHED_PLAYLIST_LIMIT`, `VISIBILITY_REQUIRED`, `VISIBILITY_INVALID`, `401`, `429`, `500`.

```bash
curl -s -b cookies.txt -X POST "https://retrotube.net/api/v1/playlists" \
  -F "name=Мой плейлист" \
  -F "description=Лучшее" \
  -F "visibility=public"
```

#### `GET /playlists/{playlistId}`

Публичный плейлист. `playlistId` — ObjectId. Состав приходит в `items[]`; отдельного `GET /playlists/{id}/videos` **не существует**.

**Ответ `200`:** `Playlist`. В детальном ответе `owner` — объект `UserSummary`; в списке `GET /users/{u}/playlists` — строка ObjectId.

**Ошибки:** `400`, `404`, `429`.

#### `PATCH /playlists/{playlistId}`

Редактирует плейлист. Требует авторизации (владелец). Тело — JSON.

| Поле | Тип | Описание |
|---|---|---|
| `name` | string | Название. |
| `description` | string | Описание. |
| `visibility` | `public` / `unlisted` / `private` | Видимость. |

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400`, `401`, `404`, `429`.

#### `DELETE /playlists/{playlistId}`

Удаляет плейлист. Требует авторизации (владелец).

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `401`, `404`, `429`.

#### `POST /playlists/{playlistId}/videos/{videoId}`

Добавляет видео в плейлист. `playlistId` — ObjectId; `videoId` — **URL-ид** видео (не ObjectId).

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 VIDEO_ALREADY_IN_PLAYLIST`, `REACHED_VIDEO_LIMIT`, `401`, `404`, `429`.

#### `PUT /playlists/{playlistId}/videos/{videoId}`

> ⚠️ **Не работает:** запрос зависает без ответа. `OPTIONS` рекламирует `PUT` (`Allow: DELETE, POST, PUT`), но и с телом `{"order":1}`, и без тела сервер не отвечает дольше 12 секунд (проверено дважды). Метод помечен как deprecated. Не используйте его.

#### `DELETE /playlists/{playlistId}/videos/{videoId}`

Убирает видео из плейлиста. `videoId` — URL-ид.

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400`, `401`, `404`, `429`.

---

### 6.7. Community

| Метод | Путь | Назначение | Доступ |
|---|---|---|---|
| `POST` | `/community/posts` | Создать пост | auth (фактически 403) |
| `GET` | `/community/{username}/posts` | Посты пользователя | публичный |
| `GET` | `/community/posts/{postId}` | Пост по id | публичный |
| `PATCH` | `/community/posts/{postId}` | Редактировать пост | auth, автор |
| `DELETE` | `/community/posts/{postId}` | Удалить пост | auth, автор |
| `POST` | `/community/polls/{pollId}/vote` | Голосовать в опросе | auth |

#### `POST /community/posts`

Создаёт пост сообщества. Маршрут существует (`Allow: GET, HEAD, POST`), но для обычного аккаунта возвращает **`403 FORBIDDEN`** (проверено). Похоже, требуется особый статус канала; публичного способа включить это через API не обнаружено.

| Поле JSON | Тип | Обяз. | Описание |
|---|---|---|---|
| `content` | string | да | Текст поста. |

**Ответ `200`:** `CommunityPost` (по спецификации).

**Ошибки:** `400`, `401`, `403`, `429`.

#### `GET /community/{username}/posts`

Публичная лента постов сообщества.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице (сайт запрашивает `limit=6`). |

**Ответ `200`:** `CommunityPostList`.

**Ошибки:** `400`, `429`.

#### `GET /community/posts/{postId}`

Публичный просмотр отдельного поста. `postId` — ObjectId.

**Ответ `200`:** `CommunityPost`.

**Ошибки:** `404`, `429`.

#### `PATCH /community/posts/{postId}`

Редактирует пост. Требует авторизации (автор). Тело — JSON.

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `content` | string | да | Новый текст. |

**Ответ `200`:** `{ "success": true }`. После успеха `isEdited` становится `true`.

**Ошибки:** `400`, `401`, `404`, `429`.

#### `DELETE /community/posts/{postId}`

Удаляет пост. Требует авторизации (автор).

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `401`, `404`, `429`.

#### `POST /community/polls/{pollId}/vote`

Голосует в опросе. `pollId` — ObjectId. `options` — массив ObjectId выбранных опций; для single-choice передаётся один элемент.

| Поле JSON | Тип | Обяз. | Описание |
|---|---|---|---|
| `options` | string[] (ObjectId) | да | Выбранные варианты. |

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 INVALID_POLL`, `POLL_EXPIRED`, `INVALID_VOTE_OPTIONS`, `MULTIPLE_CHOICE_NOT_SUPPORTED`, `401`, `404`, `429`.

---

### 6.8. Live

| Метод | Путь | Назначение | Доступ |
|---|---|---|---|
| `GET` | `/live/{liveChannelId}` | Информация о стриме | публичный |
| `PATCH` | `/live/{liveChannelId}` | Изменить стрим | auth, владелец |
| `GET` | `/live/{liveChannelId}/moderators` | Модераторы | владелец/модератор |
| `GET` | `/live/{liveChannelId}/punishments` | Наказания | владелец/модератор |
| `GET` | `/live/{liveChannelId}/banned` | Забаненные | владелец/модератор |

#### `GET /live/{liveChannelId}`

Информация о live-канале. `liveChannelId` — ObjectId.

**Ответ `200`:** `LiveChannel`. Медиапотоки отдаются отдельно:

- HLS: `https://retrotube.net/live/hls/{liveChannelId}/index.m3u8`
- FLV: `https://retrotube.net/live/flv/{liveChannelId}`

**Ошибки:** `404`, `429`.

#### `PATCH /live/{liveChannelId}`

Изменяет данные стрима. **Только владелец** (иначе `403 FORBIDDEN`). Тело — JSON.

| Поле | Тип | Описание |
|---|---|---|
| `title` | string | 3–50 символов. |
| `description` | string | Описание. |

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 TITLE_INVALID`, `401`, `403`, `404`, `429`.

#### `GET /live/{liveChannelId}/moderators`

Список модераторов. **Только для владельца/модератора** — иначе `403 FORBIDDEN`.

**Ответ `200`:**

```json
{
  "moderators": [
    { "_id": "…", "username": "moder", "displayName": "Moder" }
  ]
}
```

REST-методов добавления/снятия модератора нет (`POST …/moderators` → 404): это делается через Socket.IO (событие `live:addModerator`).

**Ошибки:** `401`, `403`, `404`, `429`.

#### `GET /live/{liveChannelId}/punishments`

История банов и мутов. **Только для владельца/модератора** (`403 FORBIDDEN`).

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |
| `type` | `ban` / `mute` | нет | Тип наказания. |

**Ответ `200`:** `PunishmentList`.

**Ошибки:** `400`, `401`, `403`, `404`, `429`.

#### `GET /live/{liveChannelId}/banned`

Список забаненных пользователей. **Только для владельца/модератора** (`403 FORBIDDEN`).

Параметры те же: `page`, `limit`, `type` (`ban` / `mute`).

**Ответ `200`:** `PunishmentList`.

**Ошибки:** `400`, `401`, `403`, `404`, `429`.

---

### 6.9. Reports

| Метод | Путь | Назначение | Доступ |
|---|---|---|---|
| `GET` | `/reports` | Список жалоб | админ/модератор |
| `POST` | `/reports` | Подать жалобу | публичный |

#### `GET /reports`

Список жалоб. **Только для админа/модератора.**

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |

**Ответ `200`:** `ReportList`.

> При пустой выборке возвращается `404 PAGE_NOT_FOUND` даже для `page=1` (проверено) — «пусто» и «нет прав» неразличимы по коду.

**Ошибки:** `401`, `403`, `404`, `429`.

#### `POST /reports`

Публичный маршрут — жалобу может подать и гость. Тело — JSON.

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `ref` | ObjectId (24 hex) | да | ObjectId ресурса. |
| `refType` | `Video` / `CommunityPost` / `LiveChannel` | да | Тип ресурса. |
| `reason` | `spam` / `nudity` / `hate` / `harassment` / `copyright` / `other` | да | Причина. |
| `info` | string | да | Дополнительная информация. |

**Ответ `200`:** `{ "success": true }`.

**Ошибки:** `400 REF_REQUIRED`, `REF_TYPE_REQUIRED`, `ID_INVALID`, `REASON_REQUIRED`, `REASON_INVALID`, `INFO_REQUIRED`, `INFO_TOO_LONG`, `REPORT_NOT_RESOLVED` (предыдущая жалоба ещё не рассмотрена), `REPORT_ALREADY_RESOLVED`, `429`, `500`.

```bash
curl -s -X POST "https://retrotube.net/api/v1/reports" \
  -H "Content-Type: application/json" \
  -d '{"ref":"6ad40af79010a953622cfa46","refType":"Video","reason":"spam","info":"Спам в описании"}'
```

---

### 6.10. Search

#### `GET /search`

Ищет видео или пользователей.

| Параметр | Тип | Обяз. | Описание |
|---|---|---|---|
| `query` | string 1–128 | да | Поисковый запрос. **Параметр называется `query`, не `q`** — с `q` сервер отвечает `400 search.QUERY_REQUIRED`. |
| `type` | `videos` / `users` | да | Что искать. |
| `page` | integer ≥ 1 | да | Страница. |
| `limit` | integer 1–25 | да | Элементов на странице. |
| `category` | одна из 12 категорий | нет | Фильтр по категории (для видео). |

**Ответ `200`:**

- при `type=videos` — `VideoList`;
- при `type=users` — `UserSearchResultList` (элементы содержат `score`).

**Ошибки:** `400 search.QUERY_REQUIRED`, `search.TYPE_REQUIRED`, `search.TYPE_INVALID`, `PAGE_INVALID`, `LIMIT_INVALID`, `429`. Префикс `search.` — часть фактического `code`, а не название раздела.

```bash
curl -s "https://retrotube.net/api/v1/search?query=api&type=videos&page=1&limit=10"
curl -s "https://retrotube.net/api/v1/search?query=delovoy24&type=users&page=1&limit=10"
```

> Поиск пользователей чувствителен к строке: `delovoy24` находит пользователя, а `delovoy` может вернуть пустой список. Это серверная реализация релевантности, а не ошибка клиента.

---

## 7. Загрузка видео: полный цикл

Загрузка состоит из четырёх шагов. Первый и последующие шаги используют **разные идентификаторы**: `_id` (ObjectId) и `url` (URL-ид).

### Шаг 1. Создать черновик

```bash
curl -s -b cookies.txt -X POST "https://retrotube.net/api/v1/videos" \
  -H "Content-Type: application/json" \
  -d '{"title":"Мой ролик"}'
```

Ответ:

```json
{
  "_id": "6ae02ad490aa27b3ebcd3cd1",
  "url": "VQcp3UEdiu7dz",
  "title": "Мой ролик",
  "uploadedBy": "69f87d6c309a8da85e884c61"
}
```

Запомните `_id` — он нужен для загрузки, статуса и публикации. `url` пригодится для ссылки `/watch?v=…` и редактирования метаданных.

### Шаг 2. Загрузить файл

```bash
curl -s -b cookies.txt -X POST \
  "https://retrotube.net/api/v1/videos/$VIDEO_ID/upload" \
  -F "video=@movie.mp4;type=video/mp4"
```

Ответ: `{ "success": true }`.

Требования:

- поле называется **`video`** (не `file`);
- файл ≤ 200 МБ;
- не более 10 видео за 24 часа;
- тип должен быть видео (проверено на `video/webm`; для `video/mp4` используйте соответствующий MIME-тип).

### Шаг 3. Дождаться кодирования

```bash
curl -s -b cookies.txt \
  "https://retrotube.net/api/v1/videos/$VIDEO_ID/status"
```

Ответ:

```json
{
  "success": true,
  "status": "encoding",
  "progress": 57
}
```

| `status` | Значение |
|---|---|
| `encoding` | Идёт перекодирование; смотрите `progress`. |
| `encoding_finished` | Готово, можно публиковать. |
| `encoding_failed` | Ошибка кодирования. |

Опрашивайте с паузой (например, 3–5 секунд), чтобы не упереться в лимит на маршрут.

### Шаг 4. Опубликовать

```bash
curl -s -b cookies.txt -X POST \
  "https://retrotube.net/api/v1/videos/$VIDEO_ID/publish"
```

Ответ: `{ "success": true }`. После публикации видео доступно по адресу:

```text
https://retrotube.net/watch?v={url}
```

### Шаг 5. Метаданные и обложка (опционально)

```bash
# Метаданные: URL-ид, не ObjectId
curl -s -b cookies.txt -X PATCH \
  "https://retrotube.net/api/v1/videos/$VIDEO_URL_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "description":"Описание",
    "tags":"тег1, тег2",
    "category":"Technology",
    "visibility":"public",
    "isSensitive":false
  }'

# Своя обложка
curl -s -b cookies.txt -X PATCH \
  "https://retrotube.net/api/v1/videos/$VIDEO_URL_ID/thumbnail" \
  -F "thumbnail=@cover.jpg;type=image/jpeg"

# Или выбрать авто-обложку (index с 0)
curl -s -b cookies.txt -X PATCH \
  "https://retrotube.net/api/v1/videos/$VIDEO_URL_ID/thumbnail/0"
```

### Полный bash-скрипт

```bash
#!/usr/bin/env bash
set -euo pipefail
BASE="https://retrotube.net/api/v1"
COOKIES="cookies.txt"
FILE="${1:?usage: $0 file.mp4}"

# 1. Черновик
DRAFT=$(curl -s -b "$COOKIES" -X POST "$BASE/videos" \
  -H "Content-Type: application/json" \
  -d '{"title":"Загрузка через API"}')
ID=$(printf '%s' "$DRAFT" | python -c 'import json,sys; print(json.load(sys.stdin)["_id"])')
URL_ID=$(printf '%s' "$DRAFT" | python -c 'import json,sys; print(json.load(sys.stdin)["url"])')
echo "draft: $ID / $URL_ID"

# 2. Файл
curl -s -b "$COOKIES" -X POST "$BASE/videos/$ID/upload" \
  -F "video=@$FILE"
echo

# 3. Ожидание кодирования
while :; do
  STATUS=$(curl -s -b "$COOKIES" "$BASE/videos/$ID/status")
  echo "$STATUS"
  STATE=$(printf '%s' "$STATUS" | python -c 'import json,sys; print(json.load(sys.stdin)["status"])')
  [ "$STATE" = "encoding_finished" ] && break
  [ "$STATE" = "encoding_failed" ] && { echo "encoding failed"; exit 1; }
  sleep 5
done

# 4. Публикация
curl -s -b "$COOKIES" -X POST "$BASE/videos/$ID/publish"
echo
echo "https://retrotube.net/watch?v=$URL_ID"
```

---

## 8. Live и Socket.IO

### 8.1. REST-часть

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/live/{liveChannelId}` | Информация о стриме |
| `PATCH` | `/live/{liveChannelId}` | Заголовок и описание (владелец) |
| `GET` | `/live/{liveChannelId}/moderators` | Модераторы (владелец/модератор) |
| `GET` | `/live/{liveChannelId}/punishments` | Баны и муты (владелец/модератор) |
| `GET` | `/live/{liveChannelId}/banned` | Забаненные (владелец/модератор) |

Медиа:

- HLS: `https://retrotube.net/live/hls/{liveChannelId}/index.m3u8`
- FLV: `https://retrotube.net/live/flv/{liveChannelId}`

### 8.2. Подключение Socket.IO

Realtime-часть работает через Socket.IO на том же origin, путь по умолчанию `/socket.io`.

```text
GET /socket.io/?EIO=4&transport=polling
```

Ответ начинается с `0{...}`:

```json
0{"sid":"…","upgrades":["websocket"],"pingInterval":25000,"pingTimeout":20000,"maxPayload":1000000}
```

| Параметр | Значение |
|---|---|
| Engine.IO | 4 |
| Socket.IO | 4 |
| `pingInterval` | 25 000 мс |
| `pingTimeout` | 20 000 мс |
| `maxPayload` | 1 000 000 байт |

Клиент:

```js
const socket = io({ autoConnect: false });
socket.connect();
```

Рукопожатие работает и анонимно; действия чата и модерации требуют cookie-сессии. Неизвестные события сервер молча игнорирует (ответа нет).

### 8.3. События чата

**Клиент → сервер:**

| Событие | Payload | Описание |
|---|---|---|
| `chat:join` | `{ liveChannelId: ObjectId }` | Войти в комнату. **Обязательно перед `chat:message`**, иначе `CHAT_NOT_FOUND`. |
| `chat:message` | `{ content: string }` | Отправить сообщение. |
| `chat:setSlowMode` | `{ seconds: int }` | Включить/изменить slow mode (модератор). |
| `chat:delete` | `{ messageId: ObjectId }` | Удалить сообщение (модератор). |
| `chat:history` | `{}` | Запросить историю. |

**Сервер → клиент:**

| Событие | Payload | Описание |
|---|---|---|
| `chat:history` | `Comment[]` | История сообщений. |
| `chat:message` | `Comment` | Новое сообщение. |
| `chat:messageDeleted` | `{ messageId }` | Сообщение удалено. |
| `chat:state` | `object` | Состояние чата. |
| `chat:slowMode` | `{ seconds }` | Slow mode изменён. |
| `chat:success` | `{ code }` | Успех. |
| `chat:error` | `{ code }` | Ошибка. |

Успешная отправка не имеет ack: сервер рассылает `chat:message` всем в комнате.

**Ошибки чата:** `CHAT_NOT_FOUND`, `MESSAGE_NOT_FOUND`, `UNAUTHORIZED`, `INVALID_MESSAGE_EMPTY`, `CHAT_RATE_LIMIT`, `MUTED_IN_THIS_CHAT`, `INVALID_SLOW_MODE_VALUE`.

### 8.4. События live-модерации

**Клиент → сервер:**

| Событие | Payload | Описание |
|---|---|---|
| `live:join` | `{ liveChannelId: ObjectId }` | Войти в комнату стрима. |
| `live:ban` | `{ userId, reason, duration? }` | Забанить. |
| `live:unban` | `{ userId }` | Разбанить. |
| `live:mute` | `{ userId, reason, clear, duration? }` | Замутить. |
| `live:unmute` | `{ userId }` | Размутить. |
| `live:addModerator` | `{ userId }` | Назначить модератором. |
| `live:removeModerator` | `{ userId }` | Снять модератора. |
| `live:mute-state` | `{}` | Запросить состояние мутов. |

**Сервер → клиент:**

| Событие | Payload | Описание |
|---|---|---|
| `live:viewers` | `int` | Количество зрителей. |
| `live:status` | `object` | Статус стрима. |
| `live:started` | `object` | Стрим начался. |
| `live:ended` | `object` | Стрим закончился. |
| `live:ban` | `object` | Пользователь забанен. |
| `live:unban` | `object` | Пользователь разбанен. |
| `live:banned` | `object` | Список забаненных. |
| `live:muted` | `object` | Пользователь замучен. |
| `live:unmuted` | `object` | Пользователь размучен. |
| `live:moderatorAdded` | `object` | Модератор назначен. |
| `live:moderatorRemoved` | `object` | Модератор снят. |
| `live:success` | `{ code }` | Успех. |
| `live:error` | `{ code }` | Ошибка. |

Ошибки модерации приходят как `live:error` (не `chat:error`). Наблюдавшиеся коды: `FORBIDDEN`, `BANNED_FROM_THIS_LIVE`, `CANNOT_PUNISH_OWNER`, `MODERATOR_LIMIT_REACHED`, `MUTED_IN_THIS_CHAT`, `CHAT_RATE_LIMIT`.

---

## 9. Модели данных

Типы ниже описаны в TypeScript-нотации. `ObjectId` — строка из 24 hex-символов; `ISODate` — строка ISO 8601.

```ts
type ObjectId = string;
type ISODate = string;
type Category =
  | "Gaming" | "Music" | "Art" | "Comedy" | "Entertainment" | "News"
  | "Sports" | "Technology" | "Travel" | "Tutorial" | "Animals" | "Science";
type Visibility = "public" | "unlisted" | "private";

interface Paginated<T> {
  items: T[];
  totalPages: number;
  isLastPage: boolean;
}
```

### 9.1. Видео

```ts
interface VideoAuthor {
  _id: ObjectId;
  username: string;
  displayName: string;
  avatarPath: string;
  badgePath: string | null;
  subscribers: number;
}

interface Video {
  _id: ObjectId;
  url: string;                 // URL-ид для /watch?v=
  title: string;
  description: string;
  tags: string;                // "tag1, tag2"
  category: Category | null;
  visibility: Visibility;
  status?: "draft" | "upload_canceled" | "uploading" | "uploaded"
         | "encoding" | "encoding_failed" | "encoding_finished"
         | "published" | "offline" | "live" | "ended"; // только в списках
  deletion?: string;           // только в списках
  filePath: string;            // /uploads/videos/{filePath}
  thumbnailPath: string;       // /uploads/thumbnails/{thumbnailPath}
  agThumbnails: string[];      // авто-обложки
  uploadedBy: VideoAuthor;
  length: number;              // секунды
  formattedLength?: string;    // "m:ss", только в списках
  views: number;
  likes: number;
  dislikes: number;
  isSensitive: boolean;
  isFeatured?: boolean | null;
  createdAt: ISODate;
  updatedAt?: ISODate | null;
}

interface VideoList extends Paginated<Video> {}

interface VideoDraft {
  _id: ObjectId;
  url: string;
  title: string;
  uploadedBy: ObjectId;        // строка, не объект
}

interface VideoStatusResponse {
  success: boolean;
  status: "encoding" | "encoding_finished" | "encoding_failed";
  progress?: number;           // 0–100
}

interface ThumbnailUploaded {
  success: boolean;
  thumbnailPath: string;       // /uploads/thumbnails/{thumbnailPath}
}
```

Особенности:

- В **списках** (`GET /videos`, `GET /users/{u}/videos`, поиск) есть `status`, `deletion` и `formattedLength`.
- В **детальном** `GET /videos/{videoId}` этих полей нет; `uploadedBy` — объект.
- `isFeatured` встречается не всегда.
- `filePath` не содержит полный URL — добавляйте `/uploads/videos/`.

### 9.2. Пользователи и сессия

```ts
interface SessionUser {
  _id: ObjectId;
  username: string;
  roles: string[];             // например ["user"]
  status?: "active" | "banned" | "deleted";
  isVerified?: boolean;
  sessionId?: string;
  hasPremium?: boolean;
  displayName?: string;
  type?: string;               // иногда: поле access-JWT
  iat?: number;                // иногда
  exp?: number;                // иногда
}

interface SessionResponse {
  user: SessionUser;
}

interface UserSummary {
  _id: ObjectId;
  username: string;
  displayName: string;
  avatarPath: string;
  badgePath: string | null;
  subscribers: number;
}

interface User {
  _id: ObjectId;
  username: string;
  displayName: string;
  avatarPath: string;
  bannerPath: string;
  backgroundPath: string;
  badgePath?: string | null;
  subscribers: number;
  bio: string;                 // HTML-экранированный текст
  createdAt: ISODate;
  channel: {
    profileTheme?: {
      presetId?: "halloween" | "terminal" | "golden" | "lava"
               | "mariana_trench" | "bluedrop" | "midnight" | null;
      custom?: { color: string } | null;
    };
    featuredChannels?: {
      groups?: Array<{ name: string; channels: UserSummary[] }>;
    } | null;
    featuredVideo?: Video | null;
    [key: string]: unknown;
  };
}

interface UserSearchResult {
  _id: ObjectId;
  username: string;
  displayName: string;
  avatarPath: string;
  bannerPath: string;
  backgroundPath: string;
  badgePath: string | null;
  bio: string;
  subscribers: number;
  createdAt: ISODate;
  score: number;               // релевантность, только в поиске
}

interface UserSearchResultList extends Paginated<UserSearchResult> {}
```

> `bio` может содержать HTML-сущности (`&#x2F;` и т. п.). Декодируйте их для отображения.

### 9.3. Комментарии

```ts
interface Comment {
  _id: ObjectId;
  user: UserSummary;
  ref: ObjectId;               // ObjectId видео/поста
  refType: "Video" | "CommunityPost";
  text: string;
  attachments: Array<object | null>;
  likes: number;
  dislikes: number;
  replies: number;             // количество ответов
  parent: ObjectId | null;     // родительский комментарий
  isPinned: boolean;
  isEdited: boolean;
  createdAt: ISODate;
  updatedAt: ISODate;
  __v: number;                 // версия документа MongoDB
}

interface CommentList extends Paginated<Comment> {}
```

### 9.4. Плейлисты

```ts
interface PlaylistItem {
  _id: ObjectId;
  playlist: ObjectId;
  video: Video;
  order: number;
  createdAt: ISODate;
  updatedAt: ISODate;
  __v: number;
}

interface Playlist {
  _id: ObjectId;
  owner: UserSummary | ObjectId; // объект в GET /playlists/{id}; строка в списке
  name: string;
  description: string;
  thumbnailPath: string | null;
  visibility: Visibility;
  stats: object;
  items: PlaylistItem[];
  createdAt: ISODate;
  updatedAt: ISODate;
  __v: number;
}

interface PlaylistList extends Paginated<Playlist> {}
```

### 9.5. Сообщество

```ts
interface PollOption {
  _id: ObjectId;
  text: string;
  votes: number;
}

interface Poll {
  _id: ObjectId;
  question: string;
  options: PollOption[];
  multipleChoice: boolean;
  expiresAt: ISODate | null;
}

interface CommunityPost {
  _id: ObjectId;
  user: UserSummary;
  content: string;
  attachments: Array<object | null>;
  likes: number;
  dislikes: number;
  isEdited: boolean;
  poll: Poll | null;
  createdAt: ISODate;
  updatedAt?: ISODate | null;
  __v?: number;
}

interface CommunityPostList extends Paginated<CommunityPost> {}
```

### 9.6. Подписки и уведомления

```ts
interface Subscriber {
  _id: ObjectId;
  subscriber: UserSummary;
  channel: ObjectId;
  notificationsEnabled: boolean;
  createdAt: ISODate;
  updatedAt: ISODate;
  __v: number;
}

interface SubscriberList extends Paginated<Subscriber> {}

interface Notification {
  _id: ObjectId;
  type: "NEW_COMMENT" | "NEW_REPLY" | "MENTION" | "REPORT_RESOLVED"
      | "NEW_POST" | "SYSTEM_MESSAGE";
  isRead: boolean;
  refType: string | null;
  ref: Video | null;
  data: object;
  caller?: UserSummary;
  message: string | null;
  createdAt: ISODate;
}

interface NotificationList extends Paginated<Notification> {}
```

### 9.7. Live

```ts
interface Moderator {
  _id?: ObjectId;
  username: string;
  displayName: string;
}

interface ModeratorList {
  moderators: Moderator[];
}

interface Punishment {
  _id: ObjectId;
  user: UserSummary;
  type: "ban" | "mute";
  reason: string | null;
  createdAt: ISODate;
  expiresAt: ISODate | null;
}

interface PunishmentList extends Paginated<Punishment> {}

interface LiveChannel {
  _id: ObjectId;
  user: ObjectId;              // ObjectId автора строкой
  title: string;
  description: string;
  visibility: Visibility;
  category: Category;
  tags: string;
  thumbnailPath: string | null;
  status: "live" | "ended" | "offline";
  startedAt: ISODate;
  isLive: boolean;
  isLoadedLive: boolean;       // поток уже отдаётся медиасервером
  viewerCount: number;
  createdAt: ISODate;
}
```

### 9.8. Жалобы и служебные модели

```ts
interface Report {
  _id: ObjectId;
  ref: ObjectId;
  refType: string;             // Video | CommunityPost | LiveChannel
  reason: "spam" | "nudity" | "hate" | "harassment" | "copyright" | "other";
  info: string | null;
  resolved: boolean;
  createdAt: ISODate;
}

interface ReportList extends Paginated<Report> {}

interface Success {
  success: boolean;
}

interface ErrorResponse {
  code: string;
  errors: Array<{ msg: string }>;
}

interface CategoriesResponse {
  categories: Category[];
}

interface SubscriptionState {
  subscribed: boolean;
  notificationsEnabled: boolean;
}

interface UserStats {
  videos: number;
  views: number;
  likes: number;
  dislikes: number;
  comments: number;
  subscribers: number;
}
```

---

## 10. Коды ошибок

`code` — стабильный машиночитаемый идентификатор. `errors[].msg` — человекочитаемый текст, который может быть локализован (cookie `i18next`) и изменён без предупреждения.

### 10.1. Основные коды

| Код | HTTP | Значение |
|---|---|---|
| `PAGE_INVALID` | 400 | Неверный или отсутствующий `page`. |
| `LIMIT_INVALID` | 400 | Неверный или отсутствующий `limit` (или > 25). |
| `PAGE_NOT_FOUND` | 404 | Страница за пределами выборки (в том числе пустая выборка). |
| `RESOURCE_NOT_FOUND` | 404 | Ресурс не существует. |
| `ID_INVALID` | 400 | Неверный ObjectId. |
| `URL_INVALID` | 400 | Неверный формат URL-ида видео. |
| `URL_REQUIRED` | 400 | Не передан URL-ид. |
| `FIELD_REQUIRED` | 400 | Не передано обязательное поле. |
| `INVALID_JSON` | 400 | Тело не является корректным JSON. |
| `BOOLEAN_INVALID` | 400 | Ожидался boolean. |
| `NUMBER_INVALID` | 400 | Ожидалось число. |
| `SORT_REQUIRED` | 400 | Не передан `sort`. |
| `TYPE_REQUIRED` | 400 | Не передан `type`. |
| `TYPE_INVALID` | 400 | Неверный `type`. |
| `UNAUTHORIZED` | 401 | Нет действующей сессии. |
| `FORBIDDEN` | 403 | Недостаточно прав. |
| `TOO_MANY_REQUESTS` | 429 | Превышен лимит запросов. |
| `INTERNAL_SERVER_ERROR` | 500 | Внутренняя ошибка. |
| `NO_CHANGES_MADE` | 200 | Изменений нет (особый случай). |

> Для `GET /search` сервер возвращает коды с префиксом: `search.QUERY_REQUIRED`, `search.TYPE_REQUIRED`, `search.TYPE_INVALID`. В перечислении спецификации эти значения записаны без префикса; ориентируйтесь на фактический ответ.

### 10.2. Аутентификация и профиль

| Код | Значение |
|---|---|
| `USERNAME_REQUIRED` | Не передан логин. |
| `USERNAME_INVALID` | Неверный формат логина. |
| `USERNAME_TAKEN` | Логин занят. |
| `PASSWORD_REQUIRED` | Не передан пароль. |
| `PASSWORD_INVALID` | Пароль не соответствует требованиям. |
| `WRONG_PASSWORD` | Неверный пароль. |
| `PASSWORDS_NOT_MATCH` | Пароли не совпадают. |
| `EMAIL_REQUIRED` | Не передан e-mail. |
| `EMAIL_INVALID` | Неверный e-mail. |
| `EMAIL_TAKEN` | E-mail занят. |
| `EMAIL_NOT_VERIFIED` | E-mail не подтверждён. |
| `EMAIL_ALREADY_VERIFIED` | E-mail уже подтверждён. |
| `TOKEN_INVALID` | Неверный/истёкший токен подтверждения. |
| `CAPTCHA_REQUIRED` | Требуется reCAPTCHA. |
| `CAPTCHA_FAILED` | reCAPTCHA не пройдена. |
| `TERMS_REQUIRED` | Не подтверждены условия. |
| `TERMS_INVALID` | Неверное подтверждение условий. |
| `DISPLAY_NAME_REQUIRED` | Не передано отображаемое имя. |
| `DISPLAY_NAME_INVALID` | Неверный формат имени. |
| `DISPLAY_NAME_INVALID_LENGTH` | Неверная длина имени. |
| `DISPLAY_NAME_CHANGE_NOT_ALLOWED` | Смена имени на кулдауне. |
| `BIO_REQUIRED` | Не передано bio. |
| `BIO_TOO_LONG` | Bio слишком длинное. |
| `PROFILE_THEME_PRESET_INVALID` | Неверный пресет темы. |
| `PROFILE_THEME_CUSTOM_INVALID` | Неверная кастомная тема. |
| `PROFILE_THEME_PREMIUM_REQUIRED` | Тема доступна только с Premium. |

### 10.3. Видео

| Код | Значение |
|---|---|
| `TITLE_INVALID` | Заголовок не 3–50 символов. |
| `DESCRIPTION_TOO_LONG` | Описание слишком длинное. |
| `TAGS_TOO_LONG` | Теги слишком длинные. |
| `CATEGORY_INVALID` | Неверная категория. |
| `VISIBILITY_REQUIRED` | Не передана видимость. |
| `VISIBILITY_INVALID` | Неверная видимость. |
| `VIDEO_FILE_REQUIRED` | Не передан файл `video`. |
| `INVALID_FILE_TYPE` | Неверный тип файла. |
| `FILE_TOO_LARGE` | Файл больше 200 МБ. |
| `VIDEO_ALREADY_UPLOADED` | В черновик уже загружен файл. |
| `TOO_MANY_VIDEOS` | Превышен лимит 10 видео за 24 часа. |
| `VIDEO_NOT_FOUND` | Видео не найдено. |
| `VIDEO_STATUS_MISSING` | Нет статуса обработки. |
| `VIDEO_NOT_PUBLISHED` | Видео не опубликовано. |
| `AG_THUMBNAILS_NOT_AVAILABLE` | Авто-обложки недоступны. |
| `AG_INDEX_INVALID` | Неверный индекс авто-обложки. |
| `THUMBNAIL_FILE_REQUIRED` | Не передана обложка. |

### 10.4. Комментарии, реакции, плейлисты

| Код | Значение |
|---|---|
| `TEXT_REQUIRED` | Не передан текст. |
| `TEXT_TOO_LONG` | Текст слишком длинный. |
| `TEXT_LENGTH_INVALID` | Неверная длина текста. |
| `COMMENT_ID_REQUIRED` | Не передан `commentId`. |
| `COMMENT_NOT_FOUND` | Комментарий не найден. |
| `CANT_PIN_REPLY` | Нельзя закрепить ответ. |
| `PINNED_INVALID` | Неверное состояние закрепления. |
| `RATING_NOT_UPDATED` | Оценки не было. |
| `SELF_SUBSCRIPTION_NOT_ALLOWED` | Нельзя подписаться на себя. |
| `SELF_UNSUBSCRIPTION_NOT_ALLOWED` | Нельзя отписаться от себя. |
| `REACHED_PLAYLIST_LIMIT` | Лимит плейлистов. |
| `REACHED_VIDEO_LIMIT` | Лимит видео в плейлисте. |
| `VIDEO_ALREADY_IN_PLAYLIST` | Видео уже в плейлисте. |
| `PLAYLIST_NOT_FOUND` | Плейлист не найден. |
| `PLAYLIST_ID_REQUIRED` | Не передан `playlistId`. |

### 10.5. Сообщество, live, жалобы

| Код | Значение |
|---|---|
| `INVALID_POLL` | Неверный опрос. |
| `POLL_EXPIRED` | Опрос завершён. |
| `INVALID_VOTE_OPTIONS` | Неверные варианты голоса. |
| `MULTIPLE_CHOICE_NOT_SUPPORTED` | Множественный выбор не поддерживается. |
| `LIVE_NOT_FOUND` | Стрим не найден. |
| `CHAT_NOT_FOUND` | Чат не найден (или не был `chat:join`). |
| `CHAT_RATE_LIMIT` | Слишком частые сообщения. |
| `MUTED_IN_THIS_CHAT` | Пользователь замучен. |
| `BANNED_FROM_THIS_LIVE` | Пользователь забанен на стриме. |
| `CANNOT_PUNISH_OWNER` | Нельзя наказать владельца. |
| `MODERATOR_LIMIT_REACHED` | Лимит модераторов. |
| `REF_REQUIRED` | Не передан `ref`. |
| `REF_TYPE_REQUIRED` | Не передан `refType`. |
| `REF_TYPE_INVALID` | Неверный `refType`. |
| `REASON_REQUIRED` | Не передана причина. |
| `REASON_INVALID` | Неверная причина. |
| `INFO_REQUIRED` | Не передана информация. |
| `INFO_TOO_LONG` | Информация слишком длинная. |
| `REPORT_NOT_RESOLVED` | Предыдущая жалоба ещё не рассмотрена. |
| `REPORT_ALREADY_RESOLVED` | Жалоба уже рассмотрена. |
| `REPORT_ID_REQUIRED` | Не передан id жалобы. |

### 10.6. Полный список кодов

Все 125 значений из перечисления `Error.code` (включая коды, используемые Socket.IO и успешные операции):

<details>
<summary>Показать полный список</summary>

```text
AG_INDEX_INVALID
AG_THUMBNAILS_NOT_AVAILABLE
BADGE_NOT_FOUND
BANNED_FROM_THIS_LIVE
BANNED_USER_SUCCESSFULLY
BIO_REQUIRED
BIO_TOO_LONG
BOOLEAN_INVALID
CANNOT_PUNISH_OWNER
CANT_PIN_REPLY
CAPTCHA_FAILED
CAPTCHA_REQUIRED
CATEGORY_INVALID
CHAT_NOT_FOUND
CHAT_RATE_LIMIT
COMMENT_ID_REQUIRED
COMMENT_NOT_FOUND
DESCRIPTION_INVALID
DESCRIPTION_TOO_LONG
DISPLAY_NAME_CHANGE_NOT_ALLOWED
DISPLAY_NAME_INVALID
DISPLAY_NAME_INVALID_LENGTH
DISPLAY_NAME_REQUIRED
DRAFT_ID_REQUIRED
DUPLICATE_GROUP_NAME
EMAIL_ALREADY_VERIFIED
EMAIL_INVALID
EMAIL_NOT_VERIFIED
EMAIL_REQUIRED
EMAIL_TAKEN
FIELD_REQUIRED
FILE_TOO_LARGE
FORBIDDEN
ID_INVALID
INFO_REQUIRED
INFO_TOO_LONG
INTERNAL_SERVER_ERROR
INVALID_FILE_TYPE
INVALID_JSON
INVALID_MESSAGE_EMPTY
INVALID_MESSAGE_TOO_LONG
INVALID_POLL
INVALID_SLOW_MODE_VALUE
INVALID_TIME
INVALID_VOTE_OPTIONS
LIMIT_INVALID
LIVE_NOT_FOUND
MESSAGE_DELETED_SUCCESSFULLY
MESSAGE_NOT_FOUND
MODERATOR_ADDED_SUCCESSFULLY
MODERATOR_LIMIT_REACHED
MODERATOR_REMOVED_SUCCESSFULLY
MULTIPLE_CHOICE_NOT_SUPPORTED
MUTED_IN_THIS_CHAT
MUTED_USER_SUCCESSFULLY
NOTE_REQUIRED
NOTE_TOO_LONG
NO_CHANGES_MADE
NUMBER_INVALID
PAGE_INVALID
PAGE_NOT_FOUND
PASSWORDS_NOT_MATCH
PASSWORD_INVALID
PASSWORD_REQUIRED
PINNED_INVALID
PLAYLIST_ID_REQUIRED
PLAYLIST_NOT_FOUND
POLL_EXPIRED
PROFILE_THEME_CUSTOM_INVALID
PROFILE_THEME_PREMIUM_REQUIRED
PROFILE_THEME_PRESET_INVALID
QUERY_REQUIRED
RATING_NOT_UPDATED
REACHED_ACCOUNT_LIMIT
REACHED_PLAYLIST_LIMIT
REACHED_VIDEO_LIMIT
REASON_INVALID
REASON_REQUIRED
REF_REQUIRED
REF_TYPE_INVALID
REF_TYPE_REQUIRED
REPORT_ALREADY_RESOLVED
REPORT_ID_REQUIRED
REPORT_NOT_RESOLVED
RESOURCE_NOT_FOUND
SELF_SUBSCRIPTION_NOT_ALLOWED
SELF_UNSUBSCRIPTION_NOT_ALLOWED
SET_SLOW_MODE_SUCCESSFULLY
SLOW_MODE_ACTIVE
SORT_REQUIRED
TAGS_TOO_LONG
TERMS_INVALID
TERMS_REQUIRED
TEXT_LENGTH_INVALID
TEXT_REQUIRED
TEXT_TOO_LONG
THUMBNAIL_FILE_REQUIRED
TITLE_INVALID
TOKEN_INVALID
TOO_MANY_REQUESTS
TOO_MANY_VIDEOS
TYPE_INVALID
TYPE_REQUIRED
UNAUTHORIZED
UNBANNED_USER_SUCCESSFULLY
UNMUTED_USER_SUCCESSFULLY
URL_INVALID
URL_REQUIRED
USERNAME_INVALID
USERNAME_REQUIRED
USERNAME_TAKEN
USER_ID_REQUIRED
USER_NOT_BANNED
USER_NOT_FOUND
USER_NOT_MUTED
VIDEO_ALREADY_IN_PLAYLIST
VIDEO_ALREADY_UPLOADED
VIDEO_FILE_REQUIRED
VIDEO_ID_REQUIRED
VIDEO_NOT_FOUND
VIDEO_NOT_PUBLISHED
VIDEO_STATUS_MISSING
VISIBILITY_INVALID
VISIBILITY_REQUIRED
WRONG_PASSWORD
```

</details>

---

## 11. Подводные камни

1. **`page` и `limit` обязательны.** Забыли `limit` — `400 LIMIT_INVALID`; забыли `page` — `400 PAGE_INVALID`; забыли оба — оба сообщения. Максимум `limit` — 25.
2. **`ref` — это ObjectId, а не URL-ид.** Для видео берите `_id` из ответа, а не `url` из ссылки `/watch?v=…`.
3. **`videoId` означает разное.** В `GET/PATCH/DELETE /videos/{videoId}` и `/thumbnail` — URL-ид; в `/upload`, `/status`, `/publish` — ObjectId.
4. **Вход возвращает HTML.** `POST /auth/login` при успехе отдаёт `200 text/html`; не вызывайте `.json()` на этом ответе. Признак успеха — статус и cookie.
5. **`bio: ""` не очищает bio.** Пустая строка игнорируется. Очистка через API не предусмотрена.
6. **`NO_CHANGES_MADE` приходит с HTTP 200.** Проверяйте поле `code`, а не только HTTP-статус.
7. **Нет подписки — это `404`, а не `false`.** `GET /engagement/{channelId}/subscription` возвращает `404 RESOURCE_NOT_FOUND`, если подписки нет.
8. **`GET /users/{username}/subscribers` не существует.** Список подписчиков — только `GET /engagement/subscribers?channelId=…`.
9. **`PUT /playlists/{id}/videos/{videoId}` зависает.** Метод помечен deprecated; используйте `POST`/`DELETE`.
10. **Создание поста сообщества может быть недоступно.** `POST /community/posts` для обычного аккаунта возвращает `403 FORBIDDEN`.
11. **Пустой список жалоб — это `404 PAGE_NOT_FOUND`.** Для админа/модератора пустая выборка `GET /reports` выглядит как отсутствие страницы.
12. **Счётчик видео в статистике не уменьшается.** `GET /users/{u}/stats` — денормализованное значение, а не `items.length`.
13. **Детальный ответ видео беднее списка.** В `GET /videos/{id}` нет `status`, `deletion`, `formattedLength`. Не рассчитывайте на них.
14. **Поиск использует `query`, а не `q`.** С `q` — `400 QUERY_REQUIRED`. `type` обязателен: `videos` или `users`.
15. **Создание комментария и плейлиста не возвращает `_id`.** Ответ — `{ "success": true }`; id ищите повторным чтением списка.
16. **`page` для уведомлений и `limit` для ответов фактически обязательны.** В спецификации они могут быть помечены как необязательные, но без них приходит `400`.
17. **CORS нет.** Браузерный клиент с другого origin не сможет читать ответы; нужен backend-прокси.
18. **Некоторые «разделы» — это `400 URL_INVALID`.** `/videos/latest`, `/videos/popular`, `/videos/trending`, `/videos/random`, `/videos/search`, `/videos/drafts` не существуют: сегмент `latest` разбирается как `videoId` и не проходит валидацию.
19. **`isLastPage` может быть несогласован с `totalPages`.** На пустых плейлистах наблюдалось `totalPages: 0`, `isLastPage: false`. Проверяйте `items.length`.
20. **Тексты HTML-экранированы.** Декодируйте сущности, но не вставляйте пользовательский текст как HTML.
21. **Лимит на маршрут может сработать раньше глобального.** Особенно при опросе `/status` и в поиске.
22. **`GET /users` и `GET /playlists` — только для админа/модератора.** Свои данные — через `/users/{username}/...`.
23. **`profileTheme` принимает только известные пресеты.** `presetId` — `halloween`, `terminal`, `golden`, `lava`, `mariana_trench`, `bluedrop`, `midnight`; кастомная тема — `custom: { color }`. Часть пресетов может требовать Premium (`PROFILE_THEME_PREMIUM_REQUIRED`).
24. **`DELETE /users/{username}` не проверялся.** Маршрут существует, но может иметь дополнительные ограничения.

---

## 12. Примеры клиентов

### 12.1. curl: публичные запросы

```bash
BASE="https://retrotube.net/api/v1"

# Каталог
curl -s "$BASE/videos?page=1&limit=10&sort=newest"

# Категории
curl -s "$BASE/videos/categories"

# Рекомендуемые (массив)
curl -s "$BASE/videos/featured"

# Видео по URL-иду
curl -s "$BASE/videos/VQcp3UEdiu7dz"

# Профиль
curl -s "$BASE/users/delovoy24"

# Поиск видео
curl -s "$BASE/search?query=api&type=videos&page=1&limit=10"

# Поиск пользователей
curl -s "$BASE/search?query=delovoy24&type=users&page=1&limit=10"
```

### 12.2. curl: вход и защищённые запросы

```bash
BASE="https://retrotube.net/api/v1"

# Вход: сохранить cookie
curl -s -c cookies.txt -X POST "$BASE/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "username=yourlogin" \
  --data-urlencode "password=YourPassword1"

# Проверка сессии
curl -s -b cookies.txt "$BASE/auth/session"

# Свои видео (включая скрытые)
curl -s -b cookies.txt "$BASE/users/yourlogin/videos?page=1&limit=10&hidden=true"

# Уведомления
curl -s -b cookies.txt "$BASE/users/yourlogin/notifications?page=1"
curl -s -b cookies.txt "$BASE/users/yourlogin/notifications/unread"

# Отметить прочитанными
curl -s -b cookies.txt -X POST \
  "$BASE/users/yourlogin/notifications?read=true"

# Выход
curl -s -b cookies.txt -X POST "$BASE/auth/logout"
```

### 12.3. curl: комментарии, реакции, жалобы

```bash
BASE="https://retrotube.net/api/v1"
VIDEO_ID="6ae02ad490aa27b3ebcd3cd1"   # ObjectId, не URL-ид

# Комментарий (multipart)
curl -s -b cookies.txt -X POST \
  "$BASE/comments?ref=$VIDEO_ID&refType=Video" \
  -F "text=Отличное видео!"

# Ответ на комментарий (id комментария — ObjectId)
curl -s -b cookies.txt -X POST \
  "$BASE/comments/$COMMENT_ID/replies" \
  -F "text=Согласен!"

# Лайк
curl -s -b cookies.txt -X POST \
  "$BASE/engagement/like?ref=$VIDEO_ID&refType=Video"

# Снять оценку
curl -s -b cookies.txt -X DELETE \
  "$BASE/engagement/rating?ref=$VIDEO_ID&refType=Video"

# Жалоба (публичный маршрут)
curl -s -X POST "$BASE/reports" \
  -H "Content-Type: application/json" \
  -d "{\"ref\":\"$VIDEO_ID\",\"refType\":\"Video\",\"reason\":\"spam\",\"info\":\"Спам\"}"
```

### 12.4. Python: сессия, пагинация, загрузка

```python
import requests

BASE = "https://retrotube.net/api/v1"

s = requests.Session()
s.headers["Accept"] = "application/json"

# Вход. Успех — 200 text/html, поэтому не вызываем .json()
r = s.post(
    f"{BASE}/auth/login",
    data={"username": "yourlogin", "password": "YourPassword1"},
    timeout=30,
)
r.raise_for_status()

me = s.get(f"{BASE}/auth/session", timeout=30).json()["user"]
print("вошли как", me["username"])

# Пагинация каталога
page = 1
while True:
    data = s.get(
        f"{BASE}/videos",
        params={"page": page, "limit": 25, "sort": "newest"},
        timeout=30,
    ).json()
    for video in data["items"]:
        print(video["url"], video["title"])
    if data["isLastPage"]:
        break
    page += 1

# Загрузка видео
draft = s.post(
    f"{BASE}/videos",
    json={"title": "Загрузка из Python"},
    timeout=30,
).json()
video_id = draft["_id"]          # ObjectId
video_url_id = draft["url"]      # URL-ид

with open("movie.mp4", "rb") as f:
    r = s.post(
        f"{BASE}/videos/{video_id}/upload",
        files={"video": ("movie.mp4", f, "video/mp4")},
        timeout=600,
    )
r.raise_for_status()

# Ожидание кодирования
import time
while True:
    status = s.get(f"{BASE}/videos/{video_id}/status", timeout=30).json()
    print(status)
    if status["status"] == "encoding_finished":
        break
    if status["status"] == "encoding_failed":
        raise RuntimeError("кодирование не удалось")
    time.sleep(5)

# Публикация
s.post(f"{BASE}/videos/{video_id}/publish", timeout=30).raise_for_status()

# Метаданные (URL-ид)
s.patch(
    f"{BASE}/videos/{video_url_id}",
    json={
        "description": "Описание",
        "tags": "api, python",
        "category": "Technology",
        "visibility": "public",
    },
    timeout=30,
).raise_for_status()

# Комментарий: multipart, не urlencoded
s.post(
    f"{BASE}/comments",
    params={"ref": video_id, "refType": "Video"},
    files={"text": (None, "Первый комментарий")},
    timeout=30,
).raise_for_status()
```

### 12.5. Python: обработка ошибок

```python
def api_error(response: requests.Response) -> str:
    try:
        data = response.json()
    except ValueError:
        return f"HTTP {response.status_code}: {response.text[:200]}"
    code = data.get("code", "UNKNOWN")
    msgs = "; ".join(e.get("msg", "") for e in data.get("errors", []))
    return f"HTTP {response.status_code} {code}: {msgs}"

r = s.get(f"{BASE}/videos", params={"page": 1}, timeout=30)
if not r.ok:
    print(api_error(r))
```

### 12.6. JavaScript: same-origin

```js
const BASE = "/api/v1";

async function api(path, options = {}) {
  const res = await fetch(BASE + path, {
    credentials: "include",          // cookie сессии
    headers: { Accept: "application/json", ...(options.headers || {}) },
    ...options,
  });
  const data = await res.json();
  if (!res.ok) {
    throw new Error(`${res.status} ${data.code || "UNKNOWN"}`);
  }
  return data;
}

// Каталог
const list = await api("/videos?page=1&limit=10&sort=newest");
console.log(list.items);

// Лайк (ref — ObjectId)
await api(`/engagement/like?ref=${videoId}&refType=Video`, { method: "POST" });

// Комментарий (multipart)
const form = new FormData();
form.append("text", "Отличное видео!");
await api(`/comments?ref=${videoId}&refType=Video`, {
  method: "POST",
  body: form,
});
```

> Этот код работает только на `https://retrotube.net` (same-origin). Для внешнего сайта нужен прокси на вашем backend.

### 12.7. Node.js: серверный клиент

```js
const BASE = "https://retrotube.net/api/v1";
let cookies = "";

async function request(path, options = {}) {
  const res = await fetch(BASE + path, {
    ...options,
    headers: { Accept: "application/json", Cookie: cookies, ...(options.headers || {}) },
  });
  const setCookie = res.headers.getSetCookie?.() || [];
  if (setCookie.length) {
    cookies = setCookie.map((c) => c.split(";")[0]).join("; ");
  }
  if (!res.ok) {
    const data = await res.json().catch(() => ({}));
    throw new Error(`${res.status} ${data.code || "UNKNOWN"}`);
  }
  // /auth/login отвечает HTML, остальные методы — JSON
  const type = res.headers.get("content-type") || "";
  return type.includes("application/json") ? res.json() : res.text();
}

// Вход form-urlencoded
await request("/auth/login", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: new URLSearchParams({ username: "yourlogin", password: "YourPassword1" }),
});

const session = await request("/auth/session");
console.log(session.user.username);
```

---

## 13. Источники и методика

### 13.1. Что проверялось

Документация составлена 27 октября 2026 года. Живыми запросами проверены, в частности:

- `GET /videos` с обязательными `page`/`limit`, лимитом 25 и фильтрами;
- `GET /videos/categories`, `GET /videos/featured`, `GET /videos/{urlId}`;
- `GET /users/{username}`, `GET /users/{username}/videos`, `GET /users/{username}/playlists`;
- `GET /comments` и `GET /comments/pinned` с `ref`/`refType`;
- `GET /search` для видео и пользователей;
- формат ошибок, локализация через cookie `i18next`, два вида `404`;
- заголовки `X-RateLimit-*`, ответ `429`;
- отсутствие CORS-заголовков в `OPTIONS`;
- валидация URL-идов и пагинации;
- публичные данные профилей, видео и комментариев.

Защищённые маршруты (загрузка, публикация, комментарии, лайки, подписки, плейлисты, уведомления, live-модерация) описаны по открытой OpenAPI-спецификации сообщества и коду клиента. Часть из них помечена в спецификации как `live` (проверена под тестовым аккаунтом), часть — как `derived` (подтверждено только наличие маршрута).

### 13.2. OpenAPI-спецификация

- Репозиторий: <https://github.com/RootGuardian10/retrotube-cli>
- Файл: <https://github.com/RootGuardian10/retrotube-cli/blob/main/retrotube_openapi.json>
- Версия спецификации: OpenAPI 3.1.0, `RetroTube API 1.0.0`.
- По данным файла, спецификация собрана 20 октября 2026 года, защищённые маршруты проверены 23 октября 2026 года.
- В спецификации 49 путей и 69 операций; для каждой операции указан `x-verified` (`live` или `derived`).

### 13.3. Ограничения

- Это **неофициальная** документация. Официального публичного API-справочника у retrotube.net нет.
- Сайт в статусе **Beta**: поля, коды и поведение могут измениться.
- Некоторые детали (точные ограничения типов файлов, поведение отдельных Socket.IO-событий) в спецификации не описаны.
- Деструктивные операции (`DELETE /users/{username}`, удаление видео/плейлистов) не проверялись на живом аккаунте.
- Ответы с реальными пользовательскими данными в примерах сокращены и обезличены.

### 13.4. Использование

- Соблюдайте условия использования сайта и лимиты запросов.
- Не используйте API для массового сбора персональных данных, спама и обхода ограничений.
- Для продакшена закладывайте обработку `429`, таймауты и повторные попытки с экспоненциальной задержкой.
- Не полагайтесь на текст ошибок: используйте `code`.

---

*Конец документа.*

