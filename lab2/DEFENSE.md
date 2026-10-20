 ### Why the scenarios are split this way
 We split the system into distinct steps to make it reliable and clean:
 * **Booking (UC_Book) and Payment (UC_Pay) are separate:**
 Creating a booking checks if there are free seats in the hall. Paying talks to the Monobank API. Keeping them separate means if Monobank goes down, our internal system still works correctly and doesn't crash.
 * **Payment Confirmation (UC_Webhook) works in the background:**
 Monobank sends a notification (webhook) to our system when the payment succeeds. This is separate from the UI so that if a student accidentally closes the browser tab while paying, their booking still gets confirmed.
 * **Clear Rules for Cancellation (UC_Cancel):** 
 A booking goes through simple steps: pending_payment -- confirmed or cancelled. If a student doesn't pay within 15 minutes, the system automatically cancels the booking and frees up the seat for others.
 ## Why Backup is Not a User Scenario (REQ-06)
 Backing up the database every night is a background technical job for the server administrator. Users and instructors don't interact with backups, so it doesn't need a Gherkin test or a Use Case diagram. It is tested on the server side instead.