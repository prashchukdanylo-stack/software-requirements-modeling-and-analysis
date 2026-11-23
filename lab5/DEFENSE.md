# DEFENSE: Component Model

## 1. Place in the chain
Lab1 (data) → Lab2 (requirements) → Lab3 (interaction) → Lab4 (process) → Lab5 (component architecture).

The component diagram puts all four previous models into one structure. Every component has an owner in the models before, every dependency matches a message in the sequence scenarios (except explicit goal-only dependencies marked `(D-1)`), and every entity has a component which owns it.

## 2. How the model integrates previous labs
- **Lab1 (ER):** 8 entities and the ENUM for statuses. Booking belongs to Booking Service. Payment belongs to Payment Service. Student and Administrator belong to Identity Service. Lesson, Hall, Instructor and DanceStyle belong to Schedule Service (while Booking Service checks slot availability and manages booking records). Everything goes through Data Access Layer to PostgreSQL.
- **Lab2 (Requirements):** actors, EARS Req-1…6, NFR-01…07, Gherkin. Examples: Web UI covers NFR-01 and NFR-06, API Gateway covers NFR-02, NFR-03 and TLS from NFR-05, Payment Service covers the ECDSA signature from NFR-05, BookingTimeoutWorker covers NFR-04, and Docker, Node.js and PostgreSQL cover NFR-07.
- **Lab3 (Interaction):** participants and messages of scenarios 1.1–1.4. Every message became a dependency with a named interface ('validateToken', 'createInvoice', 'cancelInvoice', 'sendMessage', 'cancelBookingByUser'). The webhook goes directly to Payment Service.
- **Lab4 (Process):** lanes and steps. Lane "Monobank API" became X-01, BookingTimeoutWorker became a separate component, lane "CRM-system" was split into API Gateway, Booking, Payment and Notification.

## 3. What is agreed
- The main scenario (booking, payment, timeout, declined payment, cancelling) is traced from start to end: Req-1…5, G1–G4, Lab3 1.1–1.4, Lab4 steps 1–7, components C-05, C-06, C-07 and X-01.
- The boundary is clear and there is only one external system, Monobank API. SMS, Email, OAuth and Calendar were not invented.
- All 10 internal/external interfaces have a provided side and a required side, with no dangling interfaces.
- The route of the webhook (directly to Payment Service, signature check there) is the same in Lab2 NFR-05, Lab3 1.3B and the architecture.

## 4. Intent & Criteria:
Designing the component-level architecture (C-01…C-10) and external interfaces for the Dance Studio CRM to ensure full coverage of booking and payment lifecycles (Lab 1–4), data access isolation through DAL to PostgreSQL, and secure handling of asynchronous entry points (NFR-05).

## 5. What mismatches the audit found
1. Cyclic Dependency (Booking ⇄ Payment): Resolved mutual service calls between Booking Service and Payment Service by removing 'IBookingState' and reverse calls from Payment to Booking in favor of one-way coordination via 'IPayment'.
The Resolution: Kept coordination one-way from Booking Service to Payment Service through 'IPayment' ('createInvoice', 'getStatus', 'cancelInvoice') without a reverse 'IBookingState' callback. <cdd7bc1>
2. Shared Reference to Lesson and Hall Across Two Services: Resolved ambiguity over primary ownership between Schedule Service and Booking Service, which both referenced Lesson and Hall in earlier traceability notes.
The Resolution: Confirmed that the component interfaces already separate ownership ('ISchedule' in Schedule Service vs. 'IBooking' in Booking Service), and explicitly documented in 'spec.md' and 'traceability.md' that Schedule Service owns Lesson, Hall, Instructor, and DanceStyle, while Booking Service only checks slot availability and manages Booking records. <5f518f9>
3. Goal-Only Dependencies Without Sequence Steps (D-1): Addressed dependencies such as API Gateway to Schedule Service and admin API Gateway to Payment Service that are derived from actor goals in Lab2 §1.1 rather than explicit message steps in Lab3 sequence diagrams.
The Resolution: Defined and applied the `(D-1)` marker in 'traceability.md' and documented in 'spec.md' that these dependencies trace directly to Lab2 §1.1 goals without inventing non-existent Lab3 sequence steps. <5f518f9>

### 6. ADR — removing Booking--Payment cycle:
- **Alternatives:** (A) Event-driven 'PaymentConfirmed'/'PaymentFailed' events consumed by 'Booking' — pros: loose coupling; cons: extra event contracts and async state complexity. (B) One-way coordination via 'Booking' calling 'IPayment' without reverse 'IBookingState' — pros: removes cyclic dependency and keeps call flow direct; cons: 'Booking' coordinates status checks/timeouts directly.
- **Decision:** Selected **Option B** to eliminate the cycle with minimal structural overhead. <cdd7bc1>

## 7. Verification & Traceability:
Validated end-to-end trace chains in 'traceability.md': mapped ER entities (Lab 1) and EARS requirements (Lab 2) directly to sequence steps (Lab 3) and process activities (Lab 4), marked goal-only dependencies as `(D-1)`, and kept 'spec.md' concise while moving detailed traceability to 'traceability.md'. <5f518f9>