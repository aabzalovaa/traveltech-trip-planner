# Message Broker (Kafka): события Trip Recovery

## 1. Обзор

| Событие | Публикует | Читают | Когда |
|---|---|---|---|
| `TRIP_DISRUPTION` | Monitoring Service | Alternatives Service, Core Trip Service, Notification Service | Рейс отменён или задержан выше порога |
| `ALTERNATIVES_READY` | Alternatives Service | Notification Service | Подбор альтернатив завершён |
| `TRIP_PLAN_UPDATED` | Core Trip Service | Notification Service, Monitoring Service | Организатор выбрал рейс, перебронирование подтверждено |

**Сквозной сценарий:**

1. Monitoring видит отмену рейса и публикует `TRIP_DISRUPTION`.
2. Параллельно: Alternatives молча ищет варианты, Core Trip помечает сегмент `NEEDS_ATTENTION`.
3. Alternatives публикует `ALTERNATIVES_READY`.
4. Notification отправляет push об отмене сразу, а по ALTERNATIVES_READY второй push: найдены N вариантов.
5. Организатор выбирает рейс (`PATCH /plan`), Core Trip обновляет план и публикует `TRIP_PLAN_UPDATED`.
6. Notification уведомляет остальных участников, Monitoring начинает следить за новым рейсом.

## 2. `TRIP_DISRUPTION`: обнаружен сбой

**Зачем:** сообщить всей системе, что рейс отменен или задержан, чтобы каждый сервис начал свою работу параллельно.

**Когда публикуем:** отмена или задержка выше порога (например, больше 30 минут или риск пропустить пересадку). Мелкие задержки в событие не превращаются.

| Поле | Описание |
|---|---|
| `eventId` | Уникальный ID события |
| `tripId`, `segmentId` | Какая поездка и какой сегмент пострадал |
| `flightNumber` | Номер рейса |
| `type` | `DELAYED` или `CANCELLED` |
| `delayMinutes` | Длительность задержки (только для `DELAYED`) |
| `reasonCode`, `reason` | Код причины для логики и текст для пользователя |
| `airlineUpdatedAt` | Когда авиакомпания изменила рейс |
| `receivedAt` | Когда мы узнали (разница нужна для аудита) |

```json
{
  "schemaVersion": 1,
  "eventId": "evt-999",
  "tripId": "12345",
  "segmentId": "seg-1",
  "flightNumber": "SU-100",
  "type": "CANCELLED",
  "reasonCode": "AIRLINE_CANCELLATION",
  "reason": "Отмена авиакомпанией",
  "airlineUpdatedAt": "2026-11-07T08:00:00Z",
  "receivedAt": "2026-11-07T10:15:00Z"
}
```

Key: `12345` · Headers: `trace_id: abc-123`, `source_service: monitoring-service`

**Дедупликация в Monitoring:** `segmentId + type + airlineUpdatedAt`. Один и тот же сбой не публикуется дважды, даже если внешний API присылает его повторно.

## 3. `ALTERNATIVES_READY`: альтернативы найдены

**Зачем:** сообщить пользователю, что варианты готовы. Alternatives сначала молча ищет билеты. Когда закончил, Notification отправляет один пуш вместо двух.

| Поле | Описание |
|---|---|
| `eventId` | Уникальный ID события |
| `tripId`, `segmentId` | Для какой поездки и сегмента |
| `originalDisruptionId` | `eventId` исходного `TRIP_DISRUPTION` (связь для Notification) |
| `optionsCount` | Сколько найдено вариантов |

```json
{
  "schemaVersion": 1,
  "eventId": "evt-777",
  "tripId": "12345",
  "segmentId": "seg-1",
  "originalDisruptionId": "evt-999",
  "optionsCount": 3
}
```

Key: `12345` · Headers: `trace_id: abc-123`, `source_service: alternatives-service`

## 4. `TRIP_PLAN_UPDATED`: план обновлён

**Зачем:** многопользовательский режим. Организатор выбрал рейс, остальным участникам нужно узнать о смене плана. Monitoring должен начать отслеживать новый рейс.

**Когда публикуем:** только после подтверждённого перебронирования (`CONFIRMED`), не на `PENDING`.

| Поле | Описание |
|---|---|
| `eventId` | Уникальный ID события |
| `tripId`, `segmentId` | Какая поездка и сегмент заменен |
| `updatedBy` | ID Организатора, который внес изменение |
| `oldFlightNumber`, `newFlightNumber` | Что было и что стало |

```json
{
  "schemaVersion": 1,
  "eventId": "evt-888",
  "tripId": "12345",
  "segmentId": "seg-1",
  "updatedBy": "user_organizer_1",
  "oldFlightNumber": "SU-100",
  "newFlightNumber": "SU-200"
}
```

Key: `12345` · Headers: `trace_id: abc-123`, `source_service: core-trip-service`

**Получатели:** Notification по `tripId` запрашивает у Core Trip список участников.
