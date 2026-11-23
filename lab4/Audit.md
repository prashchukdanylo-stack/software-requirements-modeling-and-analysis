# Audit
1. Completeness of transitions: All nodes in diagram have clear input and output lines. There are no hanging states. For each branching (if/else) diagram describes all possible branches: available or unavailable slots, invoice created or not, valid or not valid signature, 'Booking' status (confirmed / cancelled / pending_payment), success or err_declined status of payment, 'Booking' still pending or not.
2. Reachability of states: All final states (End: no booking, End: rejected, End: confirmed, End: cancelled, End: Webhook handled, End: expiry handled, End: nothing to cancel, End: expired) are reachable from Start or from the FORK through valid business scenarios, processing errors or timeouts. There are no unreachable states in diagram. Loop of 'BookingTimeoutWorker' (tick -> not older than 15 minutes -> tick) is finite: it stops when 'Booking' leaves 'pending_payment' or when worker cancels it.
3. Correctness of conditions:
- Availability check: condition "confirmed + pending_payment < min(Lesson.max_capacity, Hall.capacity)" is correct and is valid to Req-2 and UC01 (Scenario 1.2).
- Check for Webhook signature: condition "Signature valid (ECDSA SHA-256)?" is valid to NFR-05. Branch "No" doesn't change 'Booking' and returns HTTP 400 to 'Monobank API', as described in UC01 and Sequence 1.1. Fake request can't confirm or cancel anything.
- Check of 'Booking' status in Webhook branch: 'confirmed' (duplicate Webhook) and 'cancelled' (late Webhook) don't change the state, so one Webhook can't be applied twice. For late 'success' event is logged for manual refund (Sequence 1.1).
- Timeout: condition "now - booked_at > 15 min" is valid to NFR-04 and Req-5. Because worker polls every minute, real cancellation can happen up to ~16 minutes after 'booked_at'. It is not fully strict with NFR-04 (see Top-3 and DEFENSE).
4. Parallel branches and synchronisation:
- Block "fork" after Student gets pageUrl divides process into 2 independent branches: Branch W (processing of Webhook from 'Monobank API') and Branch T ('BookingTimeoutWorker' looks for expired 'Booking') (Req-5, NFR-04).
- Branches have no join on purpose: they compete for the same 'Booking', and each ends by its own end node. Check of 'Booking' status in BD decides who changes it first, so there is no double change (idempotency).
- Creating 'Booking' and creating invoice are NOT parallel: invoice needs bookingId, so order is strict (Req-3, Sequence 1.1).
5. Signals and events (who sends, who accepts, what happens without answer):

- 1. Student sends signal 'query for booking a place at Lesson' to CRM-system, if there is no response, 'Student' sees a notification about 'connection error' or 'timeout', BD isn't changed.
- 2. CRM-system sends signal 'error 409: Class is full' to Student, process finishes, BD isn't changed.
- 3. CRM-system sends signal 'createInvoice' to Monobank API, if there is no response in N seconds, 'Booking' status becomes 'cancelled', slot gets available, Student gets notification.
- 4. Monobank API sends signal 'invoice (invoiceId, pageUrl)' or 'error' to CRM-system, if it is 'error', 'Booking' status becomes 'cancelled'.
- 5. CRM-system sends signal '201 Created (bookingId, pageUrl)' to Student, if Student doesn't pay during 15 minutes, 'BookingTimeoutWorker' starts cancelling of 'Booking'.
- 6. Student sends signal 'pay(invoiceId)' to Monobank API, Monobank API answers 'payment accepted'. If Webhook doesn't come, 'Booking' is cancelled by 'BookingTimeoutWorker'.
- 7. Monobank API sends signal 'asynchronous Webhook with transaction status' to CRM-system, if there is no successful Webhook after 15 minutes, status is changed to 'cancelled', a slot gets available.
- 8. CRM-system sends signal 'HTTP 400 / HTTP 200 OK' to Monobank API as answer on Webhook, if Monobank doesn't receive it, behavior (retry) is not described in UC and Sequence.
- 9. CRM-system sends signal 'cancelInvoice(invoiceId)' to Monobank API after declined payment or timeout, if there is no response, 'Booking' is already 'cancelled', invoice can stay open (not described in UC and Sequence).
- 10. BookingTimeoutWorker sends signal 'cancelExpiredBookings' to CRM-system, CRM-system answers 'cleanupCompleted', if there is no answer, next tick repeats the check.
- 11. CRM-system sends signal 'notification about confirmed / declined / expired / cancelled booking' to Student, if Student doesn't receive it, status of 'Booking' in BD is already final and isn't changed.

