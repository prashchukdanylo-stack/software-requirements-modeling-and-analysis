# Choice of form

For modelling of my business process i chose UML Activity Diagram with PlantUML syntax.
## Why I chose it:
1. PLantUML gives its users built in graphical primitives for sending (send signal >) and accepting (accept event <), which is a critical requirement for modelling asynchronous integrations.
2. Clear borders of responsibility (swimlanes): |Student|, |CRM-system|, |Monobank API| clearly divide steps between actors.
3. Block fork / fork again / end fork lets correctly demonstrate simultaneous blocking of slots in BD (pending_payment) and initialization of payment session in Monobank API (Req-3, Req-4), and blocks if/else check for slots availability and processing 15-minute timeout (Req-2, Req-5);

# Participants of process
1. Student: main user of studio. Chooses lesson, initiates booking, approves payment operations in Monobank app.
2. CRM-system: checks for available slots (Hall.capacity and Lesson.max_capacity), operates lifecycle of 'Booking' (from pending_payment to confirmed/cancelled), verifies Webhook signature and controls 15-minute payment window (NFR-04, NFR-05).
3. Monobank API: External service that generates payment session, processes Student's card transaction and sends Webhook with payment status.

# Steps, messages, branching
## Steps
1. Booking initialisation: Student chooses active Lesson and clicks "Book and pay", sending a query to CRM-system.
2. Avaialbility checking: CRM-system checks, if amount of active bookings (confirmed + pending_payment) is less than Hall.capacity and Lesson.max_capacity.
- Branch "No"(no available slots): System rejects query with error "Class is full" (409 Conflict), tells Student about it and finishes the process. Query to BD isn't created.
- Bramch "Yes" (there is some available place): Process goes to block of parallel execution (fork).
3. Parallel processing (fork -- Req-1, Req-3, Req-4):
- Branch A (Blocking slot in BD): CRM-system creates query 'Booking' with 'pending_payment' status, fixates creation time to count 15 minutes (NFR-04), blocking this place from booking by other students.
- Branch B (payment initialisation): CRM-system sends request to create payment session to 'Monobank API'. External service generates a link for payment and return it to CRM.
4. Redirecting to link and payment: after flows' synchronisation (end fork) CRM-system redirects Student to Monobank payment page. Student approves payment in Monobank app.
5. Processing the payment result (fork -- Req-5, NFR-05):
- CRM-system waits for Webhook from 'Monobank API' or end of 15 minutes from the moment of creation 'Booking'.
- Branch 'Successful payment': valid Webhook was recieved (with check of SHA-256/ECDSA) with 'success' status. CRM-system changes 'Booking' status to 'confirmed', reduces amount of available slots by 1 and sends notification to 'Student' about successful registration for 'Lesson'.
- Branch 'Error or 15-minute timeout': if 'Monobank API' returned transaction error or 15 minutes passed from creation of 'Booking' without successful Webhook, CRM-system changes 'Booking' status on 'cancelled' and releases the blocking of slot.

## Signals and events

1. Student sends signal 'query for booking a place at Lesson' to CRM-system, if there is no response, 'Student' sees a notification about 'connection error' or 'timeout', BD isn't changed.
2. CRM-system sends signal 'query for payment session creation' to Monobank API, if there is no response, service isn't available or responses with error, 'Booking' status becomes 'cancelled'.
3. Student sends signal 'transaction approvement in Monobank app' to Monobank API, if Student doesn't pay during 15-minute gap, CRM-system cancels Booking.
4. Monobank API sends signal 'asynchronous Webhook with transaction status' to CRM-system, if there is no successful Webhook after 15 minutes, staatus is changed to 'cancelled', a slot gets available.
