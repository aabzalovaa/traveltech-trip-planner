<img width="772" height="185" alt="image" src="https://github.com/user-attachments/assets/6860709a-4d21-4f7d-9f56-a6a54153bf55" />

```plantuml
@startuml
!define Table(name,desc) entity name as "desc" << (T,#FFAAAA) >>
title Passport Vault DB (Passport Vault Service)

Table(PassportRecord, "PassportRecord (Паспортные данные)") {
  user_id: UUID PK (Чьи паспортные данные)
  trip_id: UUID PK (Для какой поездки собраны: у каждой поездки свой срок хранения)
  --
  citizenship_code: CHAR NOT NULL (Alternatives проверяет визы, не видя сам паспорт)
  encrypted_data: BYTEA NOT NULL (Паспортные данные только в зашифрованном виде)
  delete_after: TIMESTAMPTZ NOT NULL (Срок автоудаления: по нему задача очистки удаляет данные)
}

@enduml
```
