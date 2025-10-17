here ya go — I cleaned and organized the transcript into lecture notes, keeping the speaker’s wording wherever it made sense, tightening filler, and grouping ideas. I also added short, clearly labeled pseudocode reconstructions of the examples the speaker described.

# Lecture Notes: Test Doubles, Mocks, Fakes, Spies, and Class Testing

## Housekeeping / Opening

- “The number of people here… class start time is getting lower and lower. Eventually, there’ll be just me.”
- Plan for today:
  - Cover **test doubles** (originally planned last time).
  - If time permits, move on to **class testing** (a step beyond unit testing, short of integration testing—“probably covered next time”).

------

## Unit Testing (quick recap)

- Unit tests “aim to test a method basically in isolation from the rest of the program.”
- The *method under test* is called by a test runner/driver from the unit testing framework.
- When that method would call other code, we **replace** those dependencies with **test doubles**.
- **Don’t replace standard libraries** (I/O, math, strings): “You’d basically have to re-implement the whole thing… not practical. Millions of people use those libraries; bugs are unlikely.”
- Goal: If a test fails, the bug should be in the unit under test, not “way over here somewhere else.”

------

## Why Test Doubles?

- **Speed**: External calls (network, DB, filesystem) slow tests; we want quick feedback on every commit.
- **Determinism**: External services change (weather API, stock prices), making tests flaky.
- **Avoid side effects**: Don’t send real emails, move real money, or pay for real API calls during unit tests.
- **Set up complexity**: Real dependencies may require special hardware/environments (e.g., “dashboard of a car”).
- **Control**: With doubles, we can simulate rare conditions (e.g., “filesystem is full”) reliably.

------

## Test Doubles: Terms & Types

> “People often say *mocks* to mean everything, but *mock* is only one kind of test double.”

**Test double = anything that substitutes during testing for the real dependency** to isolate the software under test.

### 1) Dummy

- **Purpose**: Fills a required parameter slot so the call compiles/runs.
- **Behavior**: Does nothing; never inspected or used by the test.
- **Note**: Not the same as a *null object*—you simply never *use* a dummy.
- **Example**: Pass a dummy `Logger` into a constructor when testing `ReplaceCommand` logic that doesn’t use logging.

### 2) Stub

- **Purpose**: Implements the interface and returns **fixed, hard-coded values**.
- **Use**: Force particular execution paths (e.g., an exception or timeout) to cover branches.
- **Example**: `CreditService` stub that always returns `true` (good credit) vs. another that always returns `false` to hit both branches.

### 3) Mock

- **Purpose**: Supports **behavior verification**.
- **Behavior**: Records how it was used—calls, parameters, times, **order**; may enforce allowed sequences (state-based constraints).
- **Example**: Mock `Window`/`Door` that records `open()`/`close()` calls and errors on invalid order (e.g., closing an already closed door).

### 4) Fake

- **Purpose**: A **working but simplified** implementation (often in-memory).
- **Examples**: In-memory DB, in-memory “network,” in-memory filesystem.
- **Use**: Enables **state verification** (assert internal state after operations).
- **Note**: More complex than stubs/mocks; fakes may require their **own tests** and are often reused/open source.

### 5) Spy

- **Purpose**: “A fake + a mock.”
- **Behavior**: Preserves real-ish behavior/state *and* records how it was called.
- **Use**: When you want both **state** and **interaction** verification.
- **Example**: A logging spy that both stores log entries (fake) and tracks calls/ordering (mock).

------

## Verifications: Output vs. Behavior vs. State

- **Output verification**: Assert returned value/output (`assertEquals(expected, result)`).
- **Behavior verification**: Assert that interactions happened (which calls, with what args, in what order).
- **State verification**: Assert that internal state of a fake changed as expected after operations.

------

## API Test Doubles (Clients & Servers)

- For client testing: create a **virtual service** that mimics the API (REST/gRPC) to avoid real network calls and flakiness.
- For server testing: test tools may need **client callback doubles** to simulate client behavior.
- Tools mentioned (as said in class): “wire mock” (**WireMock**), “Macoon” (**Mockoon**), “VCR” (HTTP recording), and “gRPC mock” tools.
   *(Names normalized in parentheses; the lecture’s spirit kept.)*

------

## Staging vs. Production

- **Staging**: “Real code with fake data” for end-to-end/system tests across services and nodes—no real money, customers, etc.
- **Production**: Don’t run tests; instead **monitor** for errors.

------

## Class Testing (a step beyond unit tests)

> “Class testing = test the modular unit (class/module/file) as a whole, isolated from the rest of the program.”

- **Assumptions**: Units in the class have passed unit tests.
- **Goal**: Find bugs in **interactions across methods** within the class.
- **What’s real, what’s double**:
  - Code **inside** the class: use the **real** methods (no doubles).
  - Code **outside** the class: continue to use **test doubles**.
- **Test class vs. class testing**:
  - *Test class* (JUnit term): a grouping artifact to run tests with `@BeforeAll/@AfterAll`, etc.
  - *Class testing*: the activity of testing a class as a module.
- **Approach**:
  1. **Rerun** unit tests for methods in the class, but replace internal doubles with **real internal calls**.
  2. Ensure **dependency coverage**: exercise all (transitive) interactions among methods within the class, not just statement/branch coverage.
  3. Use **white-box reasoning** to craft inputs that satisfy **path conditions** to reach specific internal calls (work backward from desired call sites).
- **Redundancy note**: Multiple unit tests may all trigger the same internal dependency; for dependency *coverage* they’re redundant, though not necessarily functionally redundant.

------

## Course / Project Logistics (closing remarks)

- Upcoming: finish class testing examples, then **integration testing** (across classes).
- **Iteration 1** guidance (services project):
  - Ideally deploy to **cloud** for Iteration 1 (not strictly required); **required** for Iteration 2.
  - **GCP credits** distributed by mentors: this year **$75** per student (was $50 last year). Don’t burn all credits in Iteration 1.
  - You should have: revised proposal (due yesterday), met with mentor, started developing the service.
  - Next time: details on what to deliver for Iteration 1 and demo expectations.

------

Here’s a cleaned, structured set of lecture notes based on your transcript. I’ve preserved original wording wherever possible, tightened filler, and grouped related ideas. I also added clearly labeled PostgreSQL snippets where the discussion implied concrete “search & replace” behavior.

