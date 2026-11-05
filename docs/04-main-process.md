# 4. Главный процесс: аренда машины

Аренда реализована как сага с оркестратором в Rental Service ([ADR-0001](05-adr/0001-saga-orchestration.md)). Она проходит четыре фазы: бронирование (синхронно, пока пользователь ждёт ответа), старт (асинхронная команда машине), поездка и завершение (тоже асинхронная команда машине) и оплата (асинхронно в Billing).

Диаграммы последовательностей в этом файле широкие, на странице они уменьшаются. Под каждой есть ссылка на SVG в полном размере.

## Состояния аренды

```mermaid
stateDiagram-v2
    [*] --> CREATING: POST /rentals
    CREATING --> RESERVED: допуск, резерв и hold прошли
    CREATING --> REJECTED: шаг не удался,<br/>компенсации выполнены

    RESERVED --> STARTING: POST /start
    RESERVED --> CANCELLED: POST /cancel
    RESERVED --> EXPIRED: таймер 20 мин или<br/>CarReservationLapsed

    STARTING --> ACTIVE: CarUnlocked
    STARTING --> START_FAILED: CarCommandFailed

    ACTIVE --> FINISHING: POST /finish,<br/>finish-check прошёл
    FINISHING --> ACTIVE: CarCommandFailed REJECTED<br/>(например, открыта дверь)
    FINISHING --> PAYMENT_PENDING: CarLocked

    PAYMENT_PENDING --> CLOSED: PaymentCaptured
    PAYMENT_PENDING --> DEBT: PaymentFailed
    DEBT --> CLOSED: DebtSettled

    REJECTED --> [*]
    CANCELLED --> [*]
    EXPIRED --> [*]
    START_FAILED --> [*]
    CLOSED --> [*]
```

Каждый переход выполняется одной локальной транзакцией в Rental DB:
`UPDATE rentals SET status=:to, version=version+1 WHERE id=:id AND status=:from AND version=:v`, плюс запись в `rental_steps` и, если нужно, в `outbox`. Если `UPDATE` затронул 0 строк, значит другой переход уже случился (например, таймер истечения и `POST /start` пришли одновременно). Такое событие игнорируется или получает ответ `409`.

Ещё один инвариант Rental: у пользователя не больше одной активной аренды. Его обеспечивает частичный уникальный индекс `UNIQUE(user_id) WHERE status IN ('CREATING','RESERVED','STARTING','ACTIVE','FINISHING')`.

## Шаги саги и компенсации

| # | Шаг | Участник | Тип | Компенсация | Если компенсация не прошла |
|---|---|---|---|---|---|
| 1 | Проверить допуск | User | REST, чтение | не нужна | |
| 2 | Зарезервировать машину | Fleet | REST | `DELETE …/reservations/{rentalId}` | ретраи из `rental_steps`, на крайний случай lease во Fleet |
| 3 | Поставить hold | Billing → PSP | REST | `DELETE /internal/holds/{rentalId}` (void) | ретраи, на крайний случай reconciler в Billing и автоматическое снятие hold банком примерно через 7 дней |
| 4 | Открыть машину | Telematics → машина | Kafka-команда | `LockCar(purpose=COMPENSATION)` | ретраи команды, алерт оператору |
| 5 | Закрыть машину | Telematics → машина | Kafka-команда | нет, только повтор вперёд | повтор каждые 2 мин до 30 мин, затем инцидент |
| 6 | Рассчитать и списать | Billing → PSP | Kafka-событие | нет, только движение вперёд (долг и повторы списания) | долг, блокировка новых аренд |

Шаги 5 и 6 поворотные (pivot): если человек уже проехал на машине, поездку не отменить. С этого места сага идёт только вперёд, а сбои обрабатываются повторами и долгом, без отката.

---

## Успешный сценарий

