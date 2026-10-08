# AI Interaction Log (ai/prompts.md)

This log documents the iterative interaction with the AI assistant during the conceptual ER model design phase, fulfilling the transparency requirements for the assignment.

---

### Iteration 1: Domain Decomposition and Specification Criteria

* **Prompt:**
  > "Help define system boundaries, entities, and relationship rules for a dance studio CRM domain. I need to isolate a pure N:M relationship without junction tables, as well as an explicit associative entity holding its own attributes. Formulate strict Acceptance Criteria covering 3NF normalization and uniform key typing."

* **Result / Value:**
  * Identified core domain entities: `Student`, `Instructor`, `DanceStyle`, `Hall`, `Lesson`, and `Booking`.
  * Isolated the pure N:M relationship `Instructor }|--|{ DanceStyle` (requiring no intermediary table at the conceptual level).
  * Formalized `Booking` as an associative entity carrying business attributes (`booked_at`, `status`).
  * Established 3NF constraints: atomic attributes, surrogate `uuid` keys, and decoupled room metadata.

---

### Iteration 2: Syntax and Relationship Cardinalities (PlantUML)

* **Prompt:**
  > "What is the correct Crow's Foot arrow syntax in PlantUML for representing one-to-many relationships with optionality (0..N), one-to-one, and pure many-to-many? How should PK and FK attributes be annotated inside entity blocks?"

* **Result / Value:**
  * Adopted standard Crow's Foot notation: `||--o{` (1 to 0..N) and `}|--|{` (pure N:M).
  * Standardized field markers: `* id : uuid <<PK>>` and `* fk_id : uuid <<FK>>`.

---

### Iteration 3: Relationship Validation and Discrepancy Audit

* **Prompt:**
  > "Review these relationship mappings:
  > - Instructor }|--|{ DanceStyle
  > - Hall ||--o{ Lesson
  > - Instructor ||--o{ Lesson
  > - DanceStyle ||--|{ Lesson
  > - Student }o--o{ Booking
  > - Booking ||--o{ Lesson
  > Are there logical errors regarding booking mechanics, mandatory styles, or key typing? Help formulate the top-3 audit fixes for DEFENSE.md."

* **Result / Value:**
  * Detected an inverted direction in `Lesson ↔ Booking` (a single booking cannot reference multiple lessons).
  * Corrected invalid cardinality on `Student ↔ Booking` to a clean `Student ||--o{ Booking`.
  * Relaxed lower-bound multiplicity for `DanceStyle ↔ Lesson` from `1..N` to `0..N` (`DanceStyle ||--o{ Lesson`) to allow standalone catalog styles.
  * Generated the concise 3-point audit narrative for the "Verification and Audit" section.