## Интерфейс
 
### Список претензионных дел

<img width="1892" height="900" alt="image" src="https://github.com/user-attachments/assets/ab24afdf-17a8-4524-a52f-b4cffd15ee7f" />

### История изменения статусов

<img width="965" height="889" alt="image" src="https://github.com/user-attachments/assets/ab3c4186-19ad-4131-bb79-1db01249c81f" />

## Авторизация
 
Все методы требуют JWT-токен в заголовке:
 
```
Authorization: Bearer <JWT>
```
 
Отсутствующий или недействительный токен → `401 Unauthorized`.
 
---
 
## API спецификация 
 
### GET /claims/statuses — справочник статусов
 
Возвращает все статусы претензионных дел. Используется для заполнения выпадающих списков «Все статусы» (фильтр) и «Изменить статус» (в строке таблицы).
 
**Запрос**
 
```
GET https://api.technoimport.example/api/v1/claims/statuses
```
 
| Параметры | |
|---|---|
| Path parameters | — |
| Query parameters | — |
| Request Body | — |
 
**Headers**
 
| Header | Обязательный | Значение |
|---|---|---|
| `Authorization` | Да | `Bearer <JWT>` |
 
**Response Body — 200 OK**
 
```json
{
  "items": [
    {
      "status_name": "Создан",
      "status_code": "CREATED"
    },
    {
      "status_name": "В работе",
      "status_code": "IN_PROGRESS"
    },
    {
      "status_name": "Направлено поставщику",
      "status_code": "SENT"
    },
    {
      "status_name": "Ответ получен",
      "status_code": "RESPONSE_RECEIVED"
    },
    {
      "status_name": "На согласовании",
      "status_code": "UNDER_APPROVAL"
    },
    {
      "status_name": "Согласовано",
      "status_code": "AGREED"
    },
    {
      "status_name": "Спор с поставщиком",
      "status_code": "DISPUTE"
    },
    {
      "status_name": "Закрыта",
      "status_code": "CLOSED"
    }
  ]
}
```
 
**Коды ответов**
 
| Код | Описание |
|---|---|
| `200 OK` | Список статусов успешно получен |
| `401 Unauthorized` | Отсутствует или недействителен токен авторизации |
| `500 Internal Server Error` | Внутренняя ошибка сервера |
 
---
 
### GET /claims — список претензионных дел
 
Возвращает сводную статистику, страницу списка претензионных дел и данные пагинации. Используется для загрузки экрана «Претензионные дела» и при каждом изменении поиска, фильтра, сортировки или страницы.
 
**Запрос**
 
```
GET https://api.technoimport.example/api/v1/claims?status=allStatuses&timeSort=DESC&page=1&limit=10
```
 
**Query parameters**
 
| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `search` | string | Нет | Поиск по номеру дела (`claim_id`) или названию поставщика. Минимальной длины нет |
| `status` | string | Нет | Фильтр по одному статусу: `status_code` из справочника или `allStatuses` — все статусы. Несколько статусов одновременно не поддерживаются |
| `timeSort` | string | Нет | Сортировка по времени последнего изменения: `DESC` — сначала недавно изменённые, `ASC` — сначала давно изменённые |
| `page` | integer | Нет | Номер страницы, начиная с `1` |
| `limit` | integer | Нет | Количество записей на странице, максимум `100` |
 
**Headers**
 
| Header | Обязательный | Значение |
|---|---|---|
| `Authorization` | Да | `Bearer <JWT>` |
 
Request Body — отсутствует.
 
**Response Body — 200 OK**
 
```json
{
  "summary": {
    "all_claims": 248,
    "in_progress": 64,
    "under_approval": 18,
    "dispute": 7
  },
  "items": [
    {
      "claim_id": "CLM-2026-123456",
      "supplier_name": "ООО Вектор Снаб",
      "status_name": {
        "name": "В работе",
        "code": "IN_PROGRESS"
      },
      "last_change": {
        "name": "Иван Иванов",
        "user_avatar_url": "https://api.technoimport.example/avatars/usr-105.jpg",
        "changed_at": "2026-10-14T10:24:00+03:00"
      }
    }
  ],
  "pagination": {
    "current_page": 1,
    "items_on_page": 10,
    "total_items": 248,
    "total_pages": 25
  }
}
```
 
> Массив `items` в примере сокращён до одной записи.
 
Бизнес-правила:
 
- `summary` считается по всем делам и не зависит от `search` и `status`.
- `pagination` отражает результат с учётом `search` и `status`.

**Коды ответов**
 
| Код | Описание |
|---|---|
| `200 OK` | Список претензионных дел успешно получен |
| `400 Bad Request` | Некорректные параметры запроса |
| `401 Unauthorized` | Отсутствует или недействителен токен авторизации |
| `500 Internal Server Error` | Внутренняя ошибка сервера |
 
---
 
### PATCH /claims/{claim_id}/status — изменение статуса
 
Изменяет статус претензионного дела. Вызывается при выборе нового значения в колонке «Изменить статус». Каждое успешное изменение записывается в историю.
 
**Запрос**
 
```
PATCH https://api.technoimport.example/api/v1/claims/CLM-2026-123457/status
```
 
**Path parameters**
 
| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `claim_id` | string | Да | Номер претензионного дела, например `CLM-2026-123457` |
 
Query parameters — отсутствуют.
 
**Headers**
 
| Header | Обязательный | Значение |
|---|---|---|
| `Authorization` | Да | `Bearer <JWT>` |
| `Content-Type` | Да | `application/json` |
 
**Request Body**
 
```json
{
  "status_code": "UNDER_APPROVAL"
}
```
 
| Поле | Тип | Обязательный | Описание |
|---|---|---|---|
| `status_code` | string | Да | Код нового статуса из справочника |
 
**Response Body — 200 OK**
 
```json
{
  "claim_id": "CLM-2026-123457",
  "status": {
    "status_name": "На согласовании",
    "status_code": "UNDER_APPROVAL",
    "changed_at": "2026-10-14T09:10:00+03:00"
  }
}
```

**Коды ответов**
 
| Код | Описание |
|---|---|
| `200 OK` | Статус успешно изменён, либо новый статус совпадает с текущим |
| `400 Bad Request` | Некорректные параметры запроса или недопустимое значение `status_code` |
| `401 Unauthorized` | Отсутствует или недействителен токен авторизации |
| `403 Forbidden` | Недостаточно прав для выполнения перехода |
| `404 Not Found` | Претензионное дело не найдено |
| `409 Conflict` | Переход отсутствует в бизнес логике |
| `422 Unprocessable Entity` | Переход допустим, но условие перехода не выполнено |
| `500 Internal Server Error` | Внутренняя ошибка сервера |
 
#### Идемпотентность
 
Если `status_code` в запросе совпадает с текущим статусом дела (например, при повторном нажатии), сервер:
 
- возвращает `200 OK` с текущим состоянием дела;
- не добавляет запись в историю;
- не изменяет `changed_at`.
---
 
### GET /claims/{claim_id}/history — история изменений статусов
 
Возвращает текущий статус дела и список изменений статусов: предыдущий и новый статус, сотрудник и время изменения. Вызывается при нажатии «Смотреть» в строке таблицы.
 
**Запрос**
 
```
GET https://api.technoimport.example/api/v1/claims/CLM-2026-123457/history
```
 
**Path parameters**
 
| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `claim_id` | string | Да | Номер претензионного дела |
 
Query parameters — отсутствуют. Request Body — отсутствует.
 
**Headers**
 
| Header | Обязательный | Значение |
|---|---|---|
| `Authorization` | Да | `Bearer <JWT>` |
 
**Response Body — 200 OK**
 
```json
{
  "claim_id": "CLM-2026-123457",
  "status": {
    "status_name": "На согласовании",
    "status_code": "UNDER_APPROVAL",
    "changed_at": "2026-10-14T09:10:00+03:00"
  },
  "last_changes": [
    {
      "previous_status_code": "RESPONSE_RECEIVED",
      "previous_status_name": "Ответ получен",
      "new_status_code": "UNDER_APPROVAL",
      "new_status_name": "На согласовании",
      "user_id": "USR-214",
      "user_name": "Елена Смирнова",
      "user_avatar_url": "https://api.technoimport.example/avatars/usr-214.jpg",
      "changed_at": "2026-10-14T09:10:00+03:00"
    },
    {
      "previous_status_code": "SENT",
      "previous_status_name": "Направлено поставщику",
      "new_status_code": "RESPONSE_RECEIVED",
      "new_status_name": "Ответ получен",
      "user_id": "USR-105",
      "user_name": "Иван Иванов",
      "user_avatar_url": "https://api.technoimport.example/avatars/usr-105.jpg",
      "changed_at": "2026-10-13T16:45:00+03:00"
    },
    {
      "previous_status_code": "IN_PROGRESS",
      "previous_status_name": "В работе",
      "new_status_code": "SENT",
      "new_status_name": "Направлено поставщику",
      "user_id": "USR-105",
      "user_name": "Иван Иванов",
      "user_avatar_url": "https://api.technoimport.example/avatars/usr-105.jpg",
      "changed_at": "2026-10-10T11:15:00+03:00"
    },
    {
      "previous_status_code": "CREATED",
      "previous_status_name": "Создан",
      "new_status_code": "IN_PROGRESS",
      "new_status_name": "В работе",
      "user_id": "USR-105",
      "user_name": "Иван Иванов",
      "user_avatar_url": "https://api.technoimport.example/avatars/usr-105.jpg",
      "changed_at": "2026-10-09T14:20:00+03:00"
    }
  ]
}
```

Бизнес-правила:
 
- события отсортированы от новых к старым;
- дело создаётся системой сразу в статусе `CREATED`; это не изменение статуса, поэтому первое событие истории — переход `CREATED` → `IN_PROGRESS`;
- повторная отправка текущего статуса в историю не попадает.

**Коды ответов**
 
| Код | Описание |
|---|---|
| `200 OK` | История изменений статусов успешно получена |
| `400 Bad Request` | Некорректные параметры запроса |
| `401 Unauthorized` | Отсутствует или недействителен токен авторизации |
| `404 Not Found` | Претензионное дело не найдено |
| `500 Internal Server Error` | Внутренняя ошибка сервера |
 
---
