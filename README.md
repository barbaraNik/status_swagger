# Претензионные дела — смена статусов API

OpenAPI-спецификация модуля «Претензионные дела» компании ООО «ТехноИмпорт»: справочник статусов и изменение статуса претензионного дела по матрице переходов.

| Файл | Содержание |
|---|---|
| [`claims-openapi.yaml`](./claims-openapi.yaml) | Спецификация OpenAPI 3.1 |
| [`status-names.md`](./status-names.md) | Статусы, итоги закрытия и полная матрица переходов |
| [`prototype-description.md`](./prototype-description.md) | Описание интерфейса и API-контрактов |

---

## Просмотр спецификации

Откройте [editor.swagger.io](https://editor.swagger.io) → **File → Import file** → выберите `claims-openapi.yaml`.

---

## Общие параметры

| Параметр | Значение |
|---|---|
| Базовый URL | `https://api.technoimport.example/api/v1` |
| Авторизация | `Authorization: Bearer <JWT>` — для всех методов |
| Формат | `application/json` |

---

## Методы

| Метод | URL | Назначение |
|---|---|---|
| `GET` | `/claims/statuses` | Справочник статусов для выпадающих списков «Все статусы» и «Изменить статус» |
| `PATCH` | `/claims/{claim_id}/status` | Изменение статуса претензионного дела |

`claim_id` — номер дела в формате `CLM-ГГГГ-NNNNNN`, например `CLM-2026-123457`.

### GET /claims/statuses

**Ответ `200 OK`**

```json
{
  "items": [
    { "status_name": "Создан", "status_code": "CREATED" },
    { "status_name": "В работе", "status_code": "IN_PROGRESS" },
    { "status_name": "Направлено поставщику", "status_code": "SENT" },
    { "status_name": "Ответ получен", "status_code": "RESPONSE_RECEIVED" },
    { "status_name": "На согласовании", "status_code": "UNDER_APPROVAL" },
    { "status_name": "Согласовано", "status_code": "AGREED" },
    { "status_name": "Спор с поставщиком", "status_code": "DISPUTE" },
    { "status_name": "Закрыта", "status_code": "CLOSED" }
  ]
}
```

Коды ответов: `200`, `401`, `500`.

### PATCH /claims/{claim_id}/status

**Запрос**

```json
{
  "status_code": "UNDER_APPROVAL"
}
```

**Ответ `200 OK`**

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

Каждое успешное изменение записывается в историю.

**Идемпотентность.** Если `status_code` совпадает с текущим статусом дела, сервер возвращает `200 OK` с текущим состоянием, не добавляет запись в историю и не изменяет `changed_at`. Повторное нажатие на кнопку не создаёт дублей.

---

## Правила перехода статусов

### Порядок проверок

```
400 формат запроса
 → 404 дело не найдено
 → 200 статус совпадает с текущим
 → 409 перехода нет в матрице
 → 403 у пользователя нет роли для перехода
 → 422 условие перехода не выполнено
```

### Матрица переходов

Любой переход, которого нет в таблице, возвращает `409`. Из конечного статуса `CLOSED` переходов нет.

| № | Откуда → куда | Роль (иначе `403`) | Условие (иначе `422`) |
|---|---|---|---|
| T1 | `CREATED` → `IN_PROGRESS` | Специалист ВЭД | — |
| T2 | `IN_PROGRESS` → `SENT` | Специалист ВЭД | Evidence Pack полный; обязательные поля Claim Letter заполнены; срок предъявления претензии по контракту не истёк |
| T3 | `SENT` → `RESPONSE_RECEIVED` | Специалист ВЭД | — |
| T4 | `RESPONSE_RECEIVED` → `UNDER_APPROVAL` | Специалист ВЭД | Ответ поставщика прикреплён |
| T5 | `UNDER_APPROVAL` → `AGREED` | Руководитель ВЭД | — |
| T6 | `UNDER_APPROVAL` → `DISPUTE` | Руководитель ВЭД | — |
| T7 | `AGREED` → `CLOSED` | Специалист ВЭД | Компенсация получена |
| T8 | `DISPUTE` → `CLOSED` | Юрист ВЭД | — |
| T9 | `CREATED` → `CLOSED` | Специалист ВЭД | — |
| T10 | `IN_PROGRESS` → `CLOSED` | Специалист ВЭД | — |

Описание статусов и действия при каждом переходе (SLA, уведомления, итог закрытия) — в [`status-names.md`](./status-names.md).

---

## Ошибки

Все ошибки возвращаются в едином формате:

```json
{
  "code": "TRANSITION_NOT_ALLOWED",
  "message": "Переход «Направлено поставщику» → «Согласовано» недопустим"
}
```

| HTTP | `code` | Когда возникает |
|---|---|---|
| `400` | `VALIDATION_ERROR` | Неверный формат `claim_id` или значение `status_code` не из справочника |
| `401` | `UNAUTHORIZED` | Отсутствует или недействителен токен |
| `403` | `ROLE_NOT_ALLOWED` | Роль пользователя не совпадает с ролью перехода в матрице |
| `404` | `CLAIM_NOT_FOUND` | Претензионное дело не найдено |
| `409` | `TRANSITION_NOT_ALLOWED` | Перехода нет в матрице, в том числе любой переход из `CLOSED` |
| `422` | `EVIDENCE_PACK_INCOMPLETE` | T2: Evidence Pack заполнен не полностью |
| `422` | `CLAIM_LETTER_INCOMPLETE` | T2: не заполнены обязательные поля Claim Letter |
| `422` | `CLAIM_DEADLINE_EXPIRED` | T2: истёк срок предъявления претензии по контракту |
| `422` | `SUPPLIER_RESPONSE_MISSING` | T4: ответ поставщика не прикреплён |
| `422` | `COMPENSATION_NOT_RECEIVED` | T7: компенсация не получена |
| `500` | `INTERNAL_ERROR` | Внутренняя ошибка сервера |

---

## Вне scope текущей версии

- Комментарий, причина перехода и вложения в теле `PATCH`.
- Итог закрытия дела `resolution` (`COMPENSATED`, `REJECTED`, `WITHDRAWN`).
- Экспорт списка дел.
