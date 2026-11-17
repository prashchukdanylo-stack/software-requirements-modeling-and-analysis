# Choice of form

For modelling of my business process I chose UML Activity Diagram with Mermaid syntax (flowchart with swimlanes).
## Why I chose it:
1. Activity Diagram shows what actors do and who sends what to whom. State machine would describe only lifecycle of 'Booking' and hide the message exchange, which is a critical requirement for modelling asynchronous integrations.
2. Messages have their own notation: sending (send signal) is drawn with flag-shaped node (>), accepting (accept event) is a node with prefix 'ACCEPT:'. Asynchronous Webhook and 15-minute timer are modelled as accepted events.
3. Clear borders of responsibility (swimlanes): Student, CRM-system, Monobank API, BookingTimeoutWorker clearly divide steps between actors.
4. Nodes FORK / JOIN let correctly demonstrate simultaneous blocking of slots in BD (pending_payment) and initialization of payment session in Monobank API (Req-3, Req-4). Mermaid has no built-in fork/join, so they are drawn as separate nodes. Blocks with conditions (diamonds) show check for slots availability, check of Webhook signature and processing 15-minute timeout (Req-2, Req-5).
5. Diagram is stored as text (diagram.mmd), so it is declarative and the render (diagram.png) is generated from the same source.

# Participants of process
1. Student: main user of studio. Chooses lesson, initiates booking, approves payment operations in Monobank app.
2. CRM-system: checks for available slots (Hall.capacity and Lesson.max_capacity), operates lifecycle of 'Booking' (from pending_payment to confirmed/cancelled), verifies Webhook signature (NFR-04, NFR-05).
3. BookingTimeoutWorker: Background microservice/worker component. Periodically polls the database (every 1 minute) to detect pending reservations that have exceeded the 15-minute TTL (now() - booked_at > 15 min), automatically revoking them and triggering invoice cancellation.
4. Monobank API: External service that generates payment session, processes Student's card transaction and sends Webhook with payment status.

# Steps, messages, branching
## Steps
1. Booking initialisation: Student chooses active Lesson and clicks "Book and pay", sending a query to CRM-system. If there is no response, Student sees 'connection error' or 'timeout', BD isn't changed and the process finishes.
2. Availability checking: CRM-system checks if the amount of active bookings (confirmed + pending_payment) is less than Hall.capacity and Lesson.max_capacity.
- Branch "No" (no available slots): System rejects query with error 409 Conflict (Class is full), tells Student about it and finishes the process. 'Booking' is not created in BD.
- Branch "Yes" (there is a free place): Process proceeds to booking and payment initialization.
3. Booking creation in DB: CRM-system creates a record in the bookings table with status pending_payment, fixes the creation timestamp (created_at = now), and blocks the slot from being booked by other students (Req-1, Req-4).
4. Payment session initialization: CRM-system sends a request (createInvoice) to Monobank API passing the bookingId as a reference (Req-3). If Monobank API returns an error or does not respond in N seconds, the transaction is rolled back, the Booking status becomes cancelled, the slot is released, and the Student gets a notification.
5. Redirecting to payment: Upon receiving the payment link (pageUrl), the CRM-system redirects the Student to the Monobank payment page.
6. Parallel processing / Monitoring (FORK): After redirecting, the process splits into two independent asynchronous paths (NFR-04, Req-5):
- Branch A (CRM-system): Waits for an incoming asynchronous Webhook from Monobank API.
- Branch B (BookingTimeoutWorker): A background worker loops every 1 minute to check for expired unpaid reservations (now() - booked_at > 15 min).
7. Processing the payment result or timeout (Req-5, NFR-05):
- Branch 'Invalid Webhook signature': If a Webhook arrives without a valid SHA-256/ECDSA signature (NFR-05), it is ignored and logged, the CRM-system responds with HTTP 400 Bad Request to Monobank API, and continues to wait.
- Branch 'Idempotent duplicate (already confirmed)': If a duplicate Webhook arrives for a booking that is already confirmed, the CRM-system does not mutate the state and responds with HTTP 200 OK to ensure safe gateway retries.
- Branch 'Late Webhook after timeout (already cancelled)': If a valid 'success' Webhook arrives after the 15-minute timeout has already cancelled the booking, the CRM-system keeps the cancelled state, logs the event for a manual administrative refund (manual refund), and responds with HTTP 200 OK.
- Branch 'Successful payment (normal flow)': A valid Webhook with success status is received for a pending_payment booking. CRM-system updates Booking status to confirmed and sends a success notification to the Student (along with HTTP 200 OK to Monobank).
- Branch 'Payment error (err_declined) or 15-minute timeout': If Monobank returns an error status or the BookingTimeoutWorker detects that 15 minutes have passed, CRM-system updates Booking status to cancelled, invokes cancelInvoice to close the invoice, releases the slot, and sends a cancellation notice.
## Signals and events

1. Student sends signal 'query for booking a place at Lesson' to CRM-system, if there is no response, 'Student' sees a notification about 'connection error' or 'timeout', BD isn't changed.
2. CRM-system sends signal 'error 409: Class is full' to Student, process finishes, BD isn't changed.
3. CRM-system sends signal 'query for payment session creation' to Monobank API, if there is no response in N seconds, service isn't available or responds with error, 'Booking' status becomes 'cancelled', slot gets available.
4. Monobank API sends signal 'payment link' (or 'error') to CRM-system, if it is 'error', 'Booking' status becomes 'cancelled'.
5. CRM-system sends signal 'redirect to payment link' to Student, if Student doesn't pay during 15-minute gap, CRM-system cancels Booking.
6. Student sends signal 'transaction approvement in Monobank app' to Monobank API, if Monobank API doesn't send Webhook during 15-minute gap, CRM-system cancels Booking.
7. BookingTimeoutWorker sends signal to CRM-system / Monobank API: cancelExpiredBookings & cancelInvoice(invoiceId) when the 15-minute window expires.
8. Monobank API sends signal 'asynchronous Webhook with transaction status' to CRM-system, CRM-system responds with HTTP 200 OK(for successful, idempotent, or late webhook) or HTTP 400 Bad Request (for invalid signatures).
9. CRM-system sends signal 'notification about successful booking' or 'notification about cancellation' to Student, if Student doesn't receive it, status of 'Booking' in BD is already final and isn't changed.