# Lecture Notes — Testing: Test Inputs → Unit Testing (Search & Replace example)

## Quick check-in / housekeeping

- “Uh, can you hear me okay? … Yep, okay, good.”
- Picking up material not finished on Tuesday.
- Mini-project grading: Part 1 done; Part 2 almost done; Part 3 in progress (brief hiatus for team proposal reviews).
- Mentors: “If anyone has **not** heard from their mentors, put a note in chat.”
- Proposal deadline: extension planned. “Submit the **revised proposal** to the **separate** assignment; we’ll copy the new grade over the original so it doesn’t count twice.”
- Tools: small changes (e.g., Trello → GitHub Projects) are fine—update your README and reflect actual tools in Iteration 1 submission.

------

## Part A — Test inputs at the **functionality** level (Search & Replace)

> Scope: “We’re looking at functionality only—not UI. These are the test inputs you’d use for a user story. Acceptance criteria weren’t written, so we’re inferring them via tests.”

### 1) Basic behavior

- “**Simple match and replace**—that’s what it’s supposed to do.”
- Cases:
  - **Single occurrence** (the search term appears once).
  - **Multiple occurrences**.
  - **Whole word vs. partial word**:
    - If **whole-word only**, “dog” should not match “dog-along”.
    - If **partial** is allowed, “dog” should match “dog-along”.
    - Note: “I meant to include a partial-word example and forgot.”

### 2) Case sensitivity option

- Two modes: **case-sensitive** vs. **ignore case**.
  - Case-sensitive: only exact case matches are replaced.
  - Ignore case: “dog, Dog, DOG” all match. “We’d get all three.”

### 3) Variable-length replacements & not-found

- **Longer replacement** than search term: inserted inline; surrounding text remains intact.
- **Shorter replacement**: same—no loss of adjacent characters.
- **Empty replacement** (delete):
  - Leaves surrounding spaces untouched (may yield **double spaces**).
  - “We didn’t say we’d fix spacing; we just delete the token.”
- **Not found**: no change.

### 4) Position & overlap

- Boundary positions: beginning of document; end of document.
- Adjacent matches: e.g., “catcatcat” or runs like “AAA…B”.
- Ensure **options are respected**:
  - With **whole-word required**, “concatenate” must **not** match “cat”.
  - Without whole-word, both the word and inside-word matches are replaced.

### 5) Atypical but valid inputs

- **Accents / non-ASCII**: “Bistró → Cafe”; should also consider the reverse (ASCII → accented).
- **Punctuation**: handle “...”, “;”, quotes—replace targets that include punctuation.

> Takeaway: Even with “plain, boring, valid input,” options explode test cases. We haven’t even done invalid inputs yet.

------

## Part B — Unit testing (concepts, frameworks, patterns)

> “We treated `ReplaceCommand` and `SearchEngine` as **two classes**, so functionality spans methods. Now: unit testing the **smallest piece you can reasonably isolate**.”

### What counts as a **unit**?

- “Smallest piece of code you can reasonably isolate.”
- Often a **method/function**; sometimes a small, interconnected set.

### Choose inputs (unit scope)

- “Same baseline applies”: at least one **typical valid**, at least one **atypical valid**, at least one **invalid**.
- Better: **Equivalence partitions** & **boundary analysis**:
  - Boundaries: exactly at the limits, and one below/above (when representable).
  - Remember: `minint - 1` / `maxint + 1` may be **unrepresentable**—that’s an invalid class.

### Coverage guidance

- **Branch coverage**: every `if/else` true and false.
- **Loops**: 0 (if possible), 1, 2, and “many” iterations (and max if defined).
- **The missing else**: common pitfall when tests only hit the `if` path; assert behavior when the condition is **false** too.

### Running tests & independence

- Test runners execute tests in **some order** or **in parallel**; avoid inter-test dependencies **across groups**.
- A **test group** (often called a *test class*) may have intentional **sequential dependencies** *within* the group.
- Runners shouldn’t stop at first failure: you want the **whole failure set** (e.g., nightly CI).

### Assertions & AAA / GWT styles

- Why assertions? Console prints don’t automate pass/fail.
- **AAA** = Arrange (setup) / Act (execute) / Assert (verify).
- **GWT** = Given / When / Then (common in acceptance criteria and BDD; maps 1:1 to AAA).
- Tip: Test names should be short (e.g., `test_single_occurrence_replace`).

### Fixtures (setup/teardown)

- Goal: a **known fixed environment** so results are **predictable & repeatable**.
- Annotations (example terminology varies by framework):
  - `BeforeAll` / `AfterAll` (once per group): share a single resource across tests in the group.
  - `BeforeEach` / `AfterEach` (per test): give each test a **fresh** resource (no cross-test bleed).
  - `Timeout`: fail a test that hangs (useful for external resources or accidental infinite loops).
  - `Disabled`: skip a test temporarily.
- Language differences:
  - Java: nulling references lets **GC** reclaim memory.
  - C++/unmanaged: you must **deallocate** objects to avoid leaks.

### Structured inputs (containers)

- Many units accept containers (array/list/tree/graph). Consider:
  - **Present vs. absent** elements.
  - **Duplicates** allowed vs. not allowed.
  - Size extremes and ordering invariants.
- Fixtures are especially helpful for building **repeatable** container states.

### Inputs beyond parameters

- Functions may read:
  - **Global state**.
  - **API/system call results** (network, DB, filesystem).
  - **User input** (GUI/CLI).
- In unit tests, use **mocks** to control those inputs (moved to next class).

### Parallelism & race conditions

- If tests share mutable state and run in **parallel**, you can get race conditions.
- Strategy: isolate with `BeforeEach`, or serialize dependent tests within a group.

------

## Admin/Q&A highlights

- “Test runner vs. framework?” Runner is **part** of the framework; frameworks also provide assertions, fixtures, etc.
- Grouping: You can put one test per group, but the point of a group is to **intentionally order** dependent tests and share setup if desired.
- Internet hiccups (instructor’s connection) noted; Zoom usually reconnects.

------

# Lecture Notes — CRC Design, Testing Strategies, and Course Logistics

> Cleaned from the live transcript. Original phrasing is kept where it helps clarity; filler and repeats removed.

------

## Opening & Logistics

