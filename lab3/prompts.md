# AI Interaction Log

This logs all ai prompts that I will write during lab3.
---

### Prompt 1: activate participants on diagram

* **Request:** "why i need to activate participants on diagram?"

* **Result:** "Activation bars (the vertical rectangular boxes on participant lifelines, created via activate/deactivate in PlantUML or activate/deactivate in Mermaid) serve specific functional and visual purposes in sequence diagrams.

Here is why they are essential for professional and rigorous system design:
1. Visualizing Execution Control (Active vs. Idle State)

A lifeline represents a participant's existence over time, but an activation bar shows when that participant is actively processing.

    When the bar is thin (the lifeline line), the component is idle, waiting for a message.

    When the bar turns into a thick rectangle, the component has received control, is executing logic, querying a database, or waiting for an external API.

2. Clarifying Synchronous Calls and Nesting

In complex architectures (like the dance studio system), components call each other sequentially. Activation bars make it immediately obvious who is waiting for whom:

    If Component A calls Component B, and B calls the Database, the stacked activation bars clearly show that Component A is blocked/waiting while B and the Database do the heavy lifting.

3. Preventing Ambiguity in Complex Logic (alt, opt, loop)

As specified in rigorous modeling standards, activation bars ensure that control flow doesn't break prematurely. If a participant performs work across multiple branches of an alt (alternative) or loop fragment, keeping the activation bar active across the entire block visually confirms that the component remains engaged throughout the entire conditional process until it officially returns a response.
4. Meeting Academic and Engineering Standards

In technical documentation, architecture reviews, and engineering assignments (like software design labs), activation bars are a core part of the UML specification. They demonstrate to reviewers that you understand the precise lifecycle of a request, transaction boundaries, and when resources (like database connections or worker threads) are tied up."

### Prompt 2: check for 1.1 diagram

* **Request:** "check my 1.1 diagram, overall is everything good?"

* **Response:** "Your Scenario 1.1 PlantUML diagram is exceptionally well-crafted, professional, and demonstrates a deep understanding of how real-world backend systems and databases operate under heavy concurrency!

Key Strengths of Your Diagram:

    Race Condition Prevention (FOR UPDATE): Using row-level locking when checking hall/lesson capacity is the gold standard for maintaining data consistency under peak load (NFR-02).

    Precise Capacity Management: Evaluating limits from both the lesson (max_capacity) and the hall (hall_capacity) using min(...) perfectly matches your EARS requirements.

    Accurate Intermediate States: Using 'pending_payment' instead of a generic 'pending' status ensures slots are accurately tracked from the moment checkout is initiated.

    Clean Sections: Breaking the sequence into clear blocks (== Booking creation ==, == Payment ==, == Webhook processing ==) makes it extremely easy to read and defend during presentations."
