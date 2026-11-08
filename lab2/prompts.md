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

### Prompt 4: misspelling check
* **Request:** "check if there are any misspelling errors in my spec and defense".
* **Result:** " I have thoroughly audited your spec.md and DEFENSE.md texts for misspellings, typos, and formatting issues. Here is a list of the small typos identified along with the exact corrections to apply:

1. Typos in spec.md

REQ-02 (EARS Requirements):
Typo: confired
Fix: Change to confirmed ("check if current confirmed + pending bookings...").

REQ-03 (EARS Requirements):
Typo: succesful / initialise
Fix: Change to successful and initialize (or keep standard British spelling if used consistently throughout).

NFR-01 (NFR Section):
Typo: ui / IOS
Fix: Capitalize UI and change IOS to iOS.

NFR-03 (NFR Section):
Typo: pauload
Fix: Change to payload ("...delivery of HTTP response payload...").

Gherkin Scenario 1 (User Stories):
Typo: my my card
Fix: Remove duplicate word: my card.
Typo: succesfull
Fix: Change to successful.

Gherkin Scenario 3 (User Stories):
Typo: recieving
Fix: Change to receiving.

Gherkin Scenario 4 (User Stories):
Typo: succesfully
Fix: Change to successfully.

Traceability Matrix (Section 4):
Typo: Sсenario 1 under REQ-03
Fix: The letter "с" in Sсenario 1 was written using a Cyrillic character instead of Latin "c". Change to standard Latin Scenario 1.
Typo: Missing comma under REQ-05 in UC_Book UC_Cancel
Fix: Add a comma: UC_Book, UC_Cancel.

Audit / Top 3 Mismatches (Section 6):
Typo: mamagement
Fix: Change to management.
Typo: Restructed
Fix: Change to Restructured.

2. Status Terminology Consistency

Make sure your status strings match 100% identically across all sections:
If your ER diagram and matrix declare status values as pending, confirmed, cancelled, ensure that REQ-01, REQ-03, and Gherkin Scenario 1 use "pending" (or "pending_payment", depending on which precise string you picked, but strictly uniform across the entire document).

Summary Status
With these minor typos fixed, your spec.md and DEFENSE.md documents are completely polished, clean, and ready for submit/push!"