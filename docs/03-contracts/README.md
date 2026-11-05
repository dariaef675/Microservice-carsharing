# 3. Контракты

```
03-contracts/
├── openapi/                 синхронные вызовы (OpenAPI 3.0.3)
│   ├── rental-api.yaml      публичный API аренды
│   ├── fleet-api.yaml       поиск машин + внутренний резерв
│   ├── billing-api.yaml     тарифы, карты, счета + внутренний hold + вебхук PSP
│   ├── user-api.yaml        профиль + внутренняя проверка допуска
│   └── telematics-api.yaml  колбэки провайдера
└── asyncapi/                события и команды через Kafka (AsyncAPI 3.0.0)
    ├── common.yaml          конверт события (заголовки) и общие схемы
    ├── rental-events.yaml   rental.events
    ├── telematics-events.yaml  telematics.commands / .events / .telemetry
    ├── billing-events.yaml  billing.events
    └── fleet-events.yaml    fleet.events
```

Все файлы проходят валидацию через `npx @redocly/cli lint openapi/*.yaml` и `@asyncapi/parser`. Каждый AsyncAPI-документ описывает сервис, которому принадлежит топик, и то, что этот сервис публикует. В документе Telematics описаны ещё и команды, которые он принимает. Консьюмеры перечислены в описании каждого документа.

## Матрица взаимодействий

| От | К | Тип | Контракт | Зачем |
|---|---|---|---|---|
| Mobile App | Rental | REST | `POST /rentals`, `/start`, `/finish`, `/cancel` | Действия пользователя |
| Mobile App | Fleet | REST | `GET /cars` | Карта машин |
| Mobile App | Billing | REST | `GET /tariffs`, `/me/invoices/{id}`, `/me/payment-methods` | Тарифы, чеки, карты |
| Mobile App | User | REST | `GET /me`, `PUT /me/license` | Профиль, ВУ |
| Rental | User | REST | `GET /internal/users/{id}/eligibility` | Допуск перед бронью |
| Rental | Fleet | REST | `POST/DELETE /internal/cars/{id}/reservations` | Резерв и компенсация |
| Rental | Fleet | REST | `GET /internal/cars/{id}/finish-check` | Можно ли завершить здесь |
| Rental | Billing | REST | `POST/DELETE /internal/holds` | Hold и компенсация |
| Rental | Telematics | Kafka | `telematics.commands`: UnlockCar, LockCar | Команды машине |
| Telematics | Rental | Kafka | `telematics.events`: CarUnlocked, CarLocked, CarCommandFailed | Результат команды |
| Telematics | Fleet | Kafka | `telematics.telemetry` | Координаты, топливо, двери |
| Rental | Fleet, Billing, Notification | Kafka | `rental.events` | Факты аренды |
| Billing | Rental, User, Notification | Kafka | `billing.events` | Итог оплаты, долги |
| Fleet | Rental | Kafka | `fleet.events`: CarReservationLapsed | Страховочное освобождение машины |
| PSP | Billing | Webhook | `POST /webhooks/psp` | Статусы платежей |
| Провайдер телематики | Telematics | Webhook | `POST /webhooks/provider/*` | Результаты команд, телеметрия |

**Почему одни вызовы синхронные, а другие нет.** Синхронно делается то, без чего нельзя ответить пользователю на кнопку «Забронировать»: проверка допуска, резерв машины, hold. Пользователь ждёт эти шаги, и каждый занимает десятки или сотни миллисекунд. Асинхронно делается всё, где ответ приходит от внешней стороны с непредсказуемым временем ответа (машина, PSP при capture), и всё, где у события несколько получателей.

## Общие соглашения

### Идемпотентность
| Где | Механизм |
|---|---|
| `POST /rentals` | Заголовок `Idempotency-Key` (UUID от клиента). Ответ кешируется в `idempotency_keys` на 24 ч. |
| Переходы start/finish/cancel | Идемпотентны по состоянию: повтор в целевом или более позднем статусе ничего не делает и возвращает текущее состояние. |
| Резерв в Fleet, hold в Billing | Ключ идемпотентности `rentalId`. Повтор возвращает `200` и существующий ресурс. |
| Компенсации (`DELETE`) | Всегда `204`, даже если ресурса уже нет. |
| Команды машине | `commandId`. Журнал `car_commands` с `UNIQUE(command_id)`. |
| Все консьюмеры Kafka | Таблица `inbox(event_id PRIMARY KEY)`. Вставка в inbox и бизнес-изменение идут в одной транзакции. |
| Операции с PSP | Детерминированный ключ `{rentalId}:{operation}`. Вебхуки дедуплицируются по `pspEventId`. |

### Ошибки
Синхронные API возвращают ошибки в формате RFC 7807 (`application/problem+json`), с машиночитаемым полем `code` и `traceId`. Классы кодов:
- `4xx`: повторять запрос бессмысленно, кроме `409 INVALID_STATE_TRANSITION` после обновления состояния;
- `503` с заголовком `Retry-After`: запрос можно повторить с тем же `Idempotency-Key`.

### Таймауты и ретраи межсервисных вызовов
| Вызов | Таймаут | Ретраи | Circuit breaker |
|---|---|---|---|
| Rental → User (eligibility) | 500 мс | 2, джиттер | да |
| Rental → Fleet (reserve) | 1 с | 2 (вызов идемпотентный) | да |
| Rental → Billing (hold) | 10 с (Billing ждёт PSP до 8 с) | 0: при неизвестном результате идёт компенсация | да |
| Компенсации (`DELETE`) | 2 с | бесконечно, бэкофф до 5 мин, через `rental_steps` | нет |

### Версионирование
- REST: версия в пути (`/v1`). Добавлять поля можно; ломающее изменение означает `/v2` и полгода параллельной поддержки обеих версий.
- События: `schemaVersion` в заголовке. Добавлять опциональные поля можно. Ломающее изменение требует нового `eventType` (например, `RentalFinished.v2`) с двойной публикацией на время миграции консьюмеров.
- Консьюмеры игнорируют неизвестные поля (tolerant reader).
