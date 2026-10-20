## Actors
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

## EARS Requirements

* **Req-1(ubiquitous):** system SHALL ensure keeping mandatory relation among entites Student, Lesson and created Booking registration in data base.
* **Req-2(event-driven):** WHEN Student tries too book a Lesson, system SHALL check for Hall capacity and create payment registration in Booking table with "pending" status.
* **Req-3(event-driven):** WHEN Booking creation is succesfull with status "pending", system SHALL initialise payment session using Monobank API and generate link for payment.
* **Req-4(state-driven):** WHILE system is processing payment creation, it SHALL block an ability to book that same place at the same Lesson, so there aren't any problems.
* **Req-5(unwanted event):** IF Monobank API returns a transaction error, or time of exceptancy is over, THEN system SHALL change Booking status for "cancelled" and stop blocking that place from being booked.
* **Req-6(event_driven):** WHEN Administrator creates or updates classes timetable, system SHALL check if all relations among Hall, Instructor and DanceStyle aro correct.

## User Stories

* **UserStory-1 (booking and paying):** as a Student, I want to choose a Lesson, book a place for it and pay for it with my Monobank card, so that I am surely registered.
* **UserStory-2 (searching and filtering):** as a Student, I want to look through the timetable and filter Lessons by DanceStyles and Instructors.
* **UserStory-3 (checking students list):** as a Instructor, I want to check a list of all Students, applied for my Lesson to have a qualitative plan of it.
* **UserStory-4 (planning and timetable):** as a Administrator, I want to create new Lessons, relate them to Halls, DanceStyles and Instructors to maintain useful timetable for all users of studio.\
* **UserStory-5 (cancelling):** as a Student, I want to have an ability to cancel my Booking before Lesson starts, so other people can join.  

## Gherkin

**Feature:** Booking and paying for Lesson in dance studio. 
As a Student, I want to book a place for a Lesson and pay for it with my my card to be absolutely sure, that I am included.

*Scenario: successful booking and payment using MonoBank API*
Given student is registered in system
And there is active Lesson "Hip Hop" in "Hall 2" with free space
When Student chooses Lesson and clicks "Book and pay"
Then system registers an entry in Booking table with "pending" status
And redirects Student to MonoBank payment page
When MonoBank API sends a response, that transaction is succesfull
Then system changes entry's status on "confirmed"
And sends Student a notification about success of registration.

*Scenario: Booking is cancelled in case of an error during payment*
Given system registers an entry in Booking table with "pending" status
When MonoBank API sends a response, that there is an error or "failure" status
Then system changes entry's status on "cancelled"
And frees a place for other students to book
And notifies student about payment problems.

*Scenario: Student cancels Booking*
Given student's entry in Booking table has status "confirmed"
When Student clicks "Cancel booking" in his profile
Then system changes entry's status on "cancelled"
And frees a place for other students to book
And notifies STudent that Booking is succesfully canceled

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