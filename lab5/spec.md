# Architecture model

**System:** Dance Studio CRM.
**Notation:** UML Component diagram (PlantUML - architecture.puml), render -- architecture.html.
**Context and model chain:** 
- Lab1(ER): data model, used: Student, Instructor, DanceStyle, Hall, Lesson, Booking, Administrator, Payment; relations and keys.
- Lab2: Requirements: actors from 1.1, out-of-scope from 1.2, EARS Req-1..6, NFR-01..07, Gherkin G1-G4, traceability matrix; functions, restrictions, borders.
- Lab3: Interaction: SD 1.1 (booking and payment), 1.2 (no available slots), 1.3A/1.3B (timeout/reject), 1.4 (canceling); participants, messages - dependencies, interfaces.
- Lab4: Process: Activity diagram (Student, CRM-system, Monobank API, BookingTimeoutWorker), steps 1-7, signals 1-9; responsibilty division, asynchronous events.

## 1. System border
Inside of the border -- Web UI, Backend(API Gateway, Identity, Schedule, Booking, Payment, BookingTimeoutWorker, NotificationAdapter) and Data (Data Access Layer, PostgreSQL). This is relevant to NFR-07 (Node.js 20+, PostgreSQL 15+, Docker). Out of border only Monobank API.
Four main connections cross the system boundary: IMonobank (Payment → Monobank), IWebhook (Monobank → Payment—bypassing the Gateway directly, in accordance with Lab3 1.3B), IPaymentPage (redirecting the Web UI to the payment page), and user interaction with the Web UI. There is no direct access to the database from the external environment.

## 2. Modules
- C-01 Web UI (SPA) [Presentation]: An interface for users, students, instructors, and administrators; provides Ukrainian localization and redirects to the payment page.
Traces to: Lab2 1.1, NFR-01, NFR-06, G1-G4; Lab3 1.1 steps 1, 7, 8, 1.2 step 6, 1.4 step 1; Lab4 steps 1,5. 
- C-02 API Gateway [Backend]: A single entry point for client requests; performs token validation, manages request queuing (up to 50 RPS, with a buffer of up to 200 requests before returning a "too many requests" error), and supports TLS 1.3.
Traces to: NFR-02, NFR-03, NFR-05; 'validateToken' in Lab3 1.1, 1.2, 1.4.
- C-03 Identity Service [Backend]: Handles registration, login, and profile management, as well as the issuance and verification of tokens.
Traces to: Lab2 1.1 (Student goal), "student is registered" precondition in G1–G4; Lab1 Student, Administrator.
- C-04 Schedule Service [Backend]: Provides a schedule with filters by style and instructor; manages studios, styles, instructors, and classes.
Traces to: Lab2 1.1, NFR-03 (schedule queries); Lab1 Lesson, Hall, Instructor, DanceStyle.
- C-05 Booking Service [Backend]: Manages the booking lifecycle: checks for available slots (`hasFreeSlots`), creates requests with a 'pending_payment' status, and handles cancellations and slot blocking.
Traces to: Req-1, 2, 4, 5; G1–G4; Lab3 1.1 steps 3–4, 7, 1.2, 1.3A, 1.3B, 1.4; Lab4 steps 2, 3, 7; Lab1 Booking, Lesson, Hall.
- C-06 Payment Service [Backend]: Handles 'createInvoice'/'cancelInvoice' calls, processes and verifies webhooks (ECDSA SHA-256), and ensures idempotency.
Traces to: Req-3, 5; NFR-05; G1; Lab3 1.1 steps 5–6, 9–10, 1.3B, 1.4; Lab4 steps 4, 6A, 7; Lab1 Payment.
- C-07 BookingTimeoutWorker [Backend]: A background service for checking and canceling expired bookings with the status 'pending_payment' (15-minute limit).
Traces to: Req-5, NFR-04, G3; Lab3 1.3A; Lab4 step 6B, 7, signal 7; Lab1 Booking.booked_at.
- C-08 Notification Adapter [Backend]: Handles sending notifications ('sendMessage()') to the student in cases of success, errors/timeouts, or cancellations.
Traces to: notification lines in G1, G3, G4; 'sendMessage()' in Lab3 1.1, 1.3, 1.4; Lab4 signal 9; Lab1 Student.email and Student.phone as contact data.
- C-09 Data Access Layer [Data]: A layer comprising repositories, transactions, row-level locking ('FOR UPDATE'), and encryption of personally identifiable information (PII) using AES-256.
Traces to: Req-1 (ENUM), Req-2, Req-4; NFR-03, NFR-05; "Database" in all Lab3 scenarios; Lab4 step 3; all Lab1 entities. This component is my addition, Lab3 shows services calling "Database" directly.
- C-10 PostgreSQL [Data]: A relational store for ER entities that includes ENUM constraints and supports background integrity checks (Req-6).
Traces to: Req-1, Req-6; NFR-07; all Lab1 entities.