- “Can you hear me?” — ✔️ class audio confirmed.
- “It’s officially time to start class… 17 people are here.”
- **Mentors**
  - “You should have heard from your team mentor by now. They’ve been assigned as of yesterday.”
  - If you haven’t heard: check the posted link with mentors; *“the ones who have multiple teams are the paid IAs; the leaders each have one.”*
  - Email addresses match those on the CourseWorks front page.
  - **Action:** “Arrange to meet your mentor as soon as possible.”
- **GCP credits**
  - “You’ll get your Google Cloud Platform code… $75 of credits for everybody.”
  - Use for hosting & testing the team project service.
  - “Don’t just leave it running all the time—wastes your credits. We can’t get more.”
- **Proposal revisions**
  - If your mentor graded the original proposal and it wasn’t full/near-full credit, *“you probably should revise.”*
  - The “Optional Revised Project Proposal” assignment is **identical** to the original but **replaces** the original grade (no double counting).
  - “Talk to your team’s mentor before submitting a new proposal.”

------

## From User Stories/Use Cases → CRC Design

### Example: “Search and Replace” user story

- Reference CRC (4 cards):
  - **Document** — owns/updates text (may log for undo, though undo isn’t in this story).
  - **SearchEngine** — “searches” and returns positions/occurrences.
  - **ReplaceCommand** — “applies replacements to the Document” (design uses a *replace-all* flow).
  - **UserCommand** — captures user’s intent/inputs; interfaces with the UI; tells others what to do.

#### Walkthrough (TEH → THE)

1. `UserCommand` captures target/replacement.
2. Creates `ReplaceCommand`.
3. Asks `SearchEngine` to find occurrences in `Document`.
4. `ReplaceCommand` applies updates to `Document`.

### “Find the nouns” — but only those with **responsibilities**

- Bad candidates:
  - **“Wrong word”** — just data (string), not a class with behavior.
  - **“Word processor”** — the whole system (“God object”); avoid.
  - **“User”** — external actor; represented via `UserCommand`, not as a domain class.
- Good candidate:
  - **Document** — core entity.

### Responsibilities (verbs) & options

- Keep only those in scope for the story:
  - **Search** → `SearchEngine`.
  - **Replace** (in this design) → `ReplaceCommand` applying to `Document`.
  - Out of scope examples:
    - **Load** (Document I/O), **Spell-check “red squiggle”** — different features.
- Communication actions (“tells”) are modeled explicitly (e.g., `UserCommand` tells `ReplaceCommand`).

### Options/attributes (live **on the back of cards**)

- Ignore case
- Whole-word matching
- Replace-all vs. step-by-step confirm

------

## From CRC Cards → Code (process)

1. Create class skeletons (one per card).
2. Add instance variables:
   - Data (target/replacement strings)
   - Options (ignoreCase, wholeWord, replaceAll)
3. Stub methods matching responsibilities.
4. Add collaborator references/imports.
5. Fill method bodies; call collaborators.
6. **Unit test** each class (use test doubles/mocks).
7. Integrate & refactor (remove duplication, add missing pieces, update cards as you learn).

------

## Testing Strategy: Test-to-Pass & Test-to-Fail

- **Test-to-Pass (smoke test)**: “Does it basically run without obviously crashing?”
  - Example: Step tracker with a typical age (25) → expect a plausible steps/day (≈10k–12k).
  - “Smoke test” = turn it on; see if smoke comes out.
- **Test-to-Fail (main effort)**: “Try to cause failure safely.”
  - Cover **edge/corner cases**; confirm error handling matches docs/expectations.
  - Examples (step tracker age field):
    - **Atypical valid**: 116 (3 digits), 2 (very young but possible).
    - **Invalid**: floats (25.5), non-numeric strings (“twenty”), empty, too-long strings, control chars.

------

## Equivalence Partitioning & Boundary Analysis

- Goal: represent the full input space with a **small** set of cases that are most likely to reveal bugs.
- **Equivalence partitions** come from:
  - **Domain knowledge** (what values make sense),
  - **Software knowledge** (where bugs usually hide).
- Simple numeric range example:
  - Valid: `1..1,000,000`
  - Invalid: `<1`, `>1,000,000`
  - **Boundary values**: 1, 1,000,000, plus 0 and 1,000,001; also min/max ints where relevant.

### Domain-shaped partitions (ages)

- Assume **integer ages** and realistic constraints:

  - Negative → invalid

  - 0–3 → likely invalid for this app

  - **4–17** (minors), **18–59**, **60–150** (adults/older adults; may drive different prompts)

  - > 150 → invalid

- Test boundaries across these partitions (3,4,17,18,59,60,150,151).

------

## Calculator GUI Example (black-box testing)

- **Plus operation** takes **two numbers** (domain math), but software ≠ pure math:
  - Mixed types may fail (integers only in v1; “dot key” exists but disabled).
  - UI digit limits (e.g., 8 digits) imply **UI-level min/max** independent of language limits.
- **Equivalence classes over pairs** (order matters!):
  - (+,+), (+,0), (0,0), (–,–), (+,–), (–,+)
  - Boundaries: maxIntUI/minIntUI, overflow by one digit, zero handling.
- **Interpreting outputs**:
  - “Flashing zeros” may *be* the error indicator (overflow, trial expired, unsupported types).

------

## Optional vs Required Inputs

- Optional fields (e.g., middle name) must be tested:
  - Provided vs omitted; length limits; character constraints.
- Real-world edge: *one-name* users (no true first/last split) — many forms break here.

------

## Dependencies & Consistency Between Inputs

- **Dependencies**: output depends on a **relationship** between inputs (e.g., lab ranges spanning 2D space).
- **Consistency**: inputs must agree (DOB vs reported age; checksums vs payload).
  - Mismatch should trigger clear error feedback.

------

## System-to-System Inputs (preview)

- When services exchange data: consider **files/JSON/tuples/streams**.
- Partitions to consider: file existence, permissions, size ranges, content format/codec, and protocol-level constraints.

------

## Course Framing & Next Topics

- “This class is largely about testing and finding bugs.”
- Coming up:
  - **Unit testing** (deeper dive),
  - **Test doubles / mocks**,
  - Security-adjacent topics (e.g., integer overflow, invalid-input exploits),
  - Static analysis.
- Reminder:
  - “Check who your mentor is and talk to them to get GCP credits and proposal guidance.”

------

## Action Items for Students

