## Actors
* **Student:** looks for a class to visit, books it, main user of dance studio.
Main goals in system: register and manipulate with personal profile;
look through classes timetable, with ability to filter them by dance style and instructors;
create, cancel, check bookings whenever it is needed.

* **Instructor:** teaches students, runs dance classes.
Main goals in system: look for classes timetable to understand when to start lesson and in which hall;
have an access to list of applied students, to understand how many people is going to be there.
* **Administrator:** head of studio, responsible for managing everything about classes, instructors, control.
Main goals in system: add new halls, instructors and dance styles to the system;
create and plan timetable for lessons;
moderate all booking processes, cope with financial issues.
* **Monobank API**: external service.
Main goals in system: process requests for booking lessons, return transaction status after;
send students notifications about success/fail of their bookings. 
