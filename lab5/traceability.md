# Traceability

**System:** Dance Studio CRM.
**Context and model chain:** 
- Lab1(ER): data model, used: Student, Instructor, DanceStyle, Hall, Lesson, Booking, Administrator, Payment; relations and keys.
- Lab2: Requirements: actors from 1.1, out-of-scope from 1.2, EARS Req-1..6, NFR-01..07, Gherkin G1-G4, traceability matrix; functions, restrictions, borders.
- Lab3: Interaction: SD 1.1 (booking and payment), 1.2 (no available slots), 1.3A/1.3B (timeout/reject), 1.4 (canceling); participants, messages - dependencies, interfaces.
- Lab4: Process: Activity diagram (Student, CRM-system, Monobank API, BookingTimeoutWorker), steps 1-7, signals 1-9; responsibilty division, asynchronous events.

## 1. Traceability of components
- C-01 Web UI (SPA) [Presentation]: Lab2 1.1, NFR-01, NFR-06, G1-G4; Lab3 1.1 steps 1, 7, 8, 1.2 step 6, 1.4 step 1; Lab4 steps 1, 5.
- C-02 API Gateway [Backend]: NFR-02, NFR-03, NFR-05; 'validateToken' in Lab3 1.1, 1.2, 1.4.
- C-03 Identity Service [Backend]: Lab2 1.1 (Student goal), "student is registered" precondition in G1–G4; Lab1 Student, Administrator.
- C-04 Schedule Service [Backend]: Lab2 1.1, NFR-03 (schedule queries); Lab1 Lesson, Hall, Instructor, DanceStyle.
- C-05 Booking Service [Backend]: Req-1, 2, 4, 5; G1–G4; Lab3 1.1 steps 3–4, 7, 1.2, 1.3A, 1.3B, 1.4; Lab4 steps 2, 3, 7; Lab1 Booking, Lesson, Hall.
- C-06 Payment Service [Backend]: Req-3, 5; NFR-05; G1; Lab3 1.1 steps 5–6, 9–10, 1.3B, 1.4; Lab4 steps 4, 6A, 7; Lab1 Payment.
- C-07 BookingTimeoutWorker [Backend]: Req-5, NFR-04, G3; Lab3 1.3A; Lab4 step 6B, 7, signal 7; Lab1 Booking.booked_at.
- C-08 Notification Adapter [Backend]: notification lines in G1, G3, G4; 'sendMessage()' in Lab3 1.1, 1.3, 1.4; Lab4 signal 9; Lab1 Student.email and Student.phone as contact data.
- C-09 Data Access Layer [Data]: Req-1 (ENUM), Req-2, Req-4; NFR-03, NFR-05; "Database" in all Lab3 scenarios; Lab4 step 3; all Lab1 entities.
- C-10 PostgreSQL [Data]: Req-1, Req-6; NFR-07; all Lab1 entities.

Requirement coverage:
- **Req-1:** Booking Service, Data Access Layer, PostgreSQL.
- **Req-2:** Booking Service, Data Access Layer.
- **Req-3:** Payment Service, Monobank API.
- **Req-4:** Booking Service, Data Access Layer (locking).
- **Req-5:** Payment Service (declined payment), BookingTimeoutWorker (timeout), Booking Service.
- **Req-6:** PostgreSQL maintenance. There is no application component for it, as Lab2 5 paragraph says.

Fully traced chain (Req, Gherkin, Lab3, Lab4, component): the booking and payment scenario (C-01, C-02, C-05, C-06, C-07, X-01). Partly traced: Identity, Schedule, Notification Adapter, Data Access Layer.

## 2. Traceability of dependencies
Note: `(D-1)` marks a dependency derived from actor goals in Lab2 §1.1 without an explicit step in Lab3 sequence diagrams.
- Web UI to API Gateway: Lab3 1.1.1, 1.2.1, 1.4.1; Lab4 step 1.
- Web UI to Monobank payment page: Lab3 1.1.7–8; Lab4 step 5.
- API Gateway to Identity ('validateToken'): Lab3 1.1.2, 1.2.2, 1.4.2.
- API Gateway to Booking: Lab3 1.1.2, 1.2.2, 1.4.2.
- API Gateway to Schedule: only Lab2 §1.1 (D-1).
- API Gateway to Payment (admin): only Lab2 §1.1 and the log of late webhooks (D-1).
- Booking to Payment ('IPayment'): Lab3 1.1.5, 1.4; Lab4 step 4.
- BookingTimeoutWorker to Booking: Lab3 1.3A.3.
- Booking and Payment to Notification Adapter: Lab3 1.1.10, 1.3A, 1.3B, 1.4; Lab4 signal 9.
- Payment to Monobank API: Lab3 1.1.6, 1.3; Lab4 step 4.
- Monobank API to Payment (webhook, bypassing API Gateway): Lab3 1.1.9–10, 1.3B; Lab4 steps 6A, 7.
- Services and worker to Data Access Layer to PostgreSQL: "Database" in Lab3; Lab3 1.3A.1 (worker polls the database).