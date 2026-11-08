
## 1. Таблицы БД

| БД | Сервис | Таблицы |
|---|---|---|
| Trip DB | Core Trip Service | `Trip`, `TripMember`, `ItineraryItem`, `Flight`, `Booking`, `Airport`, `Airline` |
| Monitoring DB | Monitoring Service | `FlightSource`, `FlightStatusHistory`, `MonitoredFlight` (новая) |
| Alternatives DB | Alternatives Service | `Alternative`, `PlanChange` (новая) |
| Notification DB | Notification Service | `Notification` |


## 2. Trip DB (Core Trip Service)
<img width="2351" height="1136" alt="image" src="https://github.com/user-attachments/assets/54c3227d-8c0f-47d0-964e-af7446c92685" />

```plantuml
@startuml
!define Table(name,desc) entity name as "desc" << (T,#FFAAAA) >>

title Trip DB (Core Trip Service)

' Внешняя сущность. Не является таблицей БД авиасервиса.
entity "Bank Platform\n(внешняя система)" as BankPlatform <<external>> {
  bank_user_id: STRING | PK (Идентификатор клиента в банковской платформе)
}

Table(Trip, "Trip (Поездка)") {
  id: UUID | PK (Идентификатор поездки)
  --
  name: STRING | NOT NULL (Название поездки)
  start_date: DATE | NOT NULL (Дата начала)
  end_date: DATE | NOT NULL (Дата окончания)
  status: ENUM(PLANNED, ACTIVE, COMPLETED, CANCELLED) | NOT NULL (Статус поездки)
  created_by_bank_user_id: STRING | NOT NULL (ID создателя во внешней банковской платформе)
  created_at: TIMESTAMP | NOT NULL (Дата создания)
}

Table(TripMember, "TripMember (Участник поездки)") {
  id: UUID | PK (Идентификатор участника поездки)
  --
  trip_id: UUID | FK | NOT NULL (Идентификатор поездки)
  bank_user_id: STRING | NOT NULL (ID пользователя во внешней банковской платформе)
  role: ENUM(OWNER, MEMBER) | NOT NULL (Роль участника)
  --
  UNIQUE(trip_id, bank_user_id) (Один пользователь добавляется в поездку один раз)
}

Table(ItineraryItem, "ItineraryItem (Элемент маршрута)") {
  id: UUID | PK (Идентификатор элемента маршрута)
  --
  trip_id: UUID | FK | NOT NULL (Идентификатор поездки)
  type: ENUM(FLIGHT, HOTEL, TRANSFER, ACTIVITY, OTHER) | NOT NULL (Тип элемента)
  title: STRING | NOT NULL (Название элемента)
  start_at: TIMESTAMP (Время начала)
  end_at: TIMESTAMP (Время окончания)
  status: STRING | NOT NULL (Статус элемента маршрута)
}

Table(Flight, "Flight (Рейс)") {
  id: UUID | PK (Идентификатор рейса)
  --
  itinerary_item_id: UUID | FK | UNIQUE | NOT NULL (Элемент маршрута, к которому относится рейс)
  airline_id: UUID | FK | NOT NULL (Авиакомпания)
  flight_number: STRING | NOT NULL (Номер рейса)
  departure_airport_id: UUID | FK | NOT NULL (Аэропорт вылета)
  arrival_airport_id: UUID | FK | NOT NULL (Аэропорт прибытия)
  departure_at: TIMESTAMP | NOT NULL (Текущее время вылета в UTC)
  arrival_at: TIMESTAMP | NOT NULL (Текущее время прибытия в UTC)
  status: ENUM(SCHEDULED, DELAYED, CANCELLED, DEPARTED, ARRIVED) | NOT NULL (Текущий статус)
  updated_at: TIMESTAMP | NOT NULL (Время последнего обновления)
}

Table(Booking, "Booking (Бронирование)") {
  id: UUID | PK (Идентификатор бронирования)
  --
  flight_id: UUID | FK | NOT NULL (Идентификатор рейса)
  trip_member_id: UUID | FK | NOT NULL (Участник поездки)
  booking_reference: STRING (Код бронирования)
  status: ENUM(BOOKED, CANCELLED, CHECKED_IN) | NOT NULL (Статус бронирования)
  baggage_count: INTEGER (Количество мест багажа)
  baggage_weight: DECIMAL (Вес багажа)
}

Table(Airport, "Airport (Аэропорт)") {
  id: UUID | PK (Идентификатор аэропорта)
  --
  iata_code: STRING | UNIQUE | NOT NULL (Код IATA)
  name: STRING | NOT NULL (Название)
  city: STRING | NOT NULL (Город)
  timezone: STRING | NOT NULL (Часовой пояс IANA, например Europe/Moscow)
}

Table(Airline, "Airline (Авиакомпания)") {
  id: UUID | PK (Идентификатор авиакомпании)
  --
  iata_code: STRING | UNIQUE | NOT NULL (Код IATA)
  name: STRING | NOT NULL (Название)
}

BankPlatform ||--o{ Trip : "идентифицирует создателя"
BankPlatform ||--o{ TripMember : "идентифицирует участника"

Trip ||--o{ TripMember : "включает"
Trip ||--o{ ItineraryItem : "содержит"
ItineraryItem ||--o| Flight : "может содержать"
Flight ||--o{ Booking : "имеет"
TripMember ||--o{ Booking : "имеет"
Airline ||--o{ Flight : "выполняет"
Airport ||--o{ Flight : "аэропорт вылета"
Airport ||--o{ Flight : "аэропорт прибытия"

note right of BankPlatform
  Внешняя система.
  Не создаём таблицу пользователей
  в БД авиасервиса.
  bank_user_id хранится только
  как внешний идентификатор.
end note

@enduml
```

