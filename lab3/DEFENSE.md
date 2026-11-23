
## Intent and criteria
Creation of detailed sequence diagrams for use case scenarios from prior labs, that mostly corelate with Gherkin  logic, ER structure and with spec in this lab.


## Top 3 mismatches
**1. Attribute, which doesn't exist in ER:**
In diagram 1.4 there was an attribute, that wasn't actually there in ER, and amount of free slots counted differently in 1.1 and 1.2.
Resolution: estimate count of active bookings with confirmed + pending_payment using countActiveBookings.
*(Commits: `c5acbfd`, `ecc3791`)*

**2. Extra participant and low level SQL:**
In 1.4 there was a lifeline of Booking Timeout Worker without any messages, and in diagrams instead of abstract easy-to-read methods, I have written raw SQL queries.
Resolution: removed extra lifeline from 1.4, replaced raw SQL with calls like 'getLessonCapacity', 'saveBooking' etc.
*(Commits: `c5acbfd`, `ecc3791`, `c735494`, `452c0d5`)*

**3. Incorrect usage of deactivation and message arrows:**
In 1.2 diactivate was only in one alt branch, and in 1.3 Student got messages using '-->'
Resolution: make deactivation after 'end', changed incorrect arrows on '->'.
*(Commits: `c735494`, `452c0d5`)*

## ADR
 **Alternative 1:** to process 15-minute payment timeout using delayed Events, or queue of messages.
 **Alternative 2:** periodical pull from Database using Booking Timeout Worker.

 **Choice:** I chose 2 alternative, because it is easier to implement, personally.

 ## Verification
 Checked everywhere status names ('pending_payment, 'confirmed', 'cancelled'), table fields ('max_capacity', 'hall_capacity').