```mermaid
sequenceDiagram
    autonumber
    actor D as Водитель (App)
    participant R as Rental
    participant U as User
    participant F as Fleet
    participant B as Billing
    participant K as Kafka
    participant T as Telematics
    participant C as Машина (провайдер)
    participant P as PSP
    participant N as Notification

    rect rgba(46,111,183,0.08)
    Note over D,P: Фаза 1. Бронирование (синхронно, 1-3 с)
    D->>R: POST /rentals {carId, tariffCode}<br/>Idempotency-Key k1
    R->>R: INSERT rental CREATING, tariff snapshot
    R->>U: GET /internal/users/{userId}/eligibility
    U-->>R: 200 eligible=true
    R->>F: POST /internal/cars/{carId}/reservations<br/>{rentalId, leaseUntil = now+25 мин}
    F->>F: UPDATE cars SET status=RESERVED<br/>WHERE id=carId AND status=AVAILABLE
    F-->>R: 201 reservation
    R->>B: POST /internal/holds {rentalId, 1000 ₽}
    B->>P: создать платёж capture=false<br/>ключ rentalId:hold
    P-->>B: waiting_for_capture
    B-->>R: 201 HELD
    R->>R: CREATING → RESERVED<br/>outbox RentalReserved, таймер +20 мин
    R-->>D: 201 RESERVED, expiresAt
    R--)K: RentalReserved
    K--)N: push «Машина ждёт вас 20 минут»
    end

    rect rgba(39,174,96,0.08)
    Note over D,C: Фаза 2. Старт поездки (асинхронно, 2-10 с)
    D->>R: POST /rentals/{id}/start
    R->>R: RESERVED → STARTING<br/>outbox UnlockCar cmd1, deadline +60 с
    R-->>D: 202 STARTING
    R--)K: UnlockCar cmd1
    K--)T: UnlockCar cmd1
    T->>T: INSERT car_commands cmd1
    T->>C: unlock
    C-->>T: callback SUCCESS, odometer
    T--)K: CarUnlocked cmd1
    K--)R: CarUnlocked cmd1
    R->>R: STARTING → ACTIVE<br/>outbox RentalStarted
    R--)K: RentalStarted
    K--)F: RESERVED → IN_TRIP
    K--)N: push «Поездка началась»
    end

    rect rgba(243,156,18,0.08)
    Note over D,C: Фаза 3. Поездка и завершение
    loop каждые 5-10 с
        C--)T: телеметрия
        T--)K: CarTelemetryReported
        K--)F: координаты, топливо, двери
    end
    D->>R: POST /rentals/{id}/finish
    R->>F: GET /internal/cars/{carId}/finish-check
    F-->>R: allowed=true
    R->>R: ACTIVE → FINISHING<br/>outbox LockCar cmd2 FINISH_TRIP
    R-->>D: 202 FINISHING
    R--)K: LockCar cmd2
    K--)T: LockCar cmd2
    T->>C: lock
    C-->>T: callback SUCCESS, odometer
    T--)K: CarLocked cmd2
    K--)R: CarLocked cmd2
    R->>R: FINISHING → PAYMENT_PENDING<br/>outbox RentalFinished 37 мин, 14.2 км
    R--)K: RentalFinished
    K--)F: IN_TRIP → AVAILABLE
    end

    rect rgba(192,57,43,0.08)
    Note over B,P: Фаза 4. Оплата (асинхронно)
    K--)B: RentalFinished
    B->>B: inbox + INSERT invoice UNIQUE(rental_id)<br/>37 мин × 12,90 ₽ = 477,30 ₽<br/>payment_operation capture PENDING
    B->>P: capture 477,30 ₽ из hold<br/>ключ rentalId:capture
    P-->>B: succeeded, остаток hold освобождён
    B->>B: проводки в ledger<br/>outbox PaymentCaptured
    B--)K: PaymentCaptured
    K--)R: PAYMENT_PENDING → CLOSED<br/>outbox RentalClosed
    K--)N: чек на 477,30 ₽
    end
```

[Открыть диаграмму в полном размере](diagrams/04-main-process-2.svg)

Что важно в успешном сценарии:
- Синхронно выполняется только фаза 1. Пользователь получает ответ «забронировано» только после того, как машина действительно закреплена за ним и депозит заблокирован.
- `finishedAt` берётся из подтверждения машины, а не из времени нажатия кнопки. Так Billing считает фактическое время, и повторный `POST /finish` ничего не меняет.
- Fleet освобождает машину сразу после `RentalFinished`, не дожидаясь оплаты. Физическая доступность машины не зависит от денег.
- Если стоимость больше hold (например, 1450 ₽ при hold 1000 ₽), Billing делает `capture` на 1000 ₽ и доплату 450 ₽ по сохранённой карте с ключом `rentalId:charge-extra`. Что происходит, если доплата не проходит, показывает сценарий сбоя 3.

---

## Сценарий сбоя 1. Машина не открылась

**Что ломается:** телематический блок не отвечает. Машина стоит в подземном паркинге без связи или у неё разряжен АКБ. Через 3 минуты связь появляется, и машина всё-таки выполняет команду, которая уже считается просроченной.

