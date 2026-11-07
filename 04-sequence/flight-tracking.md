<img width="1672" height="988" alt="image" src="https://github.com/user-attachments/assets/2f639abf-20d6-4bc4-a430-2d632b3b46bf" />


```plantuml
@startuml
title Отслеживание статуса рейса
hide footbox
autonumber

participant "Monitoring\nService" as MON
participant "Flight Status API" as FS #E8E8E8
participant "Резервный провайдер\nстатусов" as RP #E8E8E8
queue "Message Broker\n(Kafka)" as KF
participant "Core Trip\nService" as CT
participant "Notification\nService" as NT

note over MON : Запуск по расписанию\n(опрос всех активных рейсов)

MON -> FS : Запрашивает статус рейса

alt Провайдер не отвечает
  FS --x MON : Таймаут
  MON -> MON : Повторяет запрос, затем ждёт следующего цикла
  note right of MON : Событие не публикуется,\nпользователь видит последний сохраненный статус

else Статус получен
  FS --> MON : Статус рейса
  MON -> MON : Проверяет актуальность данных

  opt Данные пришли с большой задержкой
    MON -> RP : Запрашивает подтверждение статуса
    RP --> MON : Статус рейса

    break Статус не подтвердился
      MON -> MON : Не публикует событие, перепроверит в следующем цикле
    end
  end

  alt Статус без изменений
    note over MON : Проверка завершена

  else Рейс задержан или отменён
    MON ->> KF : Публикует TRIP_DISRUPTION

    par Каждый сервис читает событие независимо
      KF ->> CT : TRIP_DISRUPTION
      CT -> CT : Помечает рейс и связанные сегменты\nкак «Требующие внимания»
    else
      KF ->> NT : TRIP_DISRUPTION
      ref over NT : Отправка уведомления пользователям
    end
  end
end
@enduml
```