1. **Confirm mentor contact**; schedule a meeting.
2. **Budget GCP credits** (~$75): turn services off when idle.
3. **Decide on proposal revision**; submit to the **revised** assignment (replaces original grade).
4. Start translating your **current user stories → CRC**; then stub classes and unit tests.
5. Build test suites using **equivalence partitions** and **boundary analysis**; include test-to-fail cases.

------

# Lecture Notes (cleaned & organized; original wording preserved wherever possible)

## Opening / Logistics

- “People are arriving slowly.”
- Instructor riffed on an **attendance quiz / extra credit for showing up early**. Zoom makes it easy to check **join times** via reports, but “it’s not practical on Zoom” to run a quiz that only early arrivals can pass.
- “I’m not actually taking official attendance this semester… but I can still give extra credit for people who are here.”
- Sound check: “Can you hear me okay?”
- Plan: maybe credit for those “here in the first 5 minutes,” since Zoom records join times.

> **Note on “PostgreSQL”**: The discussion refers repeatedly to **postconditions**, not PostgreSQL. There was **no database/SQL content** to reconstruct. See the sections on *preconditions* and *postconditions* below.

------

## Agenda

- Wrap-up from “Use cases to user stories”
- Today’s main topic: **CRC Design** (and “maybe a little UML design”)
- Flow: **User stories → Use cases → CRC design → Code**

------

## From User Stories to Iteration Planning

- Gather your **set of user stories** for the product/project.
- If any are **really big**, split them into **subparts**.
  - Very large user stories are called **epics** (“a really, really long story”).
  - Smaller ones are just **user stories**.
- **Prioritize** which stories go into the upcoming **iteration**.
  - In class you’ll have only **two iterations**; in industry, iterations are often 2–4 weeks on a rolling basis.

### Timeboxing vs. Scopeboxing

- **Timebox**: fix the time (e.g., two weeks), then ask “How much can we get done in that period?”
- If you can’t finish everything, **push scope** to a future iteration.
- If you **add scope**, it **takes longer** (assuming fixed team/velocity).
- You can’t get “more scope in less time” without changing resources/velocity.

------

## Example: Splitting an Epic

**Epic**: “As a registered user, I want to manage my account and personal profile.”

Split by workflow into independent, releasable stories, e.g.:

- “As a registered user, I want to **update my email address**.”
- “As a registered user, I want to **change my password**.”
- “As a registered user, I want to **update my personal information** (name, phone, etc.).”
- “As a registered user, I want to **upload/update my profile picture**.”
- Potentially many more (nickname, physical address, etc.).

### Acceptance Criteria (a.k.a. Conditions of Satisfaction)

- Use **Given/When/Then** (example for updating email):
  - **Given** I am on the Edit Profile page,
  - **When** I enter a new **valid** email address and submit the form,
  - **Then** I should see a success message, receive a confirmation email at the **new** address, and see the change reflected in my profile.
  - Instructor note: “I’d also want another AND: send a confirmation to the **old** address,” to detect account compromise.

------

## Expanding a Story into a Use Case

For the “update email address” story:

- **Identifier/Name**: e.g., `UC-Profile-ChangeEmail`
- **Actor**: Registered user (note: “actor” can be a person or an external system)
- **Goal**: Successfully change the email associated with the account
- **Preconditions**: User is **logged in**; navigated to **Profile/Settings**
- **Postconditions**:
  - **Success**: Email updated to new value; **confirmation email(s)** sent; profile reflects change
  - **Failure**: Email remains unchanged

### Flows

- **Basic (“happy path”) flow**: numbered steps showing UI/system interactions in order.
- **Alternative flows** (reasonable branches, not necessarily errors):
  - User **cancels** midway; system discards changes and returns to profile page.
  - **Invalid email format** detected immediately; show error.
- **Exception flows** (surprising/unexpected cases):
  - **Email already in use** by another account (often it’s your own forgotten account).
  - **Verification link expired**; provide a way to **resend** and resume.

> Q: “Is the flow optional?”
>  A: **No**. You need at least the **basic flow**; alternatives/exceptions may be omitted initially but must be handled before implementation is complete.

### Process Note

- Earlier days: waterfall handoffs (requirements → design → coding → testing) by separate teams.
- Today: agile teams still benefit from **working out details early**, especially for less experienced programmers.

------

## CRC Design (Class/Component–Responsibilities–Collaborators)

- Also seen as **Component** Responsibilities Collaborators (language-agnostic).
- A **class/component** represents an **entity** (not necessarily OOP-only).
  - Has **data** (attributes/properties) and **behavior** (functionality).
- **Responsibility**: what the entity **does** (methods/functions), possibly including checking **pre/postconditions**.
- **Collaborator**: another class/component this one **knows/interacts with** to fulfill a responsibility.
- **Avoid the “God class”** (“the system does X”). Assign each system action to a **specific class**.

### Finding Candidate Classes (from Use Cases/User Stories)

- Look for **nouns/noun phrases** (but exclude the actor unless you model its in-system representation).
- Watch outs:
  - Adjectives that **change meaning** (e.g., **toy** car ≠ car).
  - **Passive voice** hides agents (“is activated” → *who* activates?).
  - Distinguish **attributes** vs. **entities** (e.g., *radius* is usually an attribute of *circle*).

------

## In-Class Exercise: **Search & Replace** user story

**Story**: “As a user, I want to **search and replace** text in a document (including capitalization fixes).”

### Candidate Classes (nouns)

- **Wrong word** (the text to replace)
- **Replacement word/text**
- **Document**
- **User** (optional: owner/initiator)
- **Highlighter** (red squiggle under misspellings)
- **Input prompter** (UI for prompts)
- (Potential) **Word** entity; **Search/Replace placeholder**

### Responsibilities (verbs)

- **Search**
- **Replace**
- **Load** the document
- **Highlight** matches (e.g., red squiggle)
- **Prompt** the user for replacement text; **accept input**
- Options: **ignore case**, **whole-word match**, **replace all** vs **step-through**

### Collaborations

- `WrongWord` ↔ `ReplacementText` ↔ `Document`
- `Highlighter` ↔ `Document` (to render highlights)
- `InputPrompter` ↔ `User` (prompt/collect text)
- `Word` (as a structure) may be used by `Document`, `Highlighter`, and search logic

------

## Using CRC Cards

- Create a **card for each class** with:
  - **Class name**
  - **Responsibilities** (what it does)
  - **Collaborators** (who it needs)
