# DEFENSE: Component Model

## 1. Place in the chain
Lab1 (data) → Lab2 (requirements) → Lab3 (interaction) → Lab4 (process) → Lab5 (component architecture).

The component diagram puts all four previous models into one structure. Every component has an owner in the models before, every dependency matches a message in the sequence scenarios, and every entity has a component which owns it.

## 2. How the model integrates previous labs
* **Lab1 (ER):** 8 entities and the ENUM for statuses. Booking, Lesson and Hall belong to Booking Service. Payment belongs to Payment Service. Student and Administrator belong to Identity Service. Lesson, Hall, Instructor and DanceStyle belong to Schedule Service. Everything goes through Data Access Layer to PostgreSQL.
* **Lab2 (Requirements):** actors, EARS Req-1…6, NFR-01…07, Gherkin. Examples: Web UI covers NFR-01 and NFR-06, API Gateway covers NFR-02, NFR-03 and TLS from NFR-05, Payment Service covers the ECDSA signature from NFR-05, Data Access Layer covers AES-256, BookingTimeoutWorker covers NFR-04, Docker, Node.js and PostgreSQL cover NFR-07.
* **Lab3 (Interaction):** participants and messages of scenarios 1.1–1.4. Every message became a dependency with a named interface ('validateToken', 'createInvoice', 'cancelInvoice', 'sendMessage', 'cancelBookingByUser'). The webhook goes directly to Payment Service.
* **Lab4 (Process):** lanes and steps. Lane "Monobank API" became X-01, BookingTimeoutWorker became a separate component, lane "CRM-system" was split into API Gateway, Booking, Payment and Notification.

## 3. What is agreed
* The main scenario (booking, payment, timeout, declined payment, cancelling) is traced from start to end: Req-1…5, G1–G4, Lab3 1.1–1.4, Lab4 steps 1–7, components C-05, C-06, C-07 and X-01.
* The boundary is clear and there is only one external system, Monobank API. SMS, Email, OAuth and Calendar were not invented.
* All 11 interfaces have a provided side and a required side.
* The route of the webhook (directly to Payment Service, signature check there) is the same in Lab2 NFR-05, Lab3 1.3B and the architecture.


## 4. Intent & Criteria:
Designing the component-level architecture (C-01…C-10) and external interfaces for the Dance Studio CRM to ensure full coverage of booking and payment lifecycles (Lab 1–4), strict data isolation via DAL (NFR-07), and secure handling of asynchronous entry points (NFR-05).

## 5. What mismatches the audit found
1. Bypassing API Gateway (Webhook Bypass): Addressed the discrepancy between the single-entry-point rule and async callbacks by routing IWebhook directly to the Payment Service protected by ECDSA signature verification (NFR-05).
The Resolution: Bypassed the API Gateway for external webhooks by routing Monobank API directly to the IWebhook interface provided exclusively by the Payment Service. <ca63b5b>
2. Circular Dependency (Booking ⇄ Payment): Resolved mutual service calls during invoice confirmation and timeout flows by extracting a dedicated state-coordination interface (IBookingState).
The Resolution: Decoupled the microservices by introducing the IBookingState interface. The Booking Service provides this state contract, allowing the Payment Service to safely trigger status transitions without tightly coupling to internal database schemas or business logic. <992f402>
3. Notification Ambiguity: Resolved vague delivery requirements from earlier specs by formalizing the internal Notification Adapter (C-08) to encapsulate student contact handling.
The Resolution: Centralized notification handling within the Notification Adapter, which uses student contact information (email and phone) to send messages. The external delivery mechanism remains unspecified, as no external notification provider was defined in previous laboratory works (because of it, there is nothing to fix).
### 6. ADR — removing Booking--Payment cycle:
- **Alternatives:** (A) Event-driven 'PaymentConfirmed'/'PaymentFailed' events consumed by 'Booking' — pros: loose coupling; cons: extra event contracts and async state complexity. (B) One-way coordination via 'Booking' calling 'IPayment' — pros: removes cyclic dependency and keeps call flow direct; cons: 'Booking' coordinates status checks/timeouts directly.
- **Decision:** Selected **Option B** to eliminate the cycle with minimal structural overhead.

## 7. Verification & Traceability:
Validated end-to-end trace chains: mapped ER entities (Lab 1) and EARS requirements (Lab 2) directly to sequence steps (Lab 3) and process activities (Lab 4), ensuring zero dangling interfaces and full structural consistency.