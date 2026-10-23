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
* **Req-2(event-driven):** WHEN Student requests to book a Lesson, system SHALL check if current confired + pending bookings are less than Hall.capacity AND Lesson.max_capacity.
* **Req-3(event-driven):** WHEN Booking creation is succesful with status "pending", system SHALL initialise payment session using Monobank API and generate link for payment.
* **Req-4(state-driven):** WHILE system is processing payment creation, it SHALL block an ability to book that same place at the same Lesson, so there aren't any problems.
* **Req-5(unwanted event):** IF Monobank API returns a transaction error, or time of exceptancy for 15 minutes is over, THEN system SHALL change Booking status for "cancelled" and stop blocking that place from being booked.
* **Req-6(event_driven):** WHEN Administrator creates or updates classes timetable, system SHALL check if all relations among Hall, Instructor and DanceStyle aro correct.

## 2.2 NFR

* **NFR-01:** The Web application ui SHALL support Google Chrome (v110+), Mozilla Firefox(v110+), Safari(v16+) on mobile devices (IOS 15+, Android 10+) and desktop systems (resolution from 360px up to 1920 x 1080).
* **NFR-02:** The system backend SHALL process up to 50 concurrent booking requests per second during peak schedule opening times, with a baseline average load of 5 requests per second. If traffic exceeds 50 requests per second, extra requests SHALL be queued up to a capacity limit of 200 requests before returning an error "too many requests".
* **NFR-03:** The API response time for booking creation and schedule queries SHALL NOT exceed 300ms at 95th percentile under normal load, measured from client HTTP request to HTTP response pauload delivery.
* **NFR-04:** Payment hold window for any pending booking SHALL be strictly bounded to 15 minutes from creation timestamp.
* **NFR-05:** Webhook endpoints from Monobank API SHALL validate SHA-256 digital signatures using ECDSA public keys provided by Monobank. All student personal data (emails, full names)
 SHALL be encrypted at rest (AES-256) and transmitted via TLS 1.3 over HTTPS.
* **NFR-06:** User interface and error messages SHALL be localized in Ukrainian.
* **NFR-07:** Backend services SHALL be executed on Node.js(v20+ LTS) with PostgreSQL(v15+) database running in Docker environment.

## 3. User Stories, Gherkin

**Feature:** Booking and paying for Lesson in dance studio. 
As a Student, I want to book a place for a Lesson and pay for it with my my card to be absolutely sure, that I am included.

*Scenario: successful booking and payment using MonoBank API*
Given student "Sigma" is registered in system
And there is active Lesson "Hip Hop" on "10-23-2026 18:00" in "Hall 2" with free space
When Student chooses this Lesson and clicks "Book and pay"
Then system registers an entry in Booking table with "pending" status
And system reserves spot for 15 minutes
And redirects Student to MonoBank payment page
When MonoBank API sends a response, that transaction is succesfull
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
When the time reaches "14:15:01" without recieving a successful payment webhook
Then system updates booking status to "cancelled"
And reserved spot is returned to available capacity for that class. 

*Scenario: Student cancels Booking*
Given student "Sigma" is registered in system
And he has a booking for lesson "Breakdance" on "13-23-2026 20:00" in "Hall 1"
When Student clicks "Cancel booking" in his profile for lesson "Breakdance"
Then system changes entry's status on "cancelled"
And frees a place for other students to book
And notifies Student that Booking is succesfully canceled.

## Traceability Matrix

| EARS Req ID | Description| User Story ID | Use Case ID | Gherkin Scenario ID | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **REQ-01** | support of relations  Student --  Lesson -- Booking | UserStory-01, UserStory-05 | UC_Book, UC_Cancel | SC-01, SC-02, SC-03 | **Covered** |
| **REQ-02** | Checking capacity of Hall and creating Booking (pending_payment) | UserStory-01 | UC_Book | SC-01 | **Covered** |
| **REQ-03** | Initialisation of payment session  Monobank API | UserStory-01 | UC_Pay | SC-01 | **Covered** |
| **REQ-04** | Blocking place when processing the payment | UserStory-01 | UC_Book, UC_Pay | SC-01 | **Covered** |
| **REQ-05** | Cancelling Booking in case of errors or timeouts in  Monobank API | UserStory-01, UserStory-05 | UC_Pay, UC_Cancel, UC_Webhook | SC-02 | **Covered** |
| **REQ-06** | Automatic system checks for correctness of all data in data base| — | — | — |  **Not covered** |

---

###  About uncovered requirement

* **Req-6(event_driven):** WHEN Administrator creates or updates classes timetable, system SHALL check if all relations among Hall, Instructor and DanceStyle aro correct.

* **Why there are no useCase and Gherkin**  
  Since Use Cases and Gherkin scenarios are designed to describe system behavior during direct interaction with a user or an external API, this requirement is not verified via a user interface scenario.

### Audit

* **EARS:** All requirements within the specification have been audited to eliminate subjactive terms, such as "fast", "conveinent" etc. For example, instead of an ambiguous phrase like "System SHALL quickly check for available place", an event-driven EARS requirement REQ-2 was defined: "WHEN Student tries too book a Lesson, system SHALL check for Hall capacity and create payment registration in Booking table with "pending" status." This can be easily verified by some db queries.

* **include/extend:** "include" relationship is between 'UC_Book' (Book a Lesson) and 'UC_Pay' (Initiate Monobank payment), because booking transaction cannot be initialized without creating a payment session using Monobank API. Also, processing a payment webhook 'UC_Webhook' is an essential dependency for verifying final booking state.
"extend" relationship is between 'UC_Cancel' and 'UC_Book', beacuse cancellation is optional, happens by the student or automatically upon a payment timeout, without disrupting standart booking lifecycle.
    
* **Top 3 mismatches with Lab1 model:** 
*First mismatch:* in Lab2 specification, there is a 'Administrator' actor, who manages system directories. However, in Lab1 there was no admin or unified 'User' entity. Resolution: business logic requires an administrative role, so future schema should introduce it.
*Second mismatch:* Lab2 specifies integration with external Monobank API, including transaction processing and Webhook handling. However, Lab1 lacks "Payment" entity to store everything about it. Resolution: payment status in current fata model is stored via string field "status: string" in Booking table. Payment entity should be added, with linked to Booking using 'booking_id'.
*Third mismatch:* Requirements REQ-02 and REQ-05 define a strict lifecycle for Booking states ("pending" -- "confirmed" or "cancelled"). However, in Lab1 ER diagram, Booking entoty just has 'status: string', without any constraints etc. Resolution: create custom enum type Booking_status with 3 possible states.