- **Animate/walk through** the **use case** step-by-step using the cards:
  - Ensure **every behavior** maps to **some responsibility** on **some card**.
  - Add missing cards/responsibilities as discovered.
  - Make sure **preconditions are checked** and **postconditions achieved** by specific classes.

------

## From CRC to Code

- Map **classes/components → files/modules/classes** in your language.
- Map **responsibilities → methods/functions**.
- Add **fields/state** to hold attributes and **references to collaborators**.
- Update/extend the **test suite** to cover acceptance criteria and flow branches.

### When Legacy Code Exists

- If something **almost** fits, use an **Adapter**:
  - New **interface** = responsibilities on your card
  - Adapter wraps old code to present the required interface
- You can also **change interaction style** (e.g., wrap internal code with an **API endpoint**).
- **Technical debt** trade-offs: quick adapter vs. rewriting properly (you “pay interest” later).

------

got you — here’s a cleaned, structured set of lecture notes with original wording preserved where it helps clarity. I’ve also added a short “PostgreSQL (reconstructed)” section to model the artifacts that were implied (teams, proposals, mentors; task boards; user stories & acceptance criteria; use cases).

# Admin / course flow

- “It looks like every team submitted a proposal? Great.”
- Prospective mentors are reviewing proposals now; exact mentor ↔ team assignment will be worked out “in the next 2 or 3 days.”
   Goal: “by the end of this week, we’ll have everybody set.”
- Please don’t keep asking when Part 3 (etc.) will be graded. Mentors need time to go through the proposals first. Mini-project grading will resume after mentors are assigned.
- After you get a mentor, they’ll contact you. Meet with them, and decide whether to submit a revised proposal (optional but “probably wise” if the mentor suggests it).

# Task boards (Kanban) — why & how

- Use a task board for your **team project** this semester.
  - Simplest: **GitHub Projects** (+ GitHub Issues for bugs).
  - Also fine (free tiers): **Jira**, **Trello**. “Pick some free thing, don’t pay money.”
  - Add the board link to your repo **README** and grant mentor access.
- What a task board is: maps workflow, often per iteration (not the whole project). Can be online or a physical board.
- Typical columns: **To-Do → In-Progress → Done**. For software, In-Progress is often split (Design, Coding, Integration, Review, Testing).
- Why useful:
  - Visible ownership: “who is doing what?” helps coordinate dependencies.
  - Limit **WIP** (e.g., 5 engineers ⇒ ≤5 items in progress). Finish work before starting new work; handoff to Review/Test columns.
  - Reveals **bottlenecks** (e.g., “Ready for Review” piling up).
- Pitfalls:
  - If you don’t keep it **up to date daily**, people will ignore it.
  - Can get cluttered on very large efforts.
  - Focuses on **tasks over goals**; use other artifacts for big-picture timeline.
  - Risk of perceived micromanagement if a card lingers in In-Progress.
- For class structure:
  - Prefer **one board per iteration** (you may pre-create the next one).
  - You *can* go finer-grained (weekly), but that’s more than expected.

# Upstream vs. downstream engineering; Waterfall vs. Agile

- Waterfall “upstream”: **Requirements, Design**. “Downstream”: **Implementation, Verification**, then **Maintenance**.
- Waterfall in practice tends to be **iterative** due to backtracking when defects trace to earlier stages.
- Royce (1970) essentially argued *against* strict waterfall; industry today is mostly **Agile** (small iterations, MVP first, frequent releases).
- In these notes, “iteration” and “sprint” used interchangeably.
- Some requirements **must** be defined **up front** (e.g., internet protocols like HTTP/URL/TCP/IP—global interoperability).
- Course emphasis: downstream (clean code, APIs, testing/bug-finding). A little upstream: **user stories, use cases**, some **design** (CRC, UML class diagrams).

# User stories

- Format (3×5 card spirit):
   **As a** *type of user/role*, **I want** *goal*, **so that** *reason/value*.
  - **Acceptance criteria** (“conditions of satisfaction”), often in **Given/When/Then** (Gherkin) form.
- **Role** ≠ separate client app: same client can have many roles (e.g., CourseWorks roles).
- Examples (paraphrased from talk):
  - Login success & failure scenarios with clear messages.
  - Search scenarios: exact match, partial, no results.
- **INVEST** qualities (implied):
  - Independent (as much as feasible), Negotiable, Valuable, Estimable, Small (completable within an iteration), Testable.
- For **APIs/services** (your project):
  - Users may be **consuming entities** (other services, partner apps, front-ends).
  - Example story: “As a mobile checkout service, I want an endpoint to reserve an item and place an inventory hold so the user has a guaranteed item before payment.”
  - Service contract concerns (industry-grade; know about them even if not required here): **idempotency**, **atomicity**, **latency/SLOs**, **error handling** (structured JSON errors & specific status codes), **retry semantics**.

# Use cases

- Purpose: turn needs into **step-by-step behavior**, with structure for **flows** and **what can go wrong**.
- Components:
  - **Actor(s)**; optional **stakeholders**; **preconditions/postconditions**; optional **triggers**.
  - **Success (basic) flow**, **alternative flows**, **exception flows**.
- UML “use case diagrams” (ovals) show participation but **not** the behavioral steps; use **textual specs** for actual detail.
- **Laundry** example (canonical teaching case):
  - Preconditions: “It’s Wednesday”; dirty laundry already in laundry room.
  - Steps: report to laundry room → sort → wash loads → dry → post-processing flows:
    - **Basic**: fold items that should be folded.
    - **Alternative**: iron/hang wrinkled items; discard ruined items.
  - **Exception** ideas: housekeeper/robot doesn’t arrive; laundry room flooded; machine broken; load too big; machine too slow and worker leaves; laundry missing, etc.

# What’s next

- Next class: from **user stories → use cases → CRC design** to identify classes, methods, and relationships. Also **UML class diagrams**.
- Iteration timing: ~3+ weeks per iteration; second iteration mostly completes what began in the first; expect one “real” release at the **end of iteration 2**.

------

# Lecture Notes (cleaned & organized)

> Original wording preserved wherever practical; filler and repeated prompts condensed. I’ve grouped the material into logical sections with brief headers.

------

## Admin & Housekeeping

