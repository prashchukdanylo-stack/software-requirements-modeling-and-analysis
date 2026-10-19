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

## EARS Requirements

* **Req-1(ubiquitous):** system SHALL ensure keeping mandatory relation among entites Student, Lesson and created Booking registration in data base.
* **Req-2(event-driven):** WHEN Student tries too book a Lesson, system SHALL check for Hall capacity and create payment registration in Booking table with "pending" status.
* **Req-3(event-driven):** WHEN Booking creation is succesfull with status "pending", system SHALL initialise payment session using Monobank API and generate link for payment.
* **Req-4(state-driven):** WHILE system is processing payment creation, it SHALL block an ability to book that same place at the same Lesson, so there aren't any problems.
* **Req-5(unwanted event):** IF Monobank API returns a transaction error, or time of exceptancy is over, THEN system SHALL change Booking status for "cancelled" and stop blocking that place from being booked.
* **Req-6(event_driven):** WHEN Administrator creates or updates classes timetable, system SHALL check if all relations among Hall, Instructor and DanceStyle aro correct.

## User Stories

* **UserStory-1 (booking and paying):** as a Student, I want to choose a Lesson, book a place for it and pay for it with my Monobank card, so that I am surely registered.
* **UserStory-2 (searching and filtering):** as a Student, I want to look through the timetable and filter Lessons by DanceStyles and Instructors.
* **UserStory-3 (checking students list):** as a Instructor, I want to check a list of all Students, applied for my Lesson to have a qualitative plan of it.
* **UserStory-4 (planning and timetable):** as a Administrator, I want to create new Lessons, relate them to Halls, DanceStyles and Instructors to maintain useful timetable for all users of studio.\
* **UserStory-5 (cancelling):** as a Student, I want to have an ability to cancel my Booking before Lesson starts, so other people can join.  
