# 2. C4, уровень Container

Диаграммы нарисованы в нотации C4 (человек, контейнер с технологией, хранилище, внешняя система, граница системы, протокол на связях) через Mermaid `flowchart`. 

**Легенда**

- тёмно-синий: человек;
- синий: наш контейнер, цилиндр обозначает базу данных;
- голубой: инфраструктурный контейнер (API Gateway, Keycloak);
- фиолетовый: брокер Kafka;
- серый с пунктирной рамкой: внешняя система;
- сплошная стрелка: синхронный вызов, пунктирная: обмен через Kafka.

## Context (уровень 1, для ориентира)

```mermaid
flowchart TB
    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff,stroke-dasharray:5 3

    driver["<b>Водитель</b><br/>[Person]<br/>Ищет, бронирует, открывает<br/>машину, платит за поездку"]:::person
    operator["<b>Оператор автопарка</b><br/>[Person]<br/>Ремонт, мойка, инциденты"]:::person

    sys["<b>Каршеринг «Поехали»</b><br/>[Software System]<br/>Бронирование, поездки, расчёты"]:::system

    telematics["<b>Телематический провайдер</b><br/>[External System]<br/>IoT-блоки в машинах"]:::ext
    psp["<b>Платёжный шлюз (PSP)</b><br/>[External System]<br/>Hold, capture, refund"]:::ext
    kyc["<b>Сервис проверки ВУ</b><br/>[External System]<br/>KYC"]:::ext
    push["<b>Push / SMS / Email</b><br/>[External System]<br/>FCM, APNs, SMS, SMTP"]:::ext

    driver -- "Мобильное приложение<br/>[HTTPS]" --> sys
    operator -- "Веб-консоль<br/>[HTTPS]" --> sys
    sys -- "Команды, телеметрия<br/>[HTTPS + вебхуки]" --> telematics
    sys -- "Платежи<br/>[HTTPS + вебхуки]" --> psp
    sys -- "Проверка прав<br/>[HTTPS]" --> kyc
    sys -- "Уведомления<br/>[HTTPS]" --> push
```

## Container (уровень 2)

```mermaid
flowchart TB
    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef container fill:#438dd5,stroke:#2e6295,color:#fff
    classDef db fill:#438dd5,stroke:#2e6295,color:#fff
    classDef infra fill:#85bbf0,stroke:#5d82a8,color:#000
    classDef queue fill:#6a4c93,stroke:#4a3466,color:#fff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff,stroke-dasharray:5 3

    driver["<b>Водитель</b><br/>[Person]"]:::person
    operator["<b>Оператор</b><br/>[Person]"]:::person

    subgraph SYS["Каршеринг «Поехали» [Software System]"]
        app["<b>Mobile App</b><br/>[Container: Kotlin / Swift]<br/>Карта, бронь, открыть / закрыть"]:::container
        console["<b>Operator Console</b><br/>[Container: React SPA]<br/>Управление автопарком"]:::container
        gw["<b>API Gateway</b><br/>[Container: Spring Cloud Gateway]<br/>JWT, rate limiting, маршрутизация"]:::infra
        idp["<b>Identity Provider</b><br/>[Container: Keycloak]<br/>OAuth2 / OIDC"]:::infra

        user["<b>User Service</b><br/>[Container: Python / FastAPI]<br/>Водители, проверка ВУ, допуск"]:::container
        rental["<b>Rental Service</b><br/>[Container: Java / Spring Boot]<br/>Аренда, оркестратор саги"]:::container
        fleet["<b>Fleet Service</b><br/>[Container: Go]<br/>Машины, статусы, гео-поиск, резерв"]:::container
        billing["<b>Billing Service</b><br/>[Container: Java / Spring Boot]<br/>Тарифы, счета, hold / capture, долги"]:::container

        kafka[["<b>Event Bus</b><br/>[Container: Apache Kafka]<br/>rental.events, telematics.*,<br/>billing.events, fleet.events"]]:::queue

        telem["<b>Telematics Gateway</b><br/>[Container: Go]<br/>ACL к IoT: команды, ретраи, телеметрия"]:::container
        notif["<b>Notification Service</b><br/>[Container: Python]<br/>Шаблоны и доставка уведомлений"]:::container

        userDb[("<b>User DB</b><br/>[PostgreSQL]<br/>drivers, licenses")]:::db
        rentalDb[("<b>Rental DB</b><br/>[PostgreSQL]<br/>rentals, rental_steps,<br/>outbox, inbox")]:::db
        fleetDb[("<b>Fleet DB</b><br/>[PostgreSQL + PostGIS]<br/>cars, car_reservations")]:::db
        billingDb[("<b>Billing DB</b><br/>[PostgreSQL]<br/>invoices, payment_operations,<br/>ledger, debts")]:::db
        telemDb[("<b>Telematics DB</b><br/>[PostgreSQL]<br/>devices, car_commands")]:::db
        notifDb[("<b>Notification DB</b><br/>[PostgreSQL]<br/>notification_log")]:::db
    end

    kyc["<b>Сервис проверки ВУ</b><br/>[External System]"]:::ext
    psp["<b>Платёжный шлюз (PSP)</b><br/>[External System]"]:::ext
    provider["<b>Телематический провайдер</b><br/>[External System]"]:::ext
    push["<b>Push / SMS / Email</b><br/>[External System]"]:::ext

    driver --> app
    operator --> console
    app -- "REST [HTTPS/JSON]" --> gw
    console -- "REST [HTTPS/JSON]" --> gw
    gw -- "проверка токенов [OIDC]" --> idp

    gw -- "/me" --> user
    gw -- "/rentals" --> rental
    gw -- "/cars" --> fleet
    gw -- "/tariffs, /invoices" --> billing

    rental -- "допуск [REST]" --> user
    rental -- "резерв, снятие [REST]" --> fleet
    rental -- "hold, release [REST]" --> billing

    rental <-.-> kafka
    fleet <-.-> kafka
    billing <-.-> kafka
    user <-.-> kafka
    kafka <-.-> telem
    kafka -.-> notif

    user --> userDb
    rental --> rentalDb
    fleet --> fleetDb
    billing --> billingDb
    telem --> telemDb
    notif --> notifDb

    user -- "[HTTPS]" --> kyc
    billing -- "платежи, вебхуки [HTTPS]" --> psp
    telem -- "команды, колбэки [HTTPS]" --> provider
    notif -- "[HTTPS]" --> push

    style SYS fill:#f5f9ff,stroke:#1168bd,stroke-dasharray:6 4
```