## 3. External integrations
X-01 Monobank API: An external system operating via the HTTPS/REST protocol with support for signed webhooks. It provides the IMonobank (createInvoice, cancelInvoice) and IPaymentPage (pageUrl) interfaces, and consumes IWebhook. The interaction flow is as follows: Payment sends requests to Monobank, Monobank sends a webhook back to Payment, and the student is redirected to the payment page.

## 4. Interfaces

* **IRestAPI:** provided by API Gateway, required by Web UI. HTTP requests from the interface: booking, cancelling, timetable, profile, admin actions (Lab3 1.1 step 1, 1.2 step 1, 1.4 step 1).
* **IAuth:** provided by Identity Service, required by API Gateway. 'validateToken' (Lab3 1.1 step 2). Also registration, login and managing the personal profile (Lab2 §1.1, goal of Student). Lab3 does not describe them.
* **ISchedule:** provided by Schedule Service, required by API Gateway. Viewing the timetable with filter by dance style and instructor, adding and changing halls, instructors, dance styles and lessons (Lab2 §1.1 goals only).
* **IBooking:** provided by Booking Service, required by API Gateway and BookingTimeoutWorker. 'cancelBookingByUser(studentId, bookingId)' (Lab3 1.4 step 2). 'cancelExpiredBookings' (Lab4 signal 7, Lab3 1.3A "invokes BookingService to cancel"). Creating a booking (Lab3 1.1 step 4 "creates entry"). Own bookings of Student and list of applied students for a lesson (Lab2 §1.1 goals).
* **IBookingState:** provided by Booking Service, required by Payment Service. Confirming and cancelling the booking after a webhook (Lab3 1.1 step 10, 1.3B).
* **IPayment:** provided by Payment Service, required by API Gateway and Booking Service. 'cancelInvoice' (Lab3 1.3, 1.4). Initialising the invoice (Lab3 1.1 step 5). Viewing payments and late webhooks for manual refund (Lab2 §1.1 "financial issues", Lab3 1.1 step 10).
* **IWebhook:** provided by Payment Service, required by Monobank API. Signed webhook, answer HTTP 200 or 400 (Lab3 1.1 step 10, 1.3B; Lab4 step 7).
* **INotify:** provided by Notification Adapter, required by Booking Service and Payment Service. 'sendMessage' (Lab3 1.1, 1.3A, 1.3B, 1.4; Lab4 signal 9).
* **IRepository:** provided by Data Access Layer, required by Identity, Schedule, Booking, Payment, BookingTimeoutWorker and Notification Adapter. 'countActiveBookings' (Lab3 1.1 step 3). Reading and writing all entities in transactions, searching for expired pending bookings (Lab3 1.3A, the worker polls the database).
* **IMonobank:** provided by Monobank API, required by Payment Service. 'createInvoice', 'cancelInvoice' (Lab3 1.1 step 6, 1.3).
* **IPaymentPage:** provided by Monobank API, required by Web UI. Redirect to 'pageUrl' (Lab3 1.1 steps 7–8; Lab4 step 5).

## 5. Dependencies

Direction is from consumer to provider:
* Web UI depends on API Gateway and on Monobank API(payment page).
* API Gateway depends on Identity, Schedule, Booking and Payment.
* Booking and Payment depend on each other.
* BookingTimeoutWorker depends on Booking.
* Booking and Payment depend on Notification Adapter.
* Payment depends on Monobank API, and Monobank API calls Payment through the webhook.
* Identity, Schedule, Booking, Payment, BookingTImeoutWorker and Notification Adapter depend on Data Access Layer, which depends on PostgreSQL.

