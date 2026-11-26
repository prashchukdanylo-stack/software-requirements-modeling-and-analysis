# AI Interaction Log

This logs all ai prompts that I will write during lab5.
---

### Prompt 1: component diagram syntax

* **Request:** "розкажи, який синтаксис треба використовувати при побудові діаграми компонентів"

* **Result:** "Для побудови діаграми компонентів у PlantUML використовується стандартний і дуже інтуїтивний синтаксис. Оскільки ви вже працюєте з архітектурою вашої CRM-системи, ось ключові конструкції, які формують правильну діаграму:
1. Оголошення компонентів (component)

Компоненти створюються за допомогою ключового слова component або просто в квадратних дужках []:
Фрагмент коду

component "API Gateway" as Api
' Або короткий запис:
[API Gateway] as Api

2. Пакети та шарування (package)

Для групування компонентів за шарами (Presentation, Backend, Data) використовуються пакети:
Фрагмент коду

package "Backend Layer" {
  component "API Gateway" as Api
  component "Booking Service" as Book
}

3. Інтерфейси (()) та зв'язки з ними

PlantUML підтримує стандартну нотацію «колок» (Lollipop) для інтерфейсів.

    Оголошення інтерфейсу: () "Ім'яІнтерфейсу" as Name

    Надання інтерфейсу компонентом (provided): Component - Name

    Використання інтерфейсу компонентом (required): Consumer --> Name або Consumer ..> Name

Приклад:
Фрагмент коду

component "Booking Service" as Book
() "IBooking" as IBooking

' Booking Service надає цей інтерфейс
Book - IBooking

' API Gateway вимагає (використовує) його
[API Gateway] --> IBooking : uses

4. Напрямки та стрілки зв'язків

Для з'єднання елементів використовуються стрілки різного типу (-->, ..>, -, тошо):

    --> — прямий виклик / залежність.

    ..> — необов'язкова залежність або виклик через мережу/подію (наприклад, асинхронний webhook).

    Управління напрямком (якщо PlantUML плутає лінії, можна вказати ліру праворуч/вниз): -left-, -right-, -up-, -down-"

### Prompt 2: component diagram check

* **Request:** "@startuml
left to right direction

title Component Diagram -- Dance Studio CRM
actor "Student / Instructor / Administrator" as Actors

package "Dance Studio CRM (border of system)" {
package "Presentation" {
component "Web UI" as WebUI
}

package "Backend (Node.js 20+)" {
component "API Gateway" as Api
component "Identity Service" as Idn
component "Schedule Service" as Sch
component "Booking Service" as Book
component "Payment Service" as Pay
component "BookingTimeoutWorker" as Wrk
component "Notification Adapter" as Notif
}

package "Data" {
component "Data Access Layer" as DAL
database "PostgreSQL" as DB
}
}
package "External System (out of border)" {
component "Monobank API" as Mono
}

() "IRestAPI" as IRestAPI
() "IAuth" as IAuth
() "ISchedule" as ISchedule
() "IBooking" as IBooking
() "IBookingState" as IBookingState
() "IPayment" as IPayment
() "IWebhook" as IWebhook
() "INotify" as INotify
() "IRepository" as IRepository
() "IMonobank" as IMonobank
() "IPaymentPage" as IPaymentPage

Api - IRestAPI
Idn - IAuth
Sch - ISchedule
Book - IBooking
Book - IBookingState
Pay - IPayment
Pay - IWebhook
Notif - INotify
DAL - IRepository
Mono - IMonobank
Mono - IPaymentPage

Actors ..> WebUI
WebUI ..> IRestAPI : requires
WebUI ..> IPaymentPage : redirect(PageUrl)
Api ..> IAuth: validateToken
Api ..> ISchedule : requires
Api ..> IBooking : requires
Api ..> IPayment : requires (admin)
Book ..> IPayment : initialise invoice / cancelInvoice
Pay ..> IBookingState : confirm / cancel
Wrk ..> IBooking : cancelExpiredBookings
Book ..> INotify : sendMessage
Pay ..> INotify : sendMessage
Pay ..> IMonobank : createInvoice / cancelInvoice
Mono ..> IWebhook : signed webhook (повз Gateway)
Idn ..> IRepository
Sch ..> IRepository
Book ..> IRepository
Pay ..> IRepository
Wrk ..> IRepository
Notif ..> IRepository
DAL --> DB : SQL
@enduml перевір, чи відповідає діаграма усів потрібним умовам"

