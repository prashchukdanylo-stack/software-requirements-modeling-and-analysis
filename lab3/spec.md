## All actors and entities

- **Student**: searches for, books, and pays for classes.
- **Instructor**: views the schedule and lists of enrolled students.
- **Administrator**: manages the schedule, halls, dance styles, and transactions.
- **Monobank API (External Actor)**: an external payment service (creates payment sessions, sends signed webhooks).
- **Backend Domain Services**: APIGateway, BookingService, PaymentService, Database (PostgreSQL), BookingTimeoutWorker.


## UC01: Booking and paying for lesson

### Scenario 1.1: Successful booking and paying via Monobank API
**Initial conditions**: Student is authorised, Lesson has available slots (confirmed + pending_payment <
min(Lesson.max_capacity, Hall.capacity)).

**Step by step explaining**:
1. Student sends request for booking exact 'lesson_id'.
2. 'APIGateway' identifies authorisation token and redirects this request to 'BookingService'.
3. 'BookingService' checks 'Lesson.max_capacity' and 'Hall.capacity' in 'Database'.
4. 'BookingService' creates entry in 'bookings' table with status 'pending_payment' and fixes timestamp 'booked_at'.
5. 'BookingService' refers to 'PaymentService' to initialise invoice.
6. 'Payment Service' calls external 'Monobank API' and gets 'pageUrl' and 'invoiceId', gets sum from Lesson.price and saves entry in payments with monobank_invoice_id.
7. 'BookingService' returns 'pageUrl' to Student.
8. Student pays for lesson in his Monobank app.
9. 'Monobank API' sends asynchronous Webhook with ECDSA signature on backend.
10. 'PaymentService' validates ECDSA SHA-256 signature, updates entry status in 'bookings' on 'confirmed', sends message to Student.


### Scenario 1.2: attempt to book a slot for the lesson, which is full
**Initial conditions**: Lesson has 0 available space (confirmed + pending_payment <
min(Lesson.max_capacity, Hall.capacity)).

**Step by step explaining**:
1. Student sends request for booking exact 'lesson_id'.
2. 'APIGateway' identifies authorisation token and redirects this request to 'BookingService'.
3. 'BookingService' checks 'Lesson.max_capacity' and 'Hall.capacity' in 'Database', amount of bookings(confirmed + pending_payment).
4. 'Database' return count of available slots, 'BookingService' and understands that 0 place is free.
5. Booking is not created, transaction is rolled back.
6. Student gets HTTP 409 Conflict with localised message "Class is full".


### Scenario 1.3: cancelling booking after timeout or payment error
**Initial conditions**: 'booking' entry is created with status 'pending', but payment isn't done for 15 minutes (REQ-05, NFR-04).

**Step by step explaining**: 
1. Background service 'BookingTimeoutWorker' periodically checks unavailable slots.
2. 'BookingTimeoutWorker' detects entries in 'bookings', where 'status = pending' and 'now() - created_at > 15 minutes'.
3. 'BookingTimeoutWorker' calls 'BookingService' to cancel request.
4. State change / Webhook processing:
    - If Monobank returned status 'err_declined' or 15 minutes timeout expires:
    - 'BookingService' updates status in 'Database' on 'cancelled'.
    - Unavailable slot returns to access for other users.
    -   For the payment service sends a command to close invoice 'cancelInvoice()'.

### Scenario 1.4: student cancels booking

**Initial conditions**: student has booking with 'pending' or 'confirmed' status.

**Step by step explanation**:
1. Student clicks "Cancel booking" in his profile.
2. 'APIGateway' redirects request 'cancelBooking(booking_id)' to 'BookingService'.
3. 'BookingService' checks for booking status in 'Database'.
4. 'BookingService' changes status on 'cancelled'.
5. Amount of available slots for Lesson increments for 1.
6. Student gets message about successful cancelling and instructions for how to get a refund from Admin.

## Audit

### Participants with entities in ER

Student: entity Student, user that initilaises bookings and payment.
API Gateway: Component of routing and token validation, accepts HTTP requests from client, check JWT token and redirects to microservices.
BookingService: Business logic of coordinating Lesson, Hall, Booking entities; checks for capacity, blocks FOR UPDATE, takes care of booking statuses.
PaymentService: Business logic of Payment and external API, nitializes payment invoices, processes and checks ECDSA signatures of webhooks.
MonobankAPI: external integration payment system, processes transactions in its app and sends asynchronous Webhook.
PostgreSQL: relational database (tables lessons, halls, bookings, payments), ensures integrity and uniqueness of data.

### Steps coincide with Gherkin

The logic of the steps in the diagrams fully covers the behavioral scenarios (Gherkin from Task 2):
Successful booking: The Gherkin steps "Given student is registered... When student chooses Lesson... Then system registers entry with pending status... redirects to Monobank..." are fully represented in Diagram 1.1.
 Fully booked class: The step "Then system rejects booking request with error 'Class is full'" is implemented via a capacity check and by returning a 409 Conflict status in Diagram 1.2.
 15-minute timeout and errors: The step "When the time reaches 14:15:01 without receiving successful payment..." is handled by a background worker ('BookingTimeoutWorker') and the processing of erroneous webhooks in Diagram 1.3.
Cancellation by the student: The step "When Student clicks 'Cancel booking'..." is depicted in Diagram 1.4, involving a status change to "cancelled" and the release of the slot.

### Top 3 mismatches

**1.Refining booking status names**

    The Problem: Early Gherkin scenarios used a general "pending" status, which was too vague and didn't clearly state whether the slot was actually blocked for payment.

    The Solution: We updated the status to "pending_payment" across all diagrams and the ER model. This clearly indicates that a spot is temporarily locked while waiting for the bank transaction.

**2.Handling duplicate webhooks (idempotency)**

    The Problem: Payment gateways like Monobank can sometimes send the same webhook notification more than once due to network retries (at-least-once delivery).

    The Solution: We added an idempotency check in Diagram 1.1. Before updating the database, the system checks if the booking is already confirmed. If it is, it skips duplicate processing while still returning a success response to the bank.

**3.Combined capacity check (Lesson vs. Hall limits)**

    The Problem: Initial Gherkin scenarios only checked whether the specific lesson had free space, ignoring the physical limitations of the studio hall itself.

    The Solution: We updated the database queries on the diagrams to evaluate both limits by calculating min(max_capacity, hall_capacity), ensuring the studio never overbooks a room.