## 6. Audit

### 6.1 Traceability of components:
All components are traced to Lab1-Lab4 in section 2, each in the line "Traces to". Requirement coverage:
* **Req-1:** Booking Service, Data Access Layer, PostgreSQL.
* **Req-2:** Booking Service, Data Access Layer.
* **Req-3:** Payment Service, Monobank API.
* **Req-4:** Booking Service, Data Access Layer (locking).
* **Req-5:** Payment Service (declined payment), BookingTimeoutWorker (timeout), Booking Service.
* **Req-6:** PostgreSQL maintenance. There is no application component for it, as Lab2 5 paragraph says.

Fully traced chain (Req, Gherkin, Lab3, Lab4, component): the booking and payment scenario (C-01, C-02, C-05, C-06, C-07, X-01). Partly traced: Identity, Schedule, Notification Adapter, Data Access Layer.

## 6.2 Traceability of dependencies:
* Web UI to API Gateway: Lab3 1.1.1, 1.2.1, 1.4.1; Lab4 step 1.
* Web UI to Monobank payment page: Lab3 1.1.7–8; Lab4 step 5.
* API Gateway to Identity ('validateToken'): Lab3 1.1.2, 1.2.2, 1.4.2.
* API Gateway to Booking: Lab3 1.1.2, 1.2.2, 1.4.2.
* API Gateway to Schedule: only Lab2 §1.1 (D-1).
* API Gateway to Payment (admin): only Lab2 §1.1 and the log of late webhooks (D-1).
* Booking to Payment: Lab3 1.1.5, 1.4; Lab4 step 4.
* Payment to Booking ('IBookingState'): Lab3 1.1.10, 1.3B.
* BookingTimeoutWorker to Booking: Lab3 1.3A.3.
* Booking and Payment to Notification Adapter: Lab3 1.1.10, 1.3A, 1.3B, 1.4; Lab4 signal 9.
* Payment to Monobank API: Lab3 1.1.6, 1.3; Lab4 step 4.
* Monobank API to Payment (webhook, bypassing API Gateway): Lab3 1.1.9–10, 1.3B; Lab4 steps 6A, 7.
* Services and worker to Data Access Layer to PostgreSQL: "Database" in Lab3; Lab3 1.3A.1 (worker polls the database).

### 6.3 Border and interface check:
* **Border:** 10 internal components and 1 external system, crossings are listed in section 1.
* **Provided and required:** every one of 11 interfaces has one provider and at least one consumer, there are no dangling interfaces.
* **Layers:** Web UI never calls the database, access to the database is only through Data Access Layer.
* **Webhook:** 'IWebhook' belongs to Payment Service, not to API Gateway. It matches Lab3 1.3B and NFR-05 (signature is checked in Payment Service).

### 6.4 Top-3 mismatches:
#### 1. Bypassing the API Gateway for External Webhooks (Webhook Bypass)
The Mismatch: The current architecture uses the API Gateway as the primary entry point for client requests. However, Monobank webhooks introduce a separate external entry point directly to the Payment Service, bypassing the Gateway. This is justified by the requirement for rapid verification of the cryptographic ECDSA signature (NFR-05). Nevertheless, it creates a second entry point into the system.
#### 2. Cyclic Dependency and Two-Way Coordination Between Booking and Payment Services
The Mismatch: According to the business logic of the scenarios, the Booking Service invokes the Payment Service to initiate an invoice; subsequently, while processing a webhook, the Payment Service must call back the Booking Service to update the booking status (IBookingState). This creates a direct cyclic dependency between the services, increasing coupling and requiring careful interface design to maintain component independence and testability.
#### 3. Undefined Notification Delivery Mechanism
The Notification Adapter is described as an internal component responsible for sending messages via sendMessage(). However, the implementation of the notification delivery mechanism is not specified in the previous laboratory works. In particular, it is unclear whether messages are sent directly by the adapter or through an external SMS/Email provider.
This creates an architectural ambiguity regarding the implementation and responsibility for notification delivery. The current component model preserves the Notification Adapter as an internal component without introducing an undocumented external dependency.