* **Result:** "Так, ця діаграма повністю відповідає всім умовам і вимогам архітектурного проєкту!

Вона технічно правильна, синтаксично вивірена й ідеально відображає всю ту логіку, яку ви заклали в текстову специфікацію та попередні лабораторні роботи.
Чому діаграма складена бездоганно:

    Чіткі межі системи (System Boundaries):

        Усі внутрішні шари (Presentation, Backend, Data) і 10 внутрішніх компонентів знаходяться всередині головного пакета системи.

        Monobank API чітко винесений назовні в окремий пакет External System (out of border).

    Повний набір модулів та шарування:

        Наявні всі сервіси (Api, Idn, Sch, Book, Pay, Notif), фоновий воркер (Wrk), шар доступу до даних (DAL) та реляційна база даних (PostgreSQL).

    Правильна робота з інтерфейсами (Lollipop notation):

        Усі 11 інтерфейсів (IRestAPI, IAuth, ISchedule, IBooking, IBookingState, IPayment, IWebhook, INotify, IRepository, IMonobank, IPaymentPage) оголошені й зв'язані за схемою provided/required.

    Реалізація складних архітектурних патернів:

        Обхід шлюзу для вебхуків: Рядок Mono ..> IWebhook : signed webhook (повз Gateway) чітко показує, що запити від Monobank надходять напряму в Payment Service, оминаючи API Gateway (як вимагалося у Лаб №3 та NFR-05).

        Ізоляція бази даних: Усі сервіси та воркер звертаються до бази виключно через DAL (Idn --> IRepository, DAL --> DB), що гарантує правильне шарування.

        Фоновий процес: Wrk (BookingTimeoutWorker) коректно взаємодіє з сервісом бронювання (IBooking)."

### Prompt 3: check for validity of mismatches

* **Request:** "### 6.4 Top-3 mismatches:
#### 1. Bypassing the API Gateway for External Webhooks (Webhook Bypass)
The Mismatch: The current architecture uses the API Gateway as the primary entry point for client requests. However, Monobank webhooks introduce a separate external entry point directly to the Payment Service, bypassing the Gateway. This is justified by the requirement for rapid verification of the cryptographic ECDSA signature (NFR-05). Nevertheless, it creates a second entry point into the system.
##### 2. Cyclic Dependency and Two-Way Coordination Between Booking and Payment Services
The Mismatch: According to the business logic of the scenarios, the Booking Service invokes the Payment Service to initiate an invoice; subsequently, while processing a webhook, the Payment Service must call back the Booking Service to update the booking status (IBookingState). This creates a direct cyclic dependency between the services, increasing coupling and requiring careful interface design to maintain component independence and testability.
#### 3. Incomplete Isolation of the Notification Adapter from External Services
The Notification Adapter is described as an internal component responsible for sending messages via sendMessage(). However, the external SMS/email delivery provider and the interface for interacting with it are not defined.
This creates an architectural ambiguity: it is unclear whether the Notification Adapter sends messages itself or delegates this responsibility to an external service. Як ти оціниш валідність цих 3 розбіжностей?".
* **Result:** "Це саме той рівень академічної та технічної зрілості, який потрібен для архітектурного звіту (Lab 5). Замість чіпляння до дрібниць на кшталт назв полів у базі, тут показані реальні системні виклики та архітектурні компроміси (trade-offs), з якими стикаються при переході від бізнес-процесів до мікросервісної/компонентної архітектури:

    Інтеграційний компроміс (Webhook bypass через безпеку).

    Проєктувальний виклик (Циклічні залежності між сервісами бронювання та оплати).

    Межі відповідальності (Невизначеність зовнішнього провайдера в адаптері нотифікацій).

Єдина дрібниця — поправ вирівнювання заголовочних тегів (у другому пункті випадково стоїть ##### 2. замість #### 2.)."