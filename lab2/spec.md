## 1.1 Actors
* **User:** browses dance styles, lesson schedules.
* **Student:** looks for a class to visit, books it, main user of dance studio.
Main goals in system: register and manipulate with personal profile;
look through classes timetable, with ability to filter them by dance style and instructors;
create, cancel, check bookings whenever it is needed.

* **Instructor:** teaches students, runs dance classes.
Main goals in system: look for classes timetable to understand when to start lesson and in which hall;
have an access to list of applied students, to understand how many people is going to be there.
* **Administrator:** head of studio, responsible for managing everything about classes, instructors, control.
Main goals in system: add new halls, instructors and dance styles to the system;
create and plan timetable for lessons;
moderate all booking processes, cope with financial issues.
* **Monobank API**: external service.
Main goals in system: process requests for booking lessons, return transaction status after;
send students notifications about success/fail of their bookings. 

### 1.2 Out of Scope  
The following features are intentionally excluded from the system scope for this iteration:
*   **Automated Refunds:** If a student cancels a confirmed booking, funds are not automatically refunded via Monobank API. Manual contact with administration is required.
*   **Promotions & Discount Codes:** No promotional codes, loyalty points, or gift certificates are supported.
*   **Online Streaming / Video Lessons:** The system handles only physical in-person studio hall bookings.
*   **Instructor Payroll:** No automated salary or commission calculations for instructors.

## 2.1 EARS Requirements

* **Req-1(ubiquitous):** system SHALL maintain a Booking record with state strictly constrained to one of: pending, confirmed, cancelled.
* **Req-2(event-driven):** WHEN Student requests to book a Lesson, system SHALL check if current confirmed + pending bookings are less than Hall.capacity AND Lesson.max_capacity.
* **Req-3(event-driven):** WHEN Booking creation is successful with status "pending", system SHALL initialise payment session using Monobank API and generate link for payment.
* **Req-4(state-driven):** WHILE system is processing payment creation, it SHALL block an ability to book that same place at the same Lesson to prevent double-booking of the same spot.
* **Req-5(unwanted event):** IF Monobank API returns a transaction error, or a 15-minute payment timeout is reached, THEN system SHALL change Booking status for "cancelled" and stop blocking that place from being booked.
* **REQ-06 (Ubiquitous):** The system SHALL perform automatic background database integrity and consistency checks.

## 2.2 NFR

* **NFR-01:** The Web application ui SHALL support Google Chrome (v110+), Mozilla Firefox(v110+), Safari(v16+) on mobile devices (IOS 15+, Android 10+) and desktop systems (resolution from 360px up to 1920 x 1080).
* **NFR-02:** The system backend SHALL process up to 50 concurrent booking requests per second during peak schedule opening times, with a baseline average load of 5 requests per second. If traffic exceeds 50 requests per second, extra requests SHALL be queued up to a capacity limit of 200 requests before returning an error "too many requests".
* **NFR-03:** The API response time for booking creation and schedule queries SHALL NOT exceed 300ms at 95th percentile under normal load, measured from client HTTP request to HTTP response payload delivery.
* **NFR-04:** Payment hold window for any pending booking SHALL be strictly bounded to 15 minutes from creation timestamp.
* **NFR-05:** Webhook endpoints from Monobank API SHALL validate SHA-256 digital signatures using ECDSA public keys provided by Monobank. All student personal data (emails, full names)
 SHALL be encrypted at rest (AES-256) and transmitted via TLS 1.3 over HTTPS.
* **NFR-06:** User interface and error messages SHALL be localized in Ukrainian.
* **NFR-07:** Backend services SHALL be executed on Node.js(v20+ LTS) with PostgreSQL(v15+) database running in Docker environment.

## 3. User Stories, Gherkin

**Feature:** Booking and paying for Lesson in dance studio. 
As a Student, I want to book a place for a Lesson and pay for it with my card to be absolutely sure, that I am included.