## 3. Monitoring DB (Monitoring Service)
<img width="1649" height="573" alt="image" src="https://github.com/user-attachments/assets/845ec085-7831-44c4-bb41-eee0c098d2a9" />

```plantuml
@startuml
!define Table(name,desc) entity name as "desc" << (T,#FFAAAA) >>

title Monitoring DB (Monitoring Service)

Table(MonitoredFlight, "MonitoredFlight (Рейс под слежением)") {
  flight_id: UUID | PK (ID рейса в Trip DB. Ссылка на Trip DB, FK нет)
  --
  trip_id: UUID | NOT NULL (Поездка: нужна для события. Ссылка на Trip DB, FK нет)
  itinerary_item_id: UUID | NOT NULL (Элемент маршрута: нужен для события. Ссылка на Trip DB, FK нет)
  airline_iata_code: STRING | NOT NULL (Код авиакомпании для запроса к источнику)
  flight_number: STRING | NOT NULL (Номер рейса для запроса к источнику)
  departure_airport_iata: STRING | NOT NULL (Аэропорт вылета для запроса к источнику)
  arrival_airport_iata: STRING | NOT NULL (Аэропорт прибытия для запроса к источнику)
  departure_at: TIMESTAMP | NOT NULL (Последнее известное время вылета в UTC)
  arrival_at: TIMESTAMP | NOT NULL (Последнее известное время прибытия в UTC)
  status: ENUM(SCHEDULED, DELAYED, CANCELLED, DEPARTED, ARRIVED) | NOT NULL (Последний известный статус: с ним сравниваем новые данные)
  allowed_delay_minutes: INTEGER | NOT NULL (Допустимая задержка с учётом пересадки: передаёт Core Trip)
  last_checked_at: TIMESTAMP | NULL (Время последней успешной проверки: по нему определяем устаревание данных)
}

Table(FlightSource, "FlightSource (Источник данных о рейсах)") {
  id: UUID | PK (Идентификатор источника)
  --
  name: STRING | NOT NULL (Название источника)
  source_type: ENUM(API, EMAIL, OTHER) | NOT NULL (Тип источника)
  priority: INTEGER | NOT NULL (Приоритет источника)
}

Table(FlightStatusHistory, "FlightStatusHistory (История изменений рейса)") {
  id: UUID | PK (Идентификатор записи)
  --
  flight_id: UUID | FK | NOT NULL (Рейс под слежением)
  source_id: UUID | FK | NOT NULL (Источник данных)
  source_event_id: STRING (Идентификатор события у источника)
  status: STRING | NOT NULL (Полученный статус)
  departure_at: TIMESTAMP (Время вылета из сообщения источника)
  arrival_at: TIMESTAMP (Время прибытия из сообщения источника)
  source_updated_at: TIMESTAMP (Время обновления у источника)
  received_at: TIMESTAMP | NOT NULL (Время получения системой)
}

MonitoredFlight ||--o{ FlightStatusHistory : "имеет историю"
FlightSource ||--o{ FlightStatusHistory : "передает данные"

@enduml
```