[Открыть диаграмму в полном размере](diagrams/02-c4-container-2.svg)

Подписей на связях с Kafka нет, чтобы схема помещалась в ширину страницы. Кто что публикует и читает, показано в таблице [Топики Kafka](#топики-kafka).

## Пояснения к контейнерам

| Контейнер | Ключевые технические решения |
|---|---|
| **API Gateway** | Единая точка входа. Проверяет JWT и пробрасывает `X-User-Id`. Ограничивает частоту запросов: на `POST /rentals` не больше 5 в минуту на пользователя. Заголовок `Idempotency-Key` сам не обрабатывает, только пропускает дальше. |
| **Rental Service** | Состояние саги хранится в `rentals` (state machine) и `rental_steps` (журнал шагов). События уходят через transactional outbox ([ADR-0002](05-adr/0002-transactional-outbox.md)). Таймеры брони реализованы через таблицу `scheduled_jobs` с поллингом (`SELECT … FOR UPDATE SKIP LOCKED`). |
| **Fleet Service** | Статус машины меняется только условным `UPDATE … WHERE status='AVAILABLE'` ([ADR-0004](05-adr/0004-car-reservation-exclusivity.md)). Поиск машин рядом идёт через индекс GiST по `geography(Point)`. Последние координаты кешируются в памяти с TTL 5 секунд. |
| **Telematics Gateway** | Читает `telematics.commands`, ведёт журнал `car_commands` с уникальным `commandId`, повторяет команды с экспоненциальным бэкоффом, у каждой команды есть дедлайн. Телеметрию от провайдера нормализует и пишет в `telematics.telemetry`. |
| **Billing Service** | Ключ идемпотентности для PSP строится как `{rentalId}:{operation}`. У таблицы `invoices` есть `UNIQUE(rental_id)`. Фоновый reconciler сверяет операции в статусе `UNKNOWN` с PSP. В журнал проводок записи только добавляются (append-only). |
| **Kafka** | 3 брокера, `replication.factor=3`, `min.insync.replicas=2`, продюсеры с `acks=all` и `enable.idempotence=true`. Ключи партиционирования: `rentalId` для аренд и платежей, `carId` для команд и телеметрии ([ADR-0003](05-adr/0003-kafka-as-event-bus.md)). |
| **Базы данных** | Database-per-service: ни один сервис не читает чужую БД. На этапе реализации это может быть один инстанс PostgreSQL с отдельной схемой и отдельным пользователем на каждый сервис. |

## Топики Kafka

| Топик | Ключ | Продюсер | Консьюмеры | Retention |
|---|---|---|---|---|
| `rental.events` | `rentalId` | Rental | Fleet, Billing, Notification | 7 дней |
| `telematics.commands` | `carId` | Rental | Telematics | 1 день |
| `telematics.events` | `carId` | Telematics | Rental | 7 дней |
| `telematics.telemetry` | `carId` | Telematics | Fleet | 1 день, `compact` выключен |
| `billing.events` | `rentalId` | Billing | Rental, User, Notification | 30 дней |
| `fleet.events` | `carId` | Fleet | Rental | 7 дней |
| `*.DLT` | как в исходном топике | любой консьюмер | ручной разбор | 14 дней |