*Scenario: successful booking and payment using MonoBank API*
Given student "Sigma" is registered in system
And there is active Lesson "Hip Hop" on "10-23-2026 18:00" in "Hall 2" with free space
When Student chooses this Lesson and clicks "Book and pay"
Then system registers an entry in Booking table with "pending" status
And system reserves spot for 15 minutes
And redirects Student to MonoBank payment page
When MonoBank API sends a response, that transaction is successful
Then system changes entry's status on "confirmed"
And sends Student a notification about success of registration
And remaining capacity of Hall becomes less by 1.

*Scenario: Attempting to book a fully booked class*
Given student "Sigma" is registered in system
And the lesson "Contemporary beginners" on "11-23-2026 15:00" in "Hall 1" has 0 remaining slots
When student requests to book a spot for this class
Then system rejects booking request with error "Class is full"
And no new booking record is created.

*Scenario: Booking is cancelled in case of an error during payment*
Given student "Sigma" is registered in system
And he has a booking for "Jazz Funk" on "12-23-2026 12:00" in "Hall 3" with status "pending"
And booking record was created at "14:00:00"
When the time reaches "14:15:01" without receiving a successful payment webhook
Then system updates booking status to "cancelled"
And reserved spot is returned to available capacity for that class. 

*Scenario: Student cancels Booking*
Given student "Sigma" is registered in system
And he has a booking for lesson "Breakdance" on "13-23-2026 20:00" in "Hall 1"
When Student clicks "Cancel booking" in his profile for lesson "Breakdance"
Then system changes entry's status on "cancelled"
And frees a place for other students to book
And notifies Student that Booking is successfully canceled.

## 4. Traceability Matrix

| EARS Req ID | Description| Use Case ID | Gherkin Scenario ID | Status |
| :--- | :--- | :--- | :--- | :--- |
| **REQ-01** | Booking status enum constraint (pending, confirmed, cancelled)| UC_Book, UC_Cancel | Scenario 1, Scenario 3, Scenario 4 | **Covered** |
| **REQ-02** | Checking capacity before booking | UC_Book | Scenario 1, Scenario 2 | **Covered** |
| **REQ-03** | Initialisation of payment session  Monobank API | UC_Book | Sсenario 1 | **Covered** |
| **REQ-04** | Blocking place when processing the payment | UC_Book | Scenario 1 | **Covered** |
| **REQ-05** | Cancelling Booking in case of errors or timeouts in  Monobank API | UC_Book UC_Cancel, UC_Webhook | Scenario 3 | **Covered** |
| **REQ-06** | Automatic system checks for correctness of all data in database| — | — |  **Not covered** |

---

### 5. About uncovered requirement

* **Req-6(ubiquitous):** The system SHALL perform automatic background database integrity and consistency checks.

* **Why there isn't Gherkin for that**  
  REQ-06 describes a background database integrity check performed automatically by the database engine or system maintenance scripts. It is intentionally not covered by Gherkin scenarios or student-facing Use Cases because it represents a system-level infrastructure and data maintenance task.

### 6. Audit

* **EARS and NFR:** All functional requirements use precise mathematical conditions and explicit domain terms (e.g., "15 minutes", "zero remaining capacity")

* **UseCase and Actors:** implementation details were removed from Use Case Diagram. Monobank API acts as an external system actor strictly for sending incoming webhooks.
    
* **Top 3 mismatches with Lab1 model and solutions:** 
*Promo code removal:* Issue: initial AI generation from Prompt 3 introduced promotional discount codes into the domain model. Resolution: Manually corrected spec requirements and model scope to strictly focus on flat-rate bookings.
*Missing Administrator role:* Issue: spec.md required Administrator actor, but Lab1 ER schema lacked it. Resolution: Synchronized role management attributes in model and specification.
*Unconstrained booking status field:* Issue: "status" field in Lab1 was an unconstrained string.
Resolution: Restructured status attribute to an ENUM type (pending, confirmed, cancelled).