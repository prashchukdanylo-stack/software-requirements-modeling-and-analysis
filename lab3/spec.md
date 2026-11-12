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
1. Student sends request for booking exact 'lessonId'.
2. 'APIGateway' identifies authorisation token via 'validateToken' and redirects this request to 'BookingService'.
3. 'BookingService' checks 'Lesson.max_capacity' and 'Hall.capacity' in 'Database', and the count of active bookings ('countActiveBookings') to verify slot availability ('hasFreeSlots').
4. 'BookingService' creates entry in 'bookings' table with status 'pending_payment' and fixes timestamp 'booked_at'.
5. 'BookingService' refers to 'PaymentService' to initialise invoice.
6. 'PaymentService' calls external 'MonobankAPI' ('createInvoice') and gets 'pageUrl' and 'invoiceId', gets sum from Lesson.price and saves entry in payments with monobank_invoice_id and status pending_payment.
7. 'BookingService' returns 'pageUrl' to Student.
8. Student pays for lesson in his Monobank app.
9. 'MonobankAPI' sends asynchronous Webhook with ECDSA signature on backend.
10. PaymentService receives the webhook and processes it in the following order:
    Signature Verification: PaymentService verifies the ECDSA SHA-256 signature. If the signature is invalid, the request is rejected immediately without state changes, returning HTTP 400 Bad Request.
    Internal Branches Based on Booking State and payload.status:
        Successful Payment (payload.status = 'success'): If the booking is in pending_payment status, both payments and bookings statuses are updated to confirmed within a single database transaction, and a confirmation message is sent to the Student.
        Idempotency (Already Processed Webhook): If the booking and payment are already in confirmed or cancelled status, no database changes are made, and HTTP 200 OK is returned.
        Late success Webhook for a cancelled Booking: If the booking was already cancelled (e.g., due to a timeout), the booking status is not restored; the event is logged for a manual refund by the Administrator, and HTTP 200 OK is returned.
        Non-success Payment Status from Monobank (payload.status != 'success', e.g., err_declined): Processing is redirected to the logic defined in Scenario 1.3 (cancelling the unpaid booking and updating the payment status).

### Scenario 1.2: attempt to book a slot for the lesson, which is full
**Initial conditions**: Lesson has 0 available space (confirmed + pending_payment >=
min(Lesson.max_capacity, Hall.capacity)).

**Step by step explaining**:
1. Student sends request for booking exact 'lessonId'.
2. 'APIGateway' identifies authorisation token via 'validateToken' and redirects this request to 'BookingService'.
3. 'BookingService' checks 'Lesson.max_capacity' and 'Hall.capacity' in 'Database', amount of bookings(confirmed + pending_payment) via countActiveBookings.
4. 'Database' returns count of active Bookings and capacity, 'BookingService'calculates 'hasFreeSlots' and gets that 0 place is free.
5. Booking is not created, transaction is rolled back.
6. Student gets HTTP 409 Conflict with localised message "Class is full".


### Scenario 1.3: cancelling booking after timeout or payment error
**Initial conditions**: For the timeout trigger: A bookings entry exists with status pending_payment, and payment has not been completed within 15 minutes (REQ-05, NFR-04).
For the webhook error trigger: A bookings entry exists with status pending_payment, and an asynchronous webhook with an error status (err_declined) is received from Monobank API.

**Step by step explaining**: 
Sequence 1.3A: 15-minute payment timeout (Background Worker)
    Background service BookingTimeoutWorker periodically polls Database for expired unpaid reservations.  
    BookingTimeoutWorker detects entries in bookings where status = 'pending_payment' and now() - booked_at > 15 minutes.
    BookingTimeoutWorker invokes BookingService to cancel the expired reservation.
    BookingService updates payments.status to expired and bookings.status to cancelled in Database.
    The available slot is returned to the pool for other users, an invoice closure command cancelInvoice(invoiceId) is issued, and a timeout notification (sendMessage()) is sent to the Student.