- “Can you hear me okay? … Good, thank you.”
- Several students **did not submit team formation**—not even a “couldn’t find a team” note. If that’s you, **fix it now**.
- **You cannot continue the course** without a team and an active team project by **next week**.
- Team formation submission has been **reopened** in CourseWorks (will show as overdue but you can submit). Include:
  - **Team members**
  - **Team name (alphanumeric)** — needed to create CourseWorks group and Discord channels.
- Teams of **4 are preferred**. One team may run with **3** (considered in grading). Submitting solo = unrealistic (unless doing “four students’ worth of work”).
- **Proposals due Monday 11:59 pm.** Late proposals likely **won’t get a mentor initially**, which hinders progress.

------

## Mentors & GCP Credits

- After proposals are read, teams are **assigned a mentor**.
- **Meet with your mentor ASAP** after assignment.
- At that meeting, mentors hand out **“magic codes” → GCP credits** to deploy/test your service.
- Credits **only** for team members who **meet** with the mentor (Discord/virtual is fine).

------

## Proposal Guidance

- “Make a big point” to propose **something you really want to do**—you’ll do it **all semester**.
- Length: **~1 page (maybe 2)**. Think of it as a **promise of conversation** with your mentor.
- Include what your **service** does, **types of clients**, scope.
- Examples were shown in Tuesday’s class; review those slides/recording.

### Revised Proposals

- If mentor discussion triggers major changes, use the **Optional Revised Project Proposal** assignment to **replace** your original.

------

## Persistent Data / CRUD

- A pipeline that **only feeds data** is not enough; you should have **CRUD** in your persistent store.
- An **immutable** baked-in dataset isn’t a substitute for a proper persistent store.

------

## Code Review — Why & How

### Why human (or AI-assisted) review?

- Automated tools (style checkers, static analyzers, tests) **can’t find missing code** (absent functionality + absent tests can still yield “100% pass/coverage”).
- Reviewers, using **requirements/design**, can spot **missing features**, **unreadable code**, and potential **vulnerabilities**.
- Reading code enables reasoning about **all possible inputs** in ways tests can miss.

### Unreadable Code Example

- Reference to an **obfuscated C** tic-tac-toe (obsfuscated C contest): point is **maintainability**—no one else can maintain such code.

### Lightweight vs. Formal Reviews

- **Lightweight (common):**
  - PR-based, pre-merge gating.
  - **Asynchronous** preferred (less context-switch for reviewer; more for author).
  - Useful for onboarding; reviewer explains process (commit locally → PR → review comments).
- **Formal (Fagan inspection):**
  - Used in **safety-critical** or regulated environments (NASA/DoD).
  - Roles: **Author(s)**, **Reader** (reads code aloud), **Moderator**, **Inspectors/Note-taker**.
  - Process: **Plan → Prep (pre-read) → 2-hour inspection meeting → Rework → Follow-up**.
  - Often with **checklists** (code + docs).

### Checklists Mentioned

- **JPL “Power of Ten”** (C/C++ rules; e.g., avoid `goto`, function length limits, disciplined returns, etc.).
- **Google’s general review guidance** (language-agnostic: design, readability/simplicity, naming, tests, feedback quality).

------

## Example: Review Notes on a Factorial (Java)

Typical reviewer comments:

- **Missing Javadoc** for class/method.
- **Invalid input**: negative `n` → infinite loop; add a check (throw exception).
- **Overflow risk**: prefer `long` (or `BigInteger`) if larger inputs allowed.
- **Naming**: prefer `number` over single-letter `n`; `i` is acceptable as loop iterator.
- If using recursion, consider **stack overflow**; ensure **exceptions** are well-handled.

------

## AI in Code Review

### Advantages

- **Fast, scalable**: can sweep large codebases repeatedly.
- **Consistent** enforcement of standards.
- Can surface **security patterns** seen across vast training data.
- **Doesn’t get bored**.

### Disadvantages

- **Context blindness**: flags acceptable trade-offs; misses organization goals/constraints.
- **False positives/negatives** fatigue humans (warnings get ignored).
- **Over-reliance** harms skill development—especially for juniors/students.
- **Security/IP** risk with third-party tools; training/retention of your code; vendor trust.
  - **On-prem** deployments mitigate data exfiltration but require careful network isolation.
  - “We don’t train on your data” claims exist; trust and contracts vary; big companies can negotiate strict terms.

------

## Pair Programming

- Styles: **Driver/Navigator**, **Ping-pong**, informal over-the-shoulder.
- Benefits:
  - **Collective code ownership**, improves **truck/bus factor**.
  - **Focus** (fewer interruptions, less context drift).
  - **Quality** (catch typos, logic/design issues early).
  - **Knowledge transfer**, onboarding.
- Costs/when to use:
  - Two people ≠ always efficient; best for **hard/high-risk** tasks.
  - Remote pairing **works** (screen share) but can be clunkier.
  - Personality/fit matters.
  - Use **Pomodoro** style breaks (25/5) to manage fatigue.

### “Pairing with AI”

- Useful for **boilerplate**, tests, completions; **24/7** availability.
- Caveats: **context gaps**, **subtle bugs**, not a **mentor**, **security/IP** concerns.

------

## Task Boards

- You’re expected to use a **task board** for the team project and **link it in the README** (details in next class).

------

## Coverage & Grading Clarifications

- **Mini-project** (I3): aim is **increase** coverage across parts; adding code without adding tests will **lower %**.
- **Team project:**
  - **Iteration 2** target: **≥ 80% coverage**.
  - Higher (e.g., **90%**) is “great.”

------

## Communication & Logistics

- Use **ED** for questions of general interest.
- For grader-specific items: **email** or your grader’s **Discord** channel.
- Each team has Discord **text + voice** channels (created once we know the **alphanumeric team name**).
- **Late penalties** in CourseWorks can be overridden—**coordinate with your grader**.
- Case sensitivity issues and IDE differences: graders follow a **documented Maven** sequence; talk to your grader if discrepancies arise.

------

## Q&A Highlights (condensed)

- If you formed a team **after** your initial “no team” submission, **resubmit** the team formation now with full roster + name.
- **Mentor assignment** likely won’t be finished by **Tuesday** right after Monday’s deadline; **target is Thursday**.
- **Regrade/late issues** (e.g., homepage points): speak to your grader; instructor acknowledged specific cases for **points back**.
- **Resubmissions** for some assignments permitted due to delayed feedback—**ask your grader** to open the window.

------

# Lecture Notes (cleaned & organized)

