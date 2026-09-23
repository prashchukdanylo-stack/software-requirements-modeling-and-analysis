
## 1.Why exactly this domen
I picked this domen, because it can easily demostrate common relations between enitites, popular around the world and has as pure relations, as associative ones.

## 2. Normalization Rationale (3NF)

The data model strictly adheres to the Third Normal Form (3NF) to guarantee data integrity:

1. **First Normal Form:**
   * All attributes contain strictly atomic values.
   * Full names are decoupled into `first_name` and `last_name` across `Student` and `Instructor` entities.
   * No repeating groups.

2. **Second Normal Form:**
   * The schema is in 1NF.
   * Every entity leverages a single primary key (`id : uuid`).
   * Because composite primary keys are avoided (including within the associative `Booking` entity), partial key functional dependencies are structurally impossible.


3. **Third Normal Form:**
   * The schema is in 2NF, with no transitive dependencies.
   * Hall capacity (`capacity`) is stored exclusively in `Hall` and is not copied into `Lesson` (where `max_capacity` represents only the administrative group limit, not the physical room size).
   * Contact details (`email`, `phone`) reside solely in `Student` and `Instructor`, avoiding duplication within schedule slots (`Lesson`) or records (`Booking`).

## 3. Verification and Model Audit (spec.md ↔ Diagram)

During the iterative audit of the specification against the declarative ER model, three key discrepancies were identified and resolved to ensure strict alignment with the acceptance criteria:

1. **Cardinals and Direction (Lesson ↔ Booking):**
   * Issue: The booking relation was initially modeled backwards, implying a single booking could hold multiple lessons.
   * Resolution: Explicitly defined a strict 1:N relationship (`Lesson 1 ||--o{ Booking`), ensuring each booking references exactly one lesson, while a lesson can aggregate zero or many bookings.

2. **Entity Optionality (DanceStyle ↔ Lesson):**
   * Issue: The relation originally required mandatory participation (`1..N`), which prevented registering newly introduced dance styles before scheduling actual classes.
   * Resolution: Relaxed the lower bound to `0..N` (`DanceStyle 1 ||--o{ Lesson`), allowing dance styles to exist independently in the system catalog.

3. **Key and Attribute Type Homogeneity:**
   * Issue: Acceptance criteria mandated uniform identifiers, but initial drafts risked inconsistent scalar types across foreign keys.
   * Resolution: Audited all primary (`PK`) and foreign (`FK`) keys across all 6 entities to strictly use `uuid`, ensuring zero type mismatch between referenced and referencing attributes.

   ## 4. AI Interaction
* **Role of AI:** AI was used as an advisory tool to draft `spec.md`, validate normalization criteria (3NF), and audit cardinality syntax in PlantUML.
* **Original Work:** The declarative diagram code (`diagram.puml`), entity structure, and final relationship decomposition were developed independently.
* **Prompt Logs:** Key AI prompts are stored in the repository at `ai/prompts.md`.