## 4. Alternatives DB (Alternatives Service)
<img width="918" height="393" alt="image" src="https://github.com/user-attachments/assets/b7fcca7a-fa7e-45d7-a412-9caceacbd7f4" />

```plantuml
@startuml
!define Table(name,desc) entity name as "desc" << (T,#FFAAAA) >>

title Alternatives DB (Alternatives Service)

Table(Alternative, "Alternative (Альтернатива)") {
  id: UUID | PK (Идентификатор альтернативы)
  --
  itinerary_item_id: UUID | NOT NULL (Элемент маршрута, для которого найдена альтернатива. Ссылка на Trip DB, FK нет)
  type: ENUM(FLIGHT, ACTIVITY) | NOT NULL (Тип альтернативы)
  details: JSON | NOT NULL (Детали предложения: рейс, время, цена или активность)
  status: ENUM(PROPOSED, SELECTED, REJECTED, EXPIRED) | NOT NULL (Статус предложения)
  created_at: TIMESTAMP | NOT NULL (Дата создания)
}

Table(PlanChange, "PlanChange (Смена рейса)") {
  id: UUID | PK (Идентификатор операции бронирования)
  --
  alternative_id: UUID | FK | NOT NULL (Выбранная альтернатива)
  itinerary_item_id: UUID | NOT NULL (Какой элемент маршрута заменяем. Ссылка на Trip DB, FK нет)
  idempotency_key: UUID | UNIQUE | NOT NULL (Ключ из заголовка запроса: повторный клик не создаёт вторую бронь)
  status: ENUM(PENDING, CONFIRMED, FAILED) | NOT NULL | DEFAULT 'PENDING' (Статус бронирования)
}

Alternative ||--o{ PlanChange : "выбирается через"

@enduml
```

## 5. Notification DB (Notification Service)
<img width="952" height="426" alt="image" src="https://github.com/user-attachments/assets/0b5e3819-ab91-4912-a62d-e4a5ad628fc9" />

```plantuml
@startuml
!define Table(name,desc) entity name as "desc" << (T,#FFAAAA) >>

title Notification DB (Notification Service)

' Внешняя сущность. Не является таблицей БД авиасервиса.
entity "Bank Platform\n(внешняя система)" as BankPlatform <<external>> {
  bank_user_id: STRING | PK (Идентификатор клиента в банковской платформе)
}

Table(Notification, "Notification (Уведомление)") {
  id: UUID | PK (Идентификатор уведомления)
  --
  recipient_bank_user_id: STRING | NOT NULL (ID получателя во внешней банковской платформе)
  trip_id: UUID | NOT NULL (Поездка, к которой относится уведомление. Ссылка на Trip DB, FK нет)
  type: ENUM(FLIGHT_DELAYED, FLIGHT_CANCELLED, ALTERNATIVES_READY, TRIP_PLAN_UPDATED) | NOT NULL (Тип уведомления)
  message: STRING | NOT NULL (Текст уведомления)
  source_event_id: STRING | NOT NULL (Идентификатор исходного события)
  created_at: TIMESTAMP | NOT NULL (Дата создания)
  is_read: BOOLEAN | NOT NULL | DEFAULT false (Прочитано ли уведомление)
  --
  UNIQUE(recipient_bank_user_id, source_event_id) (Защита от повторных уведомлений одному получателю)
}

BankPlatform ||--o{ Notification : "идентифицирует получателя"

@enduml
```