**Чем опасно:** пользователь стоит у закрытой машины, а депозит заморожен. Кроме того, машина может открыться, когда аренды уже нет, и остаться открытой без присмотра.

```mermaid
sequenceDiagram
    autonumber
    actor D as Водитель (App)
    participant R as Rental
    participant F as Fleet
    participant B as Billing
    participant K as Kafka
    participant T as Telematics
    participant C as Машина (провайдер)
    participant P as PSP
    participant N as Notification

    D->>R: POST /rentals/{id}/start
    R->>R: RESERVED → STARTING<br/>outbox UnlockCar cmd1, deadline +60 с
    R-->>D: 202 STARTING
    R--)K: UnlockCar cmd1
    K--)T: UnlockCar cmd1
    loop бэкофф 1, 2, 4, 8, 16 с до deadline
        T->>C: unlock cmd1
        C--xT: нет ответа
    end
    Note over T: deadline истёк, cmd1 → TIMED_OUT
    T--)K: CarCommandFailed cmd1 TIMEOUT
    K--)R: CarCommandFailed cmd1
    R->>R: STARTING → START_FAILED<br/>rental_steps: компенсации 2 и 3

    par компенсация шага 2
        R->>F: DELETE …/reservations/{rentalId}<br/>reason=START_FAILED, markForInspection=true
        F->>F: RESERVED → NEEDS_INSPECTION
        F-->>R: 204
    and компенсация шага 3
        R->>B: DELETE /internal/holds/{rentalId}
        B->>P: cancel, ключ rentalId:void
        P-->>B: canceled
        B-->>R: 204
    end
    R--)K: RentalStartFailed CAR_UNREACHABLE
    K--)N: push «Машина не открылась. Деньги не списаны. Рядом есть другая»

    rect rgba(192,57,43,0.10)
    Note over T,C: Через 3 минуты связь появилась, машина выполнила старую команду
    C-->>T: callback SUCCESS cmd1
    T->>T: cmd1 уже TIMED_OUT, результат всё равно публикуется
    T--)K: CarUnlocked cmd1, lateAfterTimeout=true
    K--)R: CarUnlocked cmd1
    R->>R: аренда в START_FAILED, это не старт<br/>outbox LockCar cmd3 COMPENSATION
    R--)K: LockCar cmd3
    K--)T: LockCar cmd3
    T->>C: lock cmd3
    C-->>T: SUCCESS
    T--)K: CarLocked cmd3
    K--)R: CarLocked cmd3, записать в rental_steps
    end
    Note over F: Машина остаётся в NEEDS_INSPECTION,<br/>оператор проверяет её перед возвратом на линию
```

[Открыть диаграмму в полном размере](diagrams/04-main-process-3.svg)

**Как система компенсирует:**
1. У команды в Telematics есть явный дедлайн, поэтому пользователь не ждёт бесконечно: через 60 секунд он получает понятный ответ.
2. Rental выполняет компенсации шагов 2 и 3 параллельно. Обе идемпотентны, и при сбое (например, если Billing недоступен) их повторяет фоновый обработчик `rental_steps` с бэкоффом.
3. Машина уходит в `NEEDS_INSPECTION`, а не в `AVAILABLE`, иначе следующий пользователь наткнётся на ту же проблему.
4. Если машина открылась с опозданием, Telematics не выбрасывает поздний результат, а публикует его с флагом `lateAfterTimeout`. Rental видит, что аренда уже в `START_FAILED`, и отправляет компенсирующую команду `LockCar`, поэтому машина не остаётся открытой.
5. Команды `UnlockCar` и `LockCar` для одной машины не меняются местами, потому что ключом партиции в `telematics.commands` служит `carId`.

---

## Сценарий сбоя 2. Платёжный шлюз завис во время бронирования

**Что ломается:** машина уже зарезервирована, Billing отправил в PSP запрос на hold, а PSP не ответил за 8 секунд. Результат неизвестен: деньги могли заблокироваться, а могли и нет. В более сложном варианте падает сам Rental между шагами.

**Чем опасно:** если просто вернуть ошибку, машина останется зарезервированной без аренды, а на карте может остаться hold. Если повторить hold с новым ключом, можно заблокировать деньги дважды.

