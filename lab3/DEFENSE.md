## 1. How sequence diagrams detail use case scenarios

The sequence diagrams transform high-level Gherkin scenarios into a concrete, production-ready technical workflow.

* **Microservice communication:** They break down user actions (like clicking "Book and pay") into explicit API calls passing through the 'API Gateway', 'Booking Service', and 'Payment Service`.
* **Error and alternative handling:** Using explicit conditional blocks ('alt', 'opt'), the diagrams clearly show how the system reacts to edge cases—such as when a class is fully booked, when a 15-minute payment timeout is reached, or when a payment webhook fails.
* **Concurrency and safety:** They detail how the system prevents race conditions during peak traffic (using database row locking) and handles background automated maintenance tasks (via the 'BookingTimeoutWorker`).


## 2. Alignment with the ER Model

The architecture shown in the diagrams is 100% bound to the relational database structure defined in ER model:

* **Direct entity mapping:** The objects and database queries on the diagrams directly manipulate the core tables ('lessons', 'halls', 'bookings', and 'payments').
* **State consistency:** The status values used across the diagrams ('pending_payment', 'confirmed', 'cancelled') strictly match the ENUM types defined in the 'Booking' and 'Payment' entities, ensuring data integrity in 3rd Normal Form (3NF).
* **Capacity rules:** The database queries in the diagrams correctly join 'lessons' and 'halls' to evaluate limits ('min(max_capacity, hall_capacity)'), ensuring that the physical constraints of the dance studio are strictly respected.