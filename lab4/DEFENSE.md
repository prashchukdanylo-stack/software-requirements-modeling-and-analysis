# Defense

## Intent and criteria 
Intent: model process 'Booking and paying for Lesson' (UC01, Scenarios 1.1-1.3) as Activity Diagram, so that it is visible who sends which message to whom and what happens when there is no answer. 
Criteria: there is at least 1 branching and 1 parallel branch, every message has sender, receiver and reaction to missing answer, there are no unreachable states and no 'holes' in transitions, and messages of the process match arrows of Sequence 1.1-1.3.

## Why such structure of steps
1. Activity Diagram was chosen, because process has actions of four actors, branching and one really concurrent part. State machine would show only states of 'Booking' and lose who sends what to whom. BPMN gives message flow too, but for this task it is too heavy. Mermaid was chosen, because it is allowed by the task and I could check the render.
2. Availability check goes first. If class is full, system returns 409 and nothing is created in BD.
3. 'Booking' with 'pending_payment' is created before invoice. 'createInvoice' needs bookingId, and Req-3 starts payment only after successful creation of 'Booking'. It is the same order as in Sequence 1.1. Lock of lesson row until commit blocks the same place for other students (Req-4).
4. Concurrency starts after Student gets pageUrl: Webhook can come at any moment and 'BookingTimeoutWorker' works independently. That is why fork is in step 5, and not in step 3.
5. Four swimlanes: Student, CRM-system, Monobank API and 'BookingTimeoutWorker'. CRM-system joins APIGateway, BookingService, PaymentService and Database from Sequence, because for business process only responsibility matters, not internal calls.

## How disputed places were solved
1. Fork without join: both branches finish the process by themselves. Join would wait for both, but they are competitors, not partners. Race is solved by check of 'Booking' status in BD (idempotency from UC 1.1).
2. Late Webhook after timeout: 'Booking' isn't restored, event is logged for manual refund by Administrator, HTTP 200 OK is returned (UC 1.1). Automatic refunds are out of scope (Lab1, Out of Scope).
3. Invalid signature: request is rejected with HTTP 400 without state changes. Waiting doesn't stop, because branch T is still working.
4. Capacity counter: it changes only when 'pending_payment' is created and when 'Booking' is cancelled, not at confirmation, otherwise one place would be counted twice.
5. 15 minutes and polling: 'BookingTimeoutWorker' polls every minute (Sequence 1.3), so real limit can be ~16 minutes, while NFR-04 says 'strictly 15 minutes'. It is not fixed in diagram, because diagram must follow Sequence. To satisfy NFR-04 polling interval should be shorter, or NFR-04 should say 'from 15 to 16 minutes'.
6. Failure of 'createInvoice': UC and Sequence don't describe it. Diagram follows Signal 3 and cancels 'Booking' after error or after N seconds without answer. Value N is not defined in requirements, so it is left as parameter.
7. Scenario 1.4: Student cancels Booking is a separate process of UC01, so it is not drawn here. If Student cancels 'Booking' while payment is pending, late 'success' Webhook is handled by branch W as late Webhook for 'cancelled' booking.
8. Status 'pending' (Lab1) vs 'pending_payment' (UC, Sequence): diagram uses 'pending_payment'; Req-1 and Gherkin should be updated.

## ADR
**Decision:** where to put the parallel part, and how to finish it.

**Context:** Task requires at least one parallel branch. The first idea (fork in booking creation) contradicts Sequence, so real concurrency had to be found in UC.

**Alternatives:**
1. Fork 'create Booking' || 'create invoice' (first version). Rejected: invoice needs bookingId, so there is a data dependency, and Req-3 demands order.
2. Fork 'update DB' || 'send notification' after Webhook. Rejected: Sequence shows them as consecutive steps of one transaction flow, so parallelism would be invented only to satisfy the task.
3. Fork after Student gets pageUrl: Branch W (Webhook processing) || Branch T ('BookingTimeoutWorker' polling). **Chosen:** in UC01 these are two really independent actors working at the same time.

**Second question — join or no join:** I chose no join. Branches are competitors, not partners: Join would wait for both, but when one branch changes 'Booking' to final status, the other must just stop. The conflict is solved by checking 'Booking' status in BD (idempotency in UC 1.1: duplicate Webhook, late Webhook after 'cancelled', worker finds booking not 'pending_payment').

**Consequences:** the diagram is consistent with Sequence 1.1-1.3. Price: Mermaid has no built-in fork, so it is drawn as a separate node, and because worker polls every minute, cancellation can be up to ~16 minutes after 'booked_at' (NFR-04 says 'strictly 15 minutes'). It is not fixed in diagram, because diagram must follow Sequence; fix is either shorter polling interval or NFR-04 wording 'from 15 to 16 minutes'.

**Form:** Activity Diagram (not BPMN, not State machine). State machine would show only states of 'Booking' and hide message exchange. BPMN gives message flow, but is too heavy for this task. Mermaid was chosen over PlantUML, because it is allowed by the task and I could render and check it.

## Check -- traceability with prevoius model (how checked)

1. Messages: compared every signal 1-11 of the process with arrows of Sequence 1.1, 1.2, 1.3 (sender, receiver, name). Signals 1, 3 (no-response branch) and 4 ('error') are not in Sequence, they are added by Activity and marked in Audit.
2. Statuses and rules: compared 'pending_payment / confirmed / cancelled', capacity condition 'confirmed + pending_payment < min(Lesson.max_capacity, Hall.capacity)', HTTP codes (409, 400, 200) and 'booked_at' with UC01 and Sequence. Found mismatches with Lab1 ('pending' instead of 'pending_payment', capacity '-1' at confirmation, notifications sent by Monobank) are listed in Audit, point 6.
3. Structure: checked by hand that every decision node has all outgoing branches, every flow reaches an end node, and the worker loop is finite. Diagram was rendered with Mermaid CLI without errors (diagram.png from diagram.mmd).
4. Not covered: Scenario 1.4 (Student cancels Booking) is a separate process and is not drawn; it is described in spec.md as 'Out of this process'.
