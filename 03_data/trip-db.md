@startuml
!define Table(name,desc) entity name as "desc" << (T,#FFAAAA) >>
title Trip DB (Core Trip Service)

Table(Trip, "Trip (Поездки)") {
  trip_id: UUID PK 
  --
  title: VARCHAR(100) NOT NULL (Название для пользователя)
  ends_at: TIMESTAMPTZ NOT NULL (Окончание поездки UTC: по нему Passport Vault считает срок удаления паспортов)
}

Table(TripMember, "TripMember (Участники поездки)") {
  trip_id: UUID PK FK (В какой поездке состоит участник)
  email: VARCHAR(255) PK (Кого пригласили; вместе с trip_id не дает пригласить человека дважды)
  --
  user_id: UUID NULL (проверяем права по JWT; пусто, пока приглашение не принято)
  role: ENUM(ORGANIZER, PARTICIPANT) NOT NULL (Роль пользователя)
}

Table(Segment, "Segment (Сегменты плана)") {
  segment_id: UUID PK (Какой именно рейс, отель, активность)
  --
  trip_id: UUID FK NOT NULL (К какой поездке относится сегмент)
  type: ENUM(FLIGHT, HOTEL, TRANSFER, ACTIVITY) NOT NULL (Тип сегмента)
  title: VARCHAR(100) NOT NULL (Название для отображения, например SVO - IST, TK416)
  starts_at: TIMESTAMPTZ NOT NULL (Время начала UTC: для расчета пересадок и пересечений)
  time_zone: VARCHAR(64) NOT NULL (Таймзона места начала в формате IANA)
  status: ENUM(CONFIRMED, DELAYED, CANCELLED, NEEDS_ATTENTION) NOT NULL DEFAULT 'CONFIRMED' (Статус меняется в случае изменения)
  booking_id: VARCHAR(64) NOT NULL (Ссылка на бронь в Existing Service: нужна для перебронирования)
  baggage_included: BOOLEAN NOT NULL (включен ли багаж в текущий билет)
  status_updated_at: TIMESTAMPTZ NOT NULL (когда обновили статус, чтобы показать время последнего обновления)
}

Table(PlanChange, "PlanChange (Смена рейса)") {
  change_id: UUID PK (Идентификатор операции перебронирования)
  --
  segment_id: UUID FK NOT NULL (Какой сегмент заменяем, поездка определяется через него)
  alternative_id: UUID NOT NULL (Какую альтернативу выбрал Организатор)
  idempotency_key: UUID NOT NULL UNIQUE (Ключ из заголовка PATCH: повторный клик не создаёт вторую бронь)
  status: ENUM(PENDING, CONFIRMED, FAILED) NOT NULL DEFAULT 'PENDING' (Статус перебронирования)
}

Trip ||--o{ TripMember : "1:M"
Trip ||--o{ Segment : "1:M"
Segment ||--o{ PlanChange : "1:M"

@enduml
