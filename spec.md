## Entites and their atributes
* **Student:** id: UUID(PK), first_name: string, last_name: string, email: string, phone: string, registered_at: datetime.
* **Instructor:** id: UUID(PK), first_name: string, last_name: string, email: string, phone: string, bio: string, hired_at: datetime.
* **DanceStyle:** id: UUID(PK), name: string, description: string.
* **Hall:** id: UUID(PK), name: string, capacity: int.
* **Lesson:** id: UUID(PK), instructor_id: UUID(FK), hall_id: UUID(FK), dance_style_id: UUID(FK), start_time: datetime, end_time: datetime, max_capacity: int
* **Booking** id: UUID(PK),student_id: UUID(FK), lesson_id: UUID(FK), booked_at: datetime, status: string.

## Relationships
* **Instructor — DanceStyle(N:M):** Many to many, because one instructor can teach many styles (1..N), and one style can be taught by many instructors(1..N).
* **Hall — Lesson(1:N):** one hall can have zero or many lessons (0..N), but one lesson can be only in 1 hall(1..1).
* **Instructor — Lesson(1:N):** one instructor can have zero or many lessons(0..N), but one lesson is taught by one instructor(1..1).
* **DanceStyle — Lesson(1:N):** one dance style can be taught on zero or many lessons(0..N), but one lesson teaches one dance style(1..1).
* **Student — Lesson(N:M through associative entity Booking):** One student can place zero or many bookings (`Student 1 ||--o{ Booking`).
One lesson can contain zero or many bookings (`Lesson 1 ||--o{ Booking`).
Each booking strictly belongs to one student and one lesson, containing its own attributes (`booked_at`, `status`).

## Acceptance Criteria

All keys are strictly UUIDs;
attributes use "snake_case", entity names use "PascalCase";
attributes are atomic, no partial dependencies on keys and no transitive dependencies;
pure relations.