```mermaid
sequenceDiagram
    autonumber
    actor D as Водитель (App)
    participant R as Rental
    participant U as User
    participant F as Fleet
    participant B as Billing
    participant P as PSP

    D->>R: POST /rentals<br/>Idempotency-Key k1
    R->>R: INSERT rental CREATING
    R->>U: eligibility
    U-->>R: eligible=true
    R->>F: POST …/reservations
    F-->>R: 201, машина RESERVED
    R->>B: POST /internal/holds
    B->>B: INSERT payment_operation hold PENDING
    B-xP: создать платёж, ключ rentalId:hold
    Note over B,P: таймаут 8 с, результат неизвестен
    B->>B: hold → UNKNOWN
    B-->>R: 503 HOLD_UNKNOWN

    R->>R: rental_steps: компенсации 3 и 2
    R->>B: DELETE /internal/holds/{rentalId}
    B->>B: UNKNOWN → VOID_REQUESTED
    B-->>R: 204
    R->>F: DELETE …/reservations/{rentalId}<br/>reason=HOLD_FAILED
    F->>F: RESERVED → AVAILABLE
    F-->>R: 204
    R->>R: CREATING → REJECTED HOLD_UNKNOWN
    R-->>D: 503 «Не удалось заблокировать депозит,<br/>попробуйте ещё раз», Retry-After 5

    rect rgba(192,57,43,0.10)
    Note over B,P: Позже выясняется, что hold всё-таки прошёл
    alt PSP присылает вебхук
        P->>B: POST /webhooks/psp<br/>waiting_for_capture, rentalId:hold
    else вебхука нет, reconciler раз в минуту
        B->>P: GET платёж по ключу rentalId:hold
        P-->>B: waiting_for_capture
    end
    B->>B: статус VOID_REQUESTED, значит отменить
    B->>P: cancel, ключ rentalId:void
    P-->>B: canceled
    B->>B: VOID_REQUESTED → VOIDED, проводки в ledger
    end

    opt Вариант: Rental упал после резерва, до hold
        Note over R: Recovery job при старте и раз в 30 с:<br/>аренды в CREATING старше 30 с<br/>получают компенсации по rental_steps → REJECTED
        Note over F: Если и recovery не сработал,<br/>Fleet сам освободит машину по leaseUntil<br/>и опубликует CarReservationLapsed
    end
```

[Открыть диаграмму в полном размере](diagrams/04-main-process-4.svg)