## Quick announcements

- Several students’ HW2 submissions **did not compile** under the graders’ environment.
  - You may **resubmit HW2** to fix compilation issues (watch ED Discussion for the exact instructions/notes).
  - Compare what compiles **locally** vs. what you actually pushed to **GitHub**.
  - Common pitfall: **case-sensitive paths / filenames** differ between your machine and the graders’.
- **Team formation**
  - Goal: teams of **4 (preferred)** or **5**; 3 only if absolutely necessary (scope would be adjusted).
  - If you still need a team, coordinate via provided breakout rooms (Java → Room 1, C++ → Room 2).
  - CourseWorks project groups are being created; once your group exists, **one member can submit for the team** going forward.

## What’s next this semester

- Finish your **individual mini-project** by **adding CI**, then you’re done with it.
- The rest of the semester focuses on the **team project**:
  - **Project proposal** due **Monday** (the coming Monday).
  - After proposals, mentors will review, rank preferences, and each team will be assigned a mentor.
  - Meet your mentor **ASAP** (ideally in the first week after assignment).
    - There’s an **optional revised proposal** if you make major changes after mentor feedback.
  - **Iteration 1** due ~**Oct 23** (API docs required then).
  - **Demo with mentor** during the week between Iteration 1 and its demo deadline (schedule early).
  - **Iteration 2** later; **Demo Day** at semester end (10-minute slots).

------

## Core requirements for your service

Your team decides the service, but it must satisfy all of the following:

1. **It’s a service (no direct human UI).**
   - No GUI or CLI for end-users.
   - Human interaction only for **administration** (e.g., pre-populating data for demos).
2. **Multiple, genuinely different clients** can use it.
   - Not “web vs. mobile” of the same app; not merely different **roles** (buyer/seller).
   - Think: many unrelated programs/companies could use the same API for different products.
3. **Stateful with persistence.**
   - Maintain a persistent datastore (SQL/NoSQL/files OK).
   - If the service is stopped and restarted, the state **remains**.
4. **Clear API surface.**
   - Well-defined endpoints, request/response schemas, status codes, and side-effects.
5. **Comprehensive logging.**
   - Log every API entry call, inputs, caller identity (token/client ID), response, and errors.
   - Logs are **persistent** and ideally **append-only**.

> You will **build one client** for Iteration 2, but in the **proposal** you should articulate at least **two distinct** potential clients.

------

## “Varied clients” — what it really means

- Good examples:
  - **reCAPTCHA** used across Facebook, NYT, Ticketmaster, etc.
  - **PayPal** invoked by eBay, Walmart, airlines, streaming, gaming, etc.
     These clients are **unrelated programs** sharing the same service.
- Not sufficient:
  - Same app on web vs. mobile vs. desktop (variants of one client).
  - Same backend with “admin” vs. “user” roles.

------

## Example service ideas (from class)

1. **Scientific Unit Conversion Service** (simplest)
   - Converts across many scientific domains (beyond Celsius↔Fahrenheit).
   - Persistent cache/history; conversion factors as data; error on non-convertible pairs.
   - Clients: educational sites, engineering tools, calculators, lab software.
2. **Fictional World Generator Service** (richer)
   - Generates maps, NPCs, lore, quests; stores worlds for retrieval.
   - Endpoints like `generateCharacter`, `generateCity`, `getWorld`.
   - Clients: RPG games, writer tools, map-making utilities.
3. **Personal Product Review Service**
   - Your **private** cross-store memory of products you liked/hated.
   - Clients:
     - Browser extension (shopping websites).
     - Mobile app (in-store). *(These two are functionally similar.)*
     - **Different client**: vendor analytics tool (aggregated, anonymized insights).
4. **Context-aware Navigation Overlay**
   - Crowdsources **hyper-local** recurring constraints (e.g., school drop-off blockades, long-term closures).
   - Clients:
     - Planning browser extension (trip prep) and mobile driving app *(similar)*.
     - **Different client**: municipal planning dashboard (to mitigate recurring choke points).

------

## Proposal vs. later API docs

- **In the proposal (now):**
  - What your API *does* and **why it’s useful**; target users (developers); concrete use-cases.
  - Which **distinct** clients would integrate it and how.
  - Data persisted; high-level endpoints; trust/auth model at a high level.
- **In Iteration 1 docs (later):**
  - Precise endpoint specs, request/response schemas, status codes, auth/roles, side-effects.
  - Examples/snippets; error taxonomy.

------

## Logging: why and how

**Purposes**

- **Audit trail** (esp. for money/security-sensitive actions).
- **Debugging** (reconstruct state, inputs, execution path when things go wrong).
- **Analytics & optimization** (which endpoints matter).
- **Anomaly detection** (spot unusual patterns).

**Practices**

- Use a real logging library with **levels** (DEBUG/INFO/WARN/ERROR/CRITICAL).
- Record: timestamps, request ID, client ID/token, endpoint, params (PII-safe), outcome, latency, resource usage if relevant.
- Treat logs as **append-only** and **persistent**; rotate/retention for size.
- **Do not** replace logging with `print`—it can change program behavior and isn’t level-controlled.
- Protect sensitive data: avoid secrets/PII in logs; if necessary, **hash/redact**.

**Debugging mindset**

- Make implicit **beliefs** explicit via logs (e.g., “C must be non-zero here”).
- Emit path markers to recover execution flow without stepping through a debugger.

------

## Demo without a GUI

- Use an **API testing tool** (e.g., curl, Postman) to show requests/responses.
- Show **persistent state changes** by querying your datastore (e.g., psql shell).
- No bespoke end-user interface for the service.

------

Here’s a cleaned, structured set of lecture notes—with original wording preserved wherever possible—and a short set of **reconstructed PostgreSQL snippets (clearly implied)**, labeled as such.

# Tech check / opening

- “Uh, can you hear me? … Anyone there? … thank you. Alright.”
- Plan: consider opening breakout rooms for team formation; may “wait and see if more people show up.”

# Plan for today

- Start with team formation (breakout rooms) so people can “find teams if they don’t already.”
- Then continue with lecture: **Continuous Integration** → **Distributed computing & RPC (gRPC)** → logistics.

------

# Continuous Integration (CI)

## Nightly build vs CI