Sequence 1.3B: Payment declined by Monobank (err_declined Webhook)
    Monobank API sends an asynchronous webhook containing a financial failure status (err_declined or another non-success status) along with an ECDSA signature directly to PaymentService (bypassing APIGateway).
    PaymentService verifies the ECDSA SHA-256 signature (NFR-05; returns HTTP 400 Bad Request if invalid).
    Idempotency Check: PaymentService and BookingService check the current state of the booking and payment in Database:
    If the booking is already in cancelled or confirmed status (already processed previously), no database changes occur, and HTTP 200 OK is returned.
    If the payment is in pending_payment status:
        PaymentService updates payments.status to cancelled, and BookingService updates bookings.status to cancelled in Database.
        The available slot is immediately returned to the pool for other users.
        cancelInvoice(invoiceId) is invoked, HTTP 200 OK is returned to Monobank, and a payment failure notification (sendMessage()) is sent to the Student.
### Scenario 1.4: student cancels booking

**Initial conditions**: student has booking with 'pending_payment' or 'confirmed' status.

**Step by step explanation**:
1. Student clicks "Cancel booking" in his profile.
2. 'APIGateway' validates auhtorisation token via validateToken and redirects request 'cancelBookingByUser(studentId, bookingId)' to 'BookingService'.
3. 'BookingService' queries 'Database' to verify ownership (studentId) and get booking status.
4. BookingService executes the branch matching the booking state:
    Booking not found or belongs to another student: returns HTTP 404 Not Found.
    Booking already cancelled: returns HTTP 409 Conflict.
    Booking is pending_payment: updates bookings.status and payments.status to cancelled in Database, calls PaymentService (cancelInvoice(bookingId)), releases the slot, and notifies the Student of successful cancellation.
    Booking is confirmed: updates bookings.status to cancelled in Database while keeping payments.status as confirmed, releases the slot, and sends a confirmation message to the Student with instructions on requesting a manual refund from the Administrator.

## Audit

### Participants with entities in ER

Student: entity Student, user that initilaises bookings and payment.
API Gateway: Component of routing and token validation, accepts HTTP requests from client, check JWT token and redirects to microservices.
BookingService: Business logic of coordinating Lesson, Hall, Booking entities; checks for capacity, blocks FOR UPDATE, takes care of booking statuses.
PaymentService: Business logic of Payment and external API, nitializes payment invoices, processes and checks ECDSA signatures of webhooks.
MonobankAPI: external integration payment system, processes transactions in its app and sends asynchronous Webhook.
PostgreSQL: relational database (tables lessons, halls, bookings, payments), ensures integrity and uniqueness of data.

### Acceptance Criteria
**Scenario 1.1: Successful booking and payment:**
    Given the student is logged in and the lesson has free spots available.
    When the student requests to book the lesson and completes the payment via Monobank.
    Then the booking status and payment status change to confirmed.
    And the student receives a confirmation message.
    And if the same webhook is sent twice (duplicate), the system ignores the duplicate, makes no changes, and returns HTTP 200 OK.

**Scenario 1.2: Class is full:**
    Given the lesson has 0 free spots left (reached max_capacity or hall_capacity).
    When the student tries to book this lesson.
    Then the system rejects the booking, rolls back the transaction, and returns HTTP 409 Conflict with the message "Class is full".

**Scenario 1.3: Cancellation after timeout or error**
    Given a booking is in pending_payment status.
    When 15 minutes pass without payment, OR Monobank sends an error status (err_declined).
    Then the booking and payment are automatically cancelled, and the free spot returns to the pool.
    And the student receives a notification about the cancellation.

**Scenario 1.4: Student cancels the booking**
    Given the student has an active booking (pending_payment or confirmed).
    When the student clicks "Cancel booking".
    Then the booking status changes to cancelled and the spot is released.
    And if the status was pending_payment, the payment invoice is also cancelled.
    And if the status was confirmed, instructions for a manual refund from the administrator are provided.