**Как система компенсирует:**
1. Неизвестный результат обрабатывается как неудача с отложенной отменой. Rental компенсирует всё, что успел сделать, и отвечает пользователю `503`. Повторить hold с новым ключом Billing не может, потому что ключ детерминированный: `rentalId:hold`.
2. Отмена может прийти раньше подтверждения. Для hold в статусе `UNKNOWN` запрос `DELETE /holds` ставит метку `VOID_REQUESTED`. Когда PSP сообщит, что hold прошёл (через вебхук или при опросе reconciler'ом), Billing сразу его отменит.
3. Компенсации идут в обратном порядке (сначала hold, потом машина) и записываются в `rental_steps` до вызова. Если Rental упадёт посреди компенсаций, recovery job продолжит с места остановки.
4. Подстраховка есть на трёх уровнях: recovery job в Rental, lease во Fleet и автоматическое снятие hold банком.
5. Повтор с тем же `Idempotency-Key k1` разрешён: ответы `5xx` не кешируются, а прошлая попытка уже в терминальном `REJECTED`, поэтому создаётся новая аренда.

---

## Сценарий сбоя 3. Оплата после поездки: таймаут, падение Billing и нехватка денег

**Что ломается:** поездка стоила 1450 ₽, hold был 1000 ₽. Billing отправляет `capture` в PSP, PSP списывает деньги, но ответ теряется, и Billing падает. После рестарта Kafka доставляет `RentalFinished` повторно. Затем доплата 450 ₽ отклоняется, потому что на карте не хватает денег.

**Чем опасно:** если просто повторять запрос, пользователь заплатит дважды. Если просто записать ошибку, компания потеряет 450 ₽. Откатить поездку при этом нельзя.

```mermaid
sequenceDiagram
    autonumber
    participant K as Kafka
    participant B as Billing
    participant W as Billing payment worker
    participant P as PSP
    participant R as Rental
    participant U as User
    participant N as Notification
    actor D as Водитель (App)

    K--)B: RentalFinished e1 rentalId=r1
    B->>B: TX: inbox e1, invoice r1 1450 ₽<br/>op capture 1000 ₽ PENDING<br/>op charge-extra 450 ₽ PENDING
    B->>K: commit offset

    W->>P: capture 1000 ₽, ключ r1:capture
    P->>P: деньги списаны
    P--xW: ответ потерян, таймаут
    Note over W: Billing падает

    rect rgba(46,111,183,0.08)
    Note over K,W: Рестарт. Повторная доставка и повтор операции
    K--)B: RentalFinished e1, повторная доставка
    B->>B: inbox: e1 уже есть, пропустить
    W->>W: op r1:capture в PENDING, повторить
    W->>P: capture 1000 ₽, ключ r1:capture
    P-->>W: succeeded, тот же платёж, без нового списания
    W->>W: op capture → SUCCEEDED, ledger
    end

    W->>P: доплата 450 ₽ по сохранённой карте<br/>ключ r1:charge-extra:1
    P-->>W: declined INSUFFICIENT_FUNDS
    W->>W: TX: invoice PARTIALLY_PAID<br/>debt 450 ₽, retry через 1 ч<br/>outbox PaymentFailed
    W--)K: PaymentFailed r1, debt 450 ₽
    K--)R: PAYMENT_PENDING → DEBT
    K--)U: hasDebt = true
    K--)N: push «Списано 1000 ₽, не хватило 450 ₽»

    D->>R: POST /rentals, новая бронь
    R->>U: eligibility
    U-->>R: eligible=false HAS_DEBT
    R-->>D: 403 HAS_DEBT

    rect rgba(39,174,96,0.08)
    Note over W,P: Через 1 час автоматический повтор с новым ключом
    W->>P: доплата 450 ₽, ключ r1:charge-extra:2
    P-->>W: succeeded
    W->>W: debt закрыт, ledger<br/>outbox DebtSettled
    W--)K: DebtSettled r1
    K--)R: DEBT → CLOSED, RentalClosed
    K--)U: hasDebt = false, других долгов нет
    K--)N: push «Долг погашен», чек
    end
```

[Открыть диаграмму в полном размере](diagrams/04-main-process-5.svg)

**Как система компенсирует:**
1. Для двух разных причин повтора используются разные ключи. Если результат неизвестен (таймаут), запрос повторяется с тем же ключом (`r1:capture`), и PSP вернёт уже проведённый платёж. Если пришёл окончательный отказ (declined), следующая попытка считается новой операцией с новым ключом (`r1:charge-extra:2`). Если перепутать эти случаи, получится либо двойное списание, либо ошибка, которая будет повторяться бесконечно.
2. Намерение записывается раньше действия: операции `PENDING` сохраняются в той же транзакции, что и счёт, ещё до вызова PSP. После рестарта payment worker находит незавершённые операции и доводит их до конца.
3. Повторная доставка события безопасна благодаря inbox по `eventId` и `UNIQUE(rental_id)` на счёте.
4. Компенсация здесь идёт вперёд, а не назад. Поездку отменить нельзя, поэтому система открывает долг, блокирует новые аренды через User и повторяет списание по расписанию (через 1 ч, 6 ч, 24 ч, затем раз в сутки). Пользователь может погасить долг сам через `POST /me/debts/{id}/pay`.
5. Раз в сутки reconciler сверяет `payment_operations` с реестром PSP. Если расхождение не исправляется автоматически, создаётся задача финансовому оператору.

---

## Другие сбои

| Сбой | Поведение системы |
|---|---|
| Kafka недоступна | Сервисы продолжают принимать запросы, события копятся в `outbox`. Relay публикует их после восстановления. Новые старты и завершения задерживаются, но не теряются. |
| Rental упал во время поездки | Поездка продолжается: машина открыта, тарификация считается по времени из `CarLocked`. Кнопка «Завершить» временно не работает, приложение показывает статус. |
| Двойное нажатие «Завершить» | Второй вызов видит `FINISHING` и возвращает `202` с текущим состоянием. Новой команды нет. |
| `LockCar` не исполнился (машина офлайн при завершении) | Rental повторяет команду каждые 2 мин в течение 30 мин, затем создаёт инцидент для оператора. Время пользователю считается до нажатия «Завершить», потому что задержка не по его вине. |
| `LockCar` отклонён (дверь открыта) | `FINISHING → ACTIVE`, push «Закройте дверь». Тарификация продолжается. |
| Пользователь не пришёл к машине | Через 20 минут срабатывает таймер Rental: `RESERVED → EXPIRED`, затем компенсации шагов 2 и 3. Если таймер не сработал, Fleet по `leaseUntil` публикует `CarReservationLapsed`. |
| Notification или SMS-шлюз недоступен | Сага от этого не зависит. Уведомления доставляются позже, дубли отсекаются по `eventId`. |
| Ядовитое сообщение (не парсится) | После 5 попыток уходит в `*.DLT`, срабатывает алерт, партиция не блокируется. |
