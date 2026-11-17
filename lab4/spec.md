# Choice of form

For modelling of my business process I chose UML Activity Diagram with Mermaid syntax (flowchart with swimlanes).
## Why I chose it:
1. Activity Diagram shows what actors do and who sends what to whom. State machine would describe only lifecycle of 'Booking' and hide the message exchange, which is a critical requirement for modelling asynchronous integrations.
2. Messages have their own notation: sending (send signal) is drawn with flag-shaped node (>), accepting (accept event) is a node with prefix 'ACCEPT:'. Asynchronous Webhook and 15-minute timer are modelled as accepted events.
3. Clear borders of responsibility (swimlanes): Student, CRM-system, Monobank API clearly divide steps between actors.
4. Nodes FORK / JOIN let correctly demonstrate simultaneous blocking of slots in BD (pending_payment) and initialization of payment session in Monobank API (Req-3, Req-4). Mermaid has no built-in fork/join, so they are drawn as separate nodes. Blocks with conditions (diamonds) show check for slots availability, check of Webhook signature and processing 15-minute timeout (Req-2, Req-5).
5. Diagram is stored as text (diagram.mmd), so it is declarative and the render (diagram.png) is generated from the same source.

# Participants of process
1. Student: main user of studio. Chooses lesson, initiates booking, approves payment operations in Monobank app.
2. CRM-system: checks for available slots (Hall.capacity and Lesson.max_capacity), operates lifecycle of 'Booking' (from pending_payment to confirmed/cancelled), verifies Webhook signature and controls 15-minute payment window (NFR-04, NFR-05).
3. Monobank API: External service that generates payment session, processes Student's card transaction and sends Webhook with payment status.

# Steps, messages, branching
## Steps
1. Booking initialisation: Student chooses active Lesson and clicks "Book and pay", sending a query to CRM-system.
- If there is no response, Student sees 'connection error' or 'timeout', BD isn't changed and the process finishes.
2. Availability checking: CRM-system checks, if amount of active bookings (confirmed + pending_payment) is less than Hall.capacity and Lesson.max_capacity.
- Branch "No" (no available slots): System rejects query with error "Class is full" (409 Conflict), tells Student about it and finishes the process. 'Booking' isn't created in BD.
- Branch "Yes" (there is some available place): Process goes to block of parallel execution (fork).
3. Parallel processing (fork -- Req-1, Req-3, Req-4):
- Branch A (Blocking slot in BD): CRM-system creates 'Booking' with 'pending_payment' status, fixates creation time (created_at) to count 15 minutes (NFR-04), blocking this place from booking by other students.
- Branch B (payment initialisation): CRM-system sends request to create payment session to 'Monobank API'. External service generates a link for payment and returns it to CRM, or returns an error.
- If 'Monobank API' returned an error or didn't respond in N seconds, the process doesn't go to join: 'Booking' status changes to 'cancelled', the blocking of slot is released and Student gets notification about cancellation.
4. Redirecting to link and payment: after flows' synchronisation (join) CRM-system redirects Student to Monobank payment page. Student approves payment in Monobank app, and Monobank API processes the card transaction and sends Webhook to CRM-system.
5. Processing the payment result (Req-5, NFR-05):
- CRM-system waits for the first of two events: Webhook from 'Monobank API' or end of 15 minutes from the moment of creation 'Booking'.
- Branch 'Invalid signature': Webhook without valid SHA-256/ECDSA signature is ignored and logged, CRM-system continues to wait (timer keeps running).
- Branch 'Successful payment': valid Webhook (with check of SHA-256/ECDSA) with 'success' status was received. CRM-system changes 'Booking' status to 'confirmed' and sends notification to 'Student' about successful registration for 'Lesson'. The place is already counted since 'pending_payment', so amount of available slots isn't changed again.
- Branch 'Error or 15-minute timeout': if valid Webhook has transaction error or 15 minutes passed from creation of 'Booking' without successful Webhook, CRM-system changes 'Booking' status to 'cancelled', releases the blocking of slot and sends notification to 'Student' about cancellation.

## Signals and events

1. Student sends signal 'query for booking a place at Lesson' to CRM-system, if there is no response, 'Student' sees a notification about 'connection error' or 'timeout', BD isn't changed.
2. CRM-system sends signal 'error 409: Class is full' to Student, process finishes, BD isn't changed.
3. CRM-system sends signal 'query for payment session creation' to Monobank API, if there is no response in N seconds, service isn't available or responds with error, 'Booking' status becomes 'cancelled', slot gets available.
4. Monobank API sends signal 'payment link' (or 'error') to CRM-system, if it is 'error', 'Booking' status becomes 'cancelled'.
5. CRM-system sends signal 'redirect to payment link' to Student, if Student doesn't pay during 15-minute gap, CRM-system cancels Booking.
6. Student sends signal 'transaction approvement in Monobank app' to Monobank API, if Monobank API doesn't send Webhook during 15-minute gap, CRM-system cancels Booking.
7. Monobank API sends signal 'asynchronous Webhook with transaction status' to CRM-system, if there is no successful Webhook after 15 minutes, status is changed to 'cancelled', a slot gets available.
8. CRM-system sends signal 'notification about successful booking' or 'notification about cancellation' to Student, if Student doesn't receive it, status of 'Booking' in BD is already final and isn't changed.
