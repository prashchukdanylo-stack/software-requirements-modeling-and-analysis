
### Intention and сriteria:
The primary goal of this work is to establish an unambiguous, fully testable requirements specification and a Use Case diagram with well-defined scope boundaries for a dance studio system. Success is measured by having real many-to-many traceability matrix links, at least one justified uncovered requirement.

* **Top 3 mismatches with Lab1 model:** 
1. **AI Output Correction (Prompt 3 - Promo Codes Removal):**  
   * *Problem:* The initial AI output from prompt 3 introduced promotional codes and discount attributes not supported by the studio business model.  
   * *Resolution:* Corrected `spec.md` and domain boundaries to remove promo code logic.  
   * *Fix Commit SHA:* `03c32c0`.

2. **Administrator Role Model Synchronization:**  
   * *Problem:* `spec.md` defined an `Administrator` actor, but the initial ER model lacked explicit role attributes.  
   * *Resolution:* Updated ER schema and specification to support role-based authorization.  
   * *Fix Commit SHA:* `df0e5fc`.
3. **Booking Status ENUM Constraint:**  
   * *Problem:* `spec.md` enforces a strict state machine (`pending` --`confirmed` / `cancelled`), but Lab 1 defined `status : string` without DB constraints.  
   * *Resolution:* Constrained `status` to a strict enumerated type across `spec.md`, EARS `REQ-01`, and Gherkin scenarios.  
   * *Fix Commit SHA:* `df0e5fc`.


### Key decision — ADR:
* **Chosen option:** Model "Uc_Book" and "UC_CancelBooking" as independent use Cases, keeping Monobank API strictly as an external system actor calling "UC_ProcessWebhook".
* **Alternatives considered:** Adding system implementation steps ("Initialize Payment Session") as separate Use Cases. Drawing include or extend arrows directly from actors to internal system steps.
* **Rationale:** Keeping actor goals cleanly separated from technical execution steps ensures adherence to standard UML semantics.

 
 ### Why the scenarios are split this way
 We split the system into distinct steps to make it reliable and clean:
 * **Booking (UC_Book) and Webhook (UC_ProcessWebhook) are separate:**
 Creating a booking checks if there are free seats in the hall. Paying talks to the Monobank API. Keeping them separate means if Monobank goes down, our internal system still works correctly and doesn't crash.
 * **Asynchronous Webhook processing:**
 Monobank sends a notification (webhook) to our system when the payment succeeds. This is separate from the UI so that if a student accidentally closes the browser tab while paying, their booking still gets confirmed.
 * **15-minute expiration timeout:** 
 A booking goes through simple steps: pending_payment -- confirmed or cancelled. If a student doesn't pay within 15 minutes, the system automatically cancels the booking and frees up the seat for others.
 ## (REQ-06)
 REQ-06` ("Automatic system checks for database integrity") is a background system maintenance task executed by database engine triggers or DevOps scripts. It is intentionally marked as **Not covered** in the matrix because it is an internal infrastructure requirement rather than a user-facing Gherkin scenario.


 ### Verification — alignment with prior data model:
Alignment was verified by directly mapping the entities and attributes from the Lab 1 ER diagram ('Student', 'Instructor', 'Lesson', 'Hall', 'DanceStyle', 'Booking') to the actor goals in the Use Cases and EARS triggers. All identified structural gaps were documented in the audit section of spec.md, and non-functional infrastructure tasks (such as daily database backups in REQ-06) were intentionally excluded with clear engineering justifications.