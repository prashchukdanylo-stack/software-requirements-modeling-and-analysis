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
- C-05 Booking Service [Backend]: Manages the booking lifecycle: checks for available slots (`hasFreeSlots`), creates requests with a `pending_payment` status, and handles cancellations and slot blocking.
- C-06 Payment Service [Backend]: Handles `createInvoice`/`cancelInvoice` calls, processes and verifies webhooks (ECDSA SHA-256), and ensures idempotency.
- C-07 BookingTimeoutWorker [Backend]: A background service for checking and canceling expired bookings with the status `pending_payment` (15-minute limit).
- C-08 Notification Adapter [Backend]: Handles sending notifications (`sendMessage()`) to the student in cases of success, errors/timeouts, or cancellations.
- C-09 Data Access Layer [Data]: A layer comprising repositories, transactions, row-level locking (`FOR UPDATE`), and encryption of personally identifiable information (PII) using AES-256.
- C-10 PostgreSQL [Data]: A relational store for ER entities that includes ENUM constraints and supports background integrity checks (Req-6).