6. Traceability with UC (Lab2) and Sequence (Lab3) and Top-3 Mismatches:
- The status space (pending_payment, confirmed, cancelled), capacity validation rule, Webhook logic (400 / idempotency / late Webhook), 'cancelInvoice' and 15-minute timeout match UC01 (Scenarios 1.1, 1.2, 1.3) and Sequence 1.1, 1.2, 1.3. Messages 1-11 have the same sender, receiver and meaning as arrows in Sequence, except points marked 'not described' (they are added by Activity diagram).
- Top-3 mismatches with UC and Sequence and solutions:
  1. Fork in booking creation: Issue: first version of the process had fork 'create Booking' || 'create payment session'. Sequence 1.1 and Req-3 show strict order: 'saveBooking' first, then 'createInvoice(reference = bookingId)'. Invoice needs bookingId, so branches can't be parallel. Resolution: step 3 was made sequential, fork was moved to step 5, where concurrency is real (Webhook processing || 'BookingTimeoutWorker').
  2. Timeout and cancelInvoice: Issue: first version had abstract '15-minute timer' inside CRM-system and cancelled 'Booking' without call to 'Monobank API'. UC 1.3A and Sequence 1.3 use separate 'BookingTimeoutWorker' (polling every minute) and send 'cancelInvoice' to Monobank. Resolution: added swimlane 'BookingTimeoutWorker', signals 'cancelExpiredBookings', 'cleanupCompleted' and 'cancelInvoice'. Open issue: polling every minute gives up to ~16 minutes, but NFR-04 says 'strictly 15 minutes'.
  3. Webhook handling: Issue: first version ignored Webhook with invalid signature, had no answer to Monobank, no check for duplicates and no late Webhook. UC 1.1 requires HTTP 400 for invalid signature, HTTP 200 OK without changes for duplicates and log for manual refund for late 'success'. Resolution: added all these branches and signal 'HTTP 400 / HTTP 200 OK' to Monobank API.
- Other mismatches (not in Top-3):
  - Status name: Req-1 and Gherkin use 'pending', UC and Sequence use 'pending_payment'. Diagram follows UC and Sequence, Req-1 should be updated.
  - Timestamp name: NFR-04 says 'creation timestamp', UC says 'booked_at'. Diagram uses 'booked_at'.
  - Capacity counter: Gherkin Scenario 1 says that remaining capacity becomes less by 1 after confirmation, but Req-2 and UC count 'pending_payment' as active already. Diagram doesn't change the counter on confirmation.
  - Failure of 'createInvoice': UC and Sequence describe only successful invoice. Diagram adds branch 'error / no response -> cancelled'. UC01 should get this alternative flow.
  - Notifications: Lab1 says that Monobank API sends notifications to students, in Sequence they are sent by 'PaymentService' (CRM-system). Diagram follows Sequence.
  - Microservice decomposition: Sequence shows APIGateway, BookingService, PaymentService and PostgreSQL separately, Activity diagram groups them into one swimlane CRM-system, because here only business responsibility matters.
  - Scenario 1.4 (Student cancels Booking) is a separate process of UC01 and isn't drawn in this diagram.
