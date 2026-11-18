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
- C-02 API Gateway [Backend]: A single entry point for client requests; performs token validation, manages request queuing (up to 50 RPS, with a buffer of up to 200 requests before returning a "too many requests" error), and supports TLS 1.3.
- C-03 Identity Service [Backend]: Handles registration, login, and profile management, as well as the issuance and verification of tokens.
- C-04 Schedule Service [Backend]: Provides a schedule with filters by style and instructor; manages studios, styles, instructors, and classes.
- C-05 Booking Service [Backend]: Manages the booking lifecycle: checks for available slots (`hasFreeSlots`), creates requests with a 'pending_payment' status, and handles cancellations and slot blocking.
- C-06 Payment Service [Backend]: Handles 'createInvoice'/'cancelInvoice' calls, processes and verifies webhooks (ECDSA SHA-256), and ensures idempotency.
- C-07 BookingTimeoutWorker [Backend]: A background service for checking and canceling expired bookings with the status 'pending_payment' (15-minute limit).
- C-08 Notification Adapter [Backend]: Handles sending notifications ('sendMessage()') to the student in cases of success, errors/timeouts, or cancellations.
- C-09 Data Access Layer [Data]: A layer comprising repositories, transactions, row-level locking ('FOR UPDATE'), and encryption of personally identifiable information (PII) using AES-256.
- C-10 PostgreSQL [Data]: A relational store for ER entities that includes ENUM constraints and supports background integrity checks (Req-6).

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

## 6. Dependencies

Direction is from consumer to provider:
* Web UI depends on API Gateway and on Monobank API(payment page).
* API Gateway depends on Identity, Schedule, Booking and Payment.
* Booking and Payment depend on each other.
* BookingTimeoutWorker depends on Booking.
* Booking and Payment depend on Notification Adapter.
* Payment depends on Monobank API, and Monobank API calls Payment through the webhook.
* Identity, Schedule, Booking, Payment, BookingTImeoutWorker and Notification Adapter depend on Data Access Layer, which depends on PostgreSQL.


