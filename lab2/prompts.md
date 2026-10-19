# AI Interaction Log

This logs all ai prompts that I will write during lab2.
---

### Prompt 1: new Actors

* **Request:** "Wait, there is no admin and payment service in the original data model, so I need to write down the differences in audit, or to change the data model?"

* **Result** "Keep ER Model As-Is (Recommended Path): Do not modify the original ER diagram from Lab 1. Keep it focused on Student, Instructor, Lesson, Hall, DanceStyle, and Booking.

Introduce Admin in Requirements: In Lab 2 (spec.md), define the Administrator, Monobank API actors and their goals since business logic naturally requires management roles."

### Prompt 2: useCase diagram
* **Request:** " @startuml
left to right direction
actor "User" as U
actor "Anonymous user" as A
actor "Authenticated user" as AU
A -|> U
AU -|> U
rectangle "Billing" {
  usecase "Authenticate" as Auth
  usecase "Subscribe" as Sub
  usecase "Apply promo" as Promo
  usecase "Trial" as Trial
  usecase "Change payment method" as Change
  usecase "Check payment plan" as Check
}
rectangle "Payment processor" {
 usecase "Cancel subscription" as Cancel
}
U --> Auth
A --> Sub
AU --> Change
AU --> Check
AU --> Cancel
Change ..> Auth : <<include>>
Promo ..> Sub : <<extend>>
Trial ..> Sub : <<extend>>
@enduml

make a diagram for my system in similar way. "
* **Result:** "@startuml
' Налаштування стилю діаграми
skinparam actorStyle stickman
skinparam packageStyle rectangle
left to right direction
title Use Case Diagram — Dance Studio System (Billing Scope)
' Визначення акторів (Люди та Зовнішня система)
actor "User" as U << abstract >>
actor "Student" as S
actor "Administrator" as Admin
actor "Instructor" as Instructor
actor "Monobank API" as MonoPay << External System >>
' Ієрархія акторів
S -|> U
Admin -|> U
Instructor -|> U
' Межа внутрішньої системи (Dance Studio System)
rectangle "Dance Studio System" {
  usecase "Переглянути та відфільтрувати\nрозклад занять" as UC_View
  usecase "Забронювати місце\nна заняття" as UC_Book
  usecase "Ініціювати оплату\nчерез Monobank API" as UC_Pay
  usecase "Скасувати бронювання" as UC_Cancel
  usecase "Застосувати промокод /\nзнижку" as UC_Promo
  usecase "Переглянути список\nстудентів на заняття" as UC_List
  usecase "Керувати розкладом,\nзалами та викладачами" as UC_Manage
}
' Межа зовнішньої системи (Платіжний процесор)
rectangle "Monobank (External Processor)" {
  usecase "Обробити сповіщення\nпро платіж (Webhook)" as UC_Webhook
}
' Зв'язки Акторів із Прецедентами (Use Cases)
U --> UC_View
S --> UC_Book
S --> UC_Cancel
Instructor --> UC_List
Admin --> UC_Manage
Admin --> UC_Cancel
MonoPay --> UC_Webhook
' Зв'язки між Прецедентами (include / extend)
' Ініціація оплати є ОБОВ'ЯЗКОВИМ кроком при бронюванні
UC_Book ..> UC_Pay : <<include>>
' Зв'язок асинхронної взаємодії: Платіж веде до Webhook'а
UC_Pay --> MonoPay
MonoPay ..> UC_Webhook : <<include>>
' Застосування промокоду ОПЦІОНАЛЬНО розширює процес оплати
UC_Promo ..> UC_Pay : <<extend>>

@enduml" 

### Prompt 3: fixing generated diagram
* **Request:** "My dancing studio system does not include any sales or promocodes, it is better to visualise that booking cancelling extends a feature of managing bookings in profile."
* **Result:** "@startuml
' Налаштування стилю діаграми
skinparam actorStyle stickman
skinparam packageStyle rectangle
left to right direction
title Use Case Diagram — Dance Studio System
' Визначення акторів (Люди та Зовнішня система)
actor "User" as U << abstract >>
actor "Student" as S
actor "Administrator" as Admin
actor "Instructor" as Instructor
actor "Monobank API" as MonoPay << External System >>
' Ієрархія акторів
S -|> U
Admin -|> U
Instructor -|> U
' Межа внутрішньої системи (Dance Studio System)
rectangle "Dance Studio System" {
  usecase "Переглянути та відфільтрувати\nрозклад занять" as UC_View
  usecase "Забронювати місце\nна заняття" as UC_Book
  usecase "Ініціювати оплату\nчерез Monobank API" as UC_Pay
  usecase "Скасувати бронювання" as UC_Cancel
  usecase "Переглянути список\nстудентів на заняття" as UC_List
  usecase "Керувати розкладом,\nзалами та викладачами" as UC_Manage
}
' Межа зовнішньої системи (Платіжний процесор)
rectangle "Monobank (External Processor)" {
  usecase "Обробити сповіщення\nпро платіж (Webhook)" as UC_Webhook
}
' Зв'язки Акторів із Прецедентами (Use Cases)
U --> UC_View
S --> UC_Book
S --> UC_Cancel
Instructor --> UC_List
Admin --> UC_Manage
Admin --> UC_Cancel
MonoPay --> UC_Webhook
' Зв'язки між Прецедентами (include / extend)
' 1. Ініціація оплати є ОБОВ'ЯЗКОВИМ кроком при бронюванні
UC_Book ..> UC_Pay : <<include>>
' 2. Зв'язок асинхронної взаємодії з банком
UC_Pay --> MonoPay
MonoPay ..> UC_Webhook : <<include>>
' 3. Скасування бронювання є ОПЦІОНАЛЬНИМ розширенням (виконується за потреби)
UC_Cancel .> UC_Book : <<extend>>
@enduml"