- Clarification from Tuesday: “I had this big thing about nightly build… **No**—don’t set up a nightly build for team projects.”
- Nightly builds are for “Google… billions of lines of code.”
- For class projects: **set up continuous integration**:
  - “Every time you have a pull… a pull request is accepted… on every commit to the trunk.”

## Point of CI

- “Encourage **small, frequent changes**.”
- “Developers merge their code fairly frequently, **often multiple times per day**, not once per day for a nightly build.”
- “Each merge is verified by the **build and test** process.”

## Code review

- Typical industry flow: PR triggers **peer code review** (team lead or assigned reviewer).
- Often review happens **before** build & test; “we’ll talk about code review later.”

## Why continuous *integration*?

- Avoid “**integration hell**” from long-lived branches:
  - Refactor vs new feature “pretty much guaranteed to conflict.”
  - “New features break old ones; software may not compile.”
- Merge **as you go**, especially with “3–4 week” iterations—no time for late integration.

## Overwrites & merge conflicts

- Proper conflict handling prevents overwrites, **but** manual merges can still “pick mine” and ignore others. “Don’t do that.”

## Typical workflow

1. Finish “some **small piece** of a feature” (not the whole feature).
2. Commit via **pull request**.
3. **CI server** (use **GitHub Actions** in this course) runs:
   - Build (“integrates” = compiles; could be multiple components).
   - Optional deploy to staging (not focus of class now).
   - Automated tests: unit, integration, API tests (may use different tools).
4. **Feedback loop**: if it breaks, the last committer fixes it “when it’s fresh.”

## Benefits

- Reduces risk; keeps codebase near “deployable” state.
- Fast feedback.
- Promotes **shared ownership**: anyone may fix/refactor each other’s code (org-dependent in practice).

## CD vs CD (delivery vs deployment)

- **Continuous delivery**: ship a new release users actively update (“Zoom client wants to be updated, like, all the time”).
- **Continuous deployment**: new version goes live “and they don’t necessarily even notice.”

## Resources

- Instructor “curated a couple good videos” and texts on GitHub Actions for CI (and CD).

------

# Team formation & class logistics

## Breakout rooms (team finding)

- 15 rooms opened; 5 designated:
  - Room 1: C++
  - Room 5: TypeScript
  - Rooms 2–4: Java split by platform (e.g., OS variants).
- Use extra rooms or Discord team rooms. About **20 minutes**; “please don’t leave—more material afterward.”

## Deadlines & sizes

- **Team formation assignment due Monday**; “everybody on the team has to submit the same thing.”
- Preferred team size **4**; **5 is okay**; others **not okay**; **no solo teams**.
- After teams form: **project proposal due one week later**.
- Use time to start planning proposals.

## Practical advice

- **Check platform/OS compatibility** early (Mac/Windows mismatch can bite at build time).
- **Schedule compatibility**: CVN students often work weekends; on-campus often do other things on weekends—align expectations.

## Mini-project reminders

- Part 2 due **tomorrow**; late-joiners had Part 1 + Part 2 due together.
- Part 3 “opened up overnight.”

## Using external APIs in projects

- “**Yes, absolutely**—strongly encouraged.”
- **But** don’t build a thin wrapper that “just passes through” to another service.

## Grading timeline (informal)

- Part 1 grading “pretty far along,” but must wait until all extensions submit “after the deadline for everybody.”
- Part 2 “about a week.”
- You can ping **your grader** (same as Part 1) for a quick sanity check before Part 3; CourseWorks grade release is “all or nothing.”

------

# Distributed computing & RPC (including REST vs RPC)

## Local vs distributed calls

- Local: caller & callee in same process; pass params; return result.
- Distributed ideal: looks the same to app code, but caller & callee on **different machines**.

## Stubs & serialization

- Client code compiles against a **client stub** (same signature).
- Server side has a complementing stub.
- Params/returns **serialized** (no pointers) into network packets; possibly cross-language.

## Failure modes & semantics

- **Client-side**: server unreachable → retries → timeout/exception; client must handle (retry, alternate server, degrade).
- **Server-side**:
  - Server does work, then **response lost** (crash/network).
  - **Replays** can cause **non-idempotent** side effects to happen twice.
  - Some servers **reject duplicates** (anti-replay/security).
  - Load-balanced replicas complicate replays.
- RPC ≠ regular call semantically, even if it looks similar.
- Same classes of problems also occur with **REST**—just more obvious (HTTP).

## Architectures

- **Client–Server / Request–Response** (applies to REST and RPC).
- **N-tier**: caches, gateways, clusters, DBs—real systems have layers.
- **Broker/API gateway**: client knows gateway, not server; broker routes/returns.
- **Peer-to-Peer (P2P)**: nodes act as clients/servers.
- **SOA / Microservices**: similar idea; smaller vs larger services; often brokered.
- **Publish–Subscribe (Event-Driven)**:
  - Like “Twitter for computers.”
  - **Push** model common now; older systems polled.
  - **Asynchronous**: callbacks/handlers for events.

## Sync vs async

- RPC typically **synchronous** call/return (with timeouts/exceptions).
- Pub/Sub **asynchronous** by nature.

------

# gRPC (specific RPC framework)

- Popular, open source (originated at Google); multi-language interop.
- Uses **Protocol Buffers** (binary) vs REST’s common **JSON**—typically faster over the wire; requires (de)serialization.
- **Streaming** support:
  - Client-streaming, server-streaming, bidirectional streaming.
  - Mix-and-match with unary calls.
- **Deadlines**: clients specify “must get response within X” or “don’t bother.” Introduces distinct server/client-side deadline behaviors.
- Choosing:
  - **REST**: easier to learn; common in business/consumer apps.
  - **RPC/gRPC**: strong fit for **systems/performance-sensitive** use cases.

------

# Selected Q&A

- **Mixed OS (Mac/Windows) in a team?** “Could be a problem” (tooling/build parity); recommend avoiding if possible.
- **REST ↔ gRPC bridge?** Possible but “weird;” specifics depend on chosen bridge/tooling; not a course requirement.
- **APIs in projects?** Yes—encouraged; avoid pass-through wrappers.
- **Grading release**: see logistics above.

------

# Closing / office hours

- Instructor left Zoom main room open longer so breakout rooms wouldn’t be forcibly closed.
- Team formation submissions: “Please get yours in as soon as you can” so CourseWorks team spaces can be set up (only one person needs to submit proposals later).
- Project demo day listed with “100,000 points” but pass/fail in reality.

------

