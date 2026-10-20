
### Intention and сriteria:
The primary goal of this work is to establish an unambiguous, fully testable requirements specification and a Use Case diagram with well-defined scope boundaries for a dance studio system. Success is measured by having real many-to-many traceability matrix links, at least one justified uncovered requirement.

* **Top 3 mismatches with Lab1 model:** 
*First mismatch:* in Lab2 specification, there is a 'Administrator' actor, who manages system directories. However, in Lab1 there was no admin or unified 'User' entity. Resolution: business logic requires an administrative role, so future schema should introduce it.
*Second mismatch:* Lab2 specifies integration with external Monobank API, including transaction processing and Webhook handling. However, Lab1 lacks "Payment" entity to store everything about it. Resolution: payment status in current fata model is stored via string field "status: string" in Booking table. Payment entity should be added, with linked to Booking using 'booking_id'.
*Third mismatch:* Requirements REQ-02 and REQ-05 define a strict lifecycle for Booking states ("pending" -- "confirmed" or "cancelled"). However, in Lab1 ER diagram, Booking entity just has 'status: string', without any constraints etc. Resolution: create custom enum type Booking_status with 3 possible states.


### Key decision — ADR:
* **Chosen option:** Decouple booking creation 'UC_Book' from external payment initialization 'UC_Pay' and asynchronous webhook processing 'UC_Webhook'.
* **Alternatives considered:** Combining booking and payment into a single monolithic Use Case with synchronous waiting for the payment response.
* **Rationale:** Decoupling ensures that if the external Monobank API experiences downtime or network latency, the internal reservation logic remains functional. Furthermore, asynchronous webhook handling guarantees that a student's booking is confirmed even if they accidentally close their browser tab mid-transaction.

 
 ### Why the scenarios are split this way
 We split the system into distinct steps to make it reliable and clean:
 * **Booking (UC_Book) and Payment (UC_Pay) are separate:**
 Creating a booking checks if there are free seats in the hall. Paying talks to the Monobank API. Keeping them separate means if Monobank goes down, our internal system still works correctly and doesn't crash.
 * **Payment Confirmation (UC_Webhook) works in the background:**
 Monobank sends a notification (webhook) to our system when the payment succeeds. This is separate from the UI so that if a student accidentally closes the browser tab while paying, their booking still gets confirmed.
 * **Clear Rules for Cancellation (UC_Cancel):** 
 A booking goes through simple steps: pending_payment -- confirmed or cancelled. If a student doesn't pay within 15 minutes, the system automatically cancels the booking and frees up the seat for others.
 ## Why Backup is Not a User Scenario (REQ-06)
 Backing up the database every night is a background technical job for the server administrator. Users and instructors don't interact with backups, so it doesn't need a Gherkin test or a Use Case diagram. It is tested on the server side instead.

 ### Verification — alignment with prior data model:
Alignment was verified by directly mapping the entities and attributes from the Lab 1 ER diagram ('Student', 'Instructor', 'Lesson', 'Hall', 'DanceStyle', 'Booking') to the actor goals in the Use Cases and EARS triggers. All identified structural gaps were documented in the audit section of spec.md, and non-functional infrastructure tasks (such as daily database backups in REQ-06) were intentionally excluded with clear engineering justifications.