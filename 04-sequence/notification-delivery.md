@startuml
title Отправка уведомления пользователям
hide footbox
autonumber

queue "Message Broker\n(Kafka)" as KF
participant "Notification\nService" as NT
database "Notification DB" as DB
participant "Push Provider" as PP #E8E8E8
actor "Пользователь" as U

KF ->> NT : TRIP_DISRUPTION

NT -> NT : Определяет получателей и формирует текст

NT -> DB : Сохраняет в «Центр уведомлений»

NT -> PP : Отправляет push

alt Пользователь принял push
  PP ->> U : Показывает push на телефоне\n(сразу или когда пользователь выйдет в сеть)
else Пользователь офлайн
  NT -> NT : Повторяет отправку позже
  note right of NT : Уведомление не потеряно:\nсохранено в «Центре уведомлений»
end

== Пользователь открывает приложение ==

U -> NT : Запрашивает уведомления
NT -> DB : Читает непрочитанные и помечает прочитанными
DB --> NT : Уведомления
NT --> U : Список уведомлений
@enduml
