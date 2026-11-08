# Apache Kafka: топики и события


## 1. `TRIP_DISRUPTION`: изменения статуса 

Топик `trip.disruption`.


| Поле | Описание |
|---|---|
| `eventId` | Уникальный ID события |
| `tripId`, `itineraryItemId`, `flightId` | Поездка, элемент маршрута и рейс (Monitoring берёт их из своей таблицы рейсов под слежением) |
| `flightNumber` | Номер рейса для текста уведомления |
| `type` | `DELAYED` или `CANCELLED` |
| `departureAt`, `arrivalAt` | Новое время вылета и прибытия в UTC (для `CANCELLED` не заполняются). Core Trip записывает их в `Flight` |
| `sourceUpdatedAt` | Когда источник обновил данные (`source_updated_at`) |
| `receivedAt` | Когда получила система (`received_at`), разница нужна для аудита |

```json
{
  "schemaVersion": 1,
  "eventId": "evt-999",
  "tripId": "12345",
  "itineraryItemId": "item-1",
  "flightId": "flt-1",
  "flightNumber": "SU-100",
  "type": "DELAYED",
  "departureAt": "2026-11-07T12:00:00Z",
  "arrivalAt": "2026-11-07T16:30:00Z",
  "sourceUpdatedAt": "2026-11-07T08:00:00Z",
  "receivedAt": "2026-11-07T10:15:00Z"
}
```

Key: `12345` · Headers: `trace_id: abc-123`, `source_service: monitoring-service`


## 2. `ALTERNATIVE_BOOKED`: выбранная альтернатива забронирована

Топик `alternative.booked`.

| Поле | Описание |
|---|---|
| `eventId` | Уникальный ID события |
| `tripId`, `itineraryItemId` | Какая поездка и какой элемент заменён |
| `alternativeId` | Выбранная альтернатива |
| `bookedByBankUserId` | Организатор, который выбрал вариант |
| `replacedFlightId` | Рейс, который заменяется |
| `airlineIataCode`, `flightNumber` | Новый рейс |
| `departureAirportIata`, `arrivalAirportIata` | Аэропорты нового рейса |
| `departureAt`, `arrivalAt` | Время нового рейса в UTC |
| `bookings` | Список броней: `bankUserId` и `bookingReference` на каждого участника |

```json
{
  "schemaVersion": 1,
  "eventId": "evt-801",
  "tripId": "12345",
  "itineraryItemId": "item-1",
  "alternativeId": "alt-2",
  "bookedByBankUserId": "bank-user-1",
  "replacedFlightId": "flt-1",
  "airlineIataCode": "SU",
  "flightNumber": "SU-200",
  "departureAirportIata": "SVO",
  "arrivalAirportIata": "IST",
  "departureAt": "2026-11-07T15:00:00Z",
  "arrivalAt": "2026-11-07T19:30:00Z",
  "bookings": [
    { "bankUserId": "bank-user-1", "bookingReference": "ABC123" }
  ]
}
```

Key: `12345` · Headers: `trace_id: abc-123`, `source_service: alternatives-service`

## 3. `FLIGHT_TRACKING_UPDATED`: рейс поставлен на слежение или изменен

Топик `flight-tracking.updated`.

| Поле | Описание |
|---|---|
| `eventId` | Уникальный ID события |
| `action` | `ADDED`, `UPDATED` или `REMOVED` |
| `tripId`, `itineraryItemId`, `flightId` | Поездка, элемент маршрута и рейс |
| `airlineIataCode`, `flightNumber` | Авиакомпания и номер рейса |
| `departureAirportIata`, `arrivalAirportIata` | Аэропорты вылета и прибытия |
| `departureAt`, `arrivalAt` | Текущее время вылета и прибытия в UTC |
| `status` | Текущий статус рейса |
| `allowedDelayMinutes` | Допустимая задержка с учётом пересадки (если пересадки нет, 30 минут) |


```json
{
  "schemaVersion": 1,
  "eventId": "evt-555",
  "action": "ADDED",
  "tripId": "12345",
  "itineraryItemId": "item-1",
  "flightId": "flt-1",
  "airlineIataCode": "SU",
  "flightNumber": "SU-100",
  "departureAirportIata": "SVO",
  "arrivalAirportIata": "IST",
  "departureAt": "2026-11-07T09:00:00Z",
  "arrivalAt": "2026-11-07T13:30:00Z",
  "status": "SCHEDULED",
  "allowedDelayMinutes": 30
}
```

Key: `flt-1` · Headers: `trace_id: abc-123`, `source_service: core-trip-service`
