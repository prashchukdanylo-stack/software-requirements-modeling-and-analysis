## All actors and entities

- **Student (users з role='STUDENT')**: здійснює пошук, бронювання та оплату занять.
- **Instructor (users з role='INSTRUCTOR', instructors)**: переглядає розклад та списки записаних студентів.
- **Administrator (users з role='ADMIN')**: керує розкладом, залами (halls), напрямами (dance_styles) та транзакціями.
- **Monobank API (External Actor)**: зовнішній платіжний сервіс (створення сесій оплати, надсилання Signed Webhook).
- **Backend Domain Services**: APIGateway, BookingService, LessonService, PaymentService, Database (PostgreSQL).


## UC01: Booking and paying for lesson

### Scenario 1.1: Successfull booking and paying using Monobank API
**Initial conditions**: Student is authorised, Lesson has available slots ('current_bookings < capacity').

**Step by step explaining**:
1. Student sends request for booking exact 'lesson_id'.
2. 'APIGateway' identifies authorisation token and redirects this request to 'BookingService'.
3. 'BookingService' checks 'Lesson.max_capacity' and 'Hall.capacity' in 'Database'.
4. 'BookingService' creates entry in 'bookings' table with status 'pending' and fixes timestamp 'created_at'.
5. 'BookingService' refers to 'PaymentService' to initialise invoice.
6. 'Payment Service' calls external 'Monobank API' and gets 'pageUrl' and 'invoiceId'.
7. 'BookingService' returns 'pageUrl' to Student.
8. Student pays for lesson in his Monobank app.
9. 'Monobank API' sends asynchronous Webhook with ECDSA signature on backend.
10. 'PaymentService' validates ECDSA SHA-256 signature, updates entry status in 'bookings' on 'confirmed', decrements counter of available slots by one and sends message to Student.


### Scenario 1.2: attempt to book a slot for the lesson, which is full
**Initial conditions**: Lesson has 0 available space ('confirmed + pending >= capacity').

**Step by step explaining**:
1. Student sends request for booking exact 'lesson_id'.
2. 'APIGateway' identifies authorisation token and redirects this request to 'BookingService'.
3. 'BookingService' checks 'Lesson.max_capacity' and 'Hall.capacity' in 'Database'.
4. 'Database' return present status of lesson capacity: 0 slots are available.
5. 'BookingService' declines request for creating an entry in 'bookings' table.
6. Student gets HTTP 409 Conflict with localised message "Class is full".


### Scenario 1.3: cancelling booking after timeout or paying error
**Initial conditions**: 'booking' entry is created with status 'pending', but payment isn't done for 15 minutes (REQ-05, NFR-04).

**Step by step explaining**: 
1. Background service 'BookingTimeoutWorker' periodically checks unavailable slots.
2. 'BookingTimeoutWorker' detects entries in 'bookings', where 'status = pending' and 'now() - created_at > 15 minutes'.
3. 'BookingTimeoutWorker' calls 'BookingService' to cancel request.
4. State change / Webhook processing:
    - If Monobank returned status 'err_declined' or 15 minutes timeout is over:
    - 'BookingService' updates status in 'Database' on 'cancelled'.
    - Unavailable slot returns to access for other users.
    -   For the payment service sends a command to close invoice 'cancelInvoice()'.

### Scenario 1.4: student cancels booking

**Initial conditions**: student has booking with 'pending' or 'confirmed' status.

**Step by step explanation**:
1. Student clickes "Cancel booking" in his profile.
2. 'APIGateway' redirects request 'cancelBooking(booking_id)' to 'BookingService'.
3. 'BookingService' checks for booking status in 'Database'.
4. 'BookingService' changes status on 'cancelled'.
5. Amount of available slots for Lesson increments for 1.
6. Student gets message about successful cancelling and instructions for how to get a refund from Admin.