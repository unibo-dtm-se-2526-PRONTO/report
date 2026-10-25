---
title: Requirements
has_children: false
nav_order: 3
---

# Requirements

## User stories

1) As a student, I want to register with my personal data and institutional e-mail, so that I can be identified when I book an appointment. 

2) As an employee, I want to register with my institutional e-mail, so that I can work on the appointments of my office. 

3) As an employee, I want to declare and update my weekly shifts, so that students can book slots only when I am available. 

4) As a student, I want to ask a question in free text without choosing an office, so that I can get help even if I do not know which office is responsible. 

5) As a student, I want to receive the correct answer to my inquiry, so that I can solve my problem immediately without booking. 

6) As a student, I want to say whether the suggested answer solves my problem, so that no appointment is created when it does.

7) As a student, I want to book a free slot with the right office when the answer is not enough, so that I can talk to an employee who already knows my question.

8) As a student, I want my booking to be guaranteed once it is confirmed, so that I never find the same employee booked by someone else at the same time.

9) As an employee, I want incoming appointments to be spread evenly among my colleagues, so that nobody in the office is overloaded. 

10) As a student or employee, I want to cancel an appointment before it starts, so that the slot is freed for someone else.

11) As an employee, I want to mark an appointment as completed after the meeting, so that I can keep track of what is still to be done.

12) As a student or employee, I want to be notified by e-mail when an appointment is booked or cancelled.

13) As an administrator, I want to oversee all appointments and maintain users, offices and FAQs.


## Requirements analysis

### Functional requirements

- **FR1** (US1): A student can register, providing name, surname, student ID, course of study and institutional e-mail, and can then authenticate with e-mail and password.

- **FR2** (US2): An employee can register, providing name and surname, and can then authenticate with e-mail and password. The office the employee works for is chosen once, after registration, as the first step of declaring availability (FR4).

- **FR3** (US13): Authenticated users can only access the actions and data pertinent to their role (student or employee). A third role, administrator, is reserved for the staff who run the helpdesk: it cannot be obtained through registration, and administrators oversee all appointments and maintain offices and FAQs.

- **FR4** (US3): An employee can declare their office and weekly shifts; the system computes the bookable time slots of an office on demand, from the shifts declared by its employees, rather than storing them as separate records.

- **FR5** (US3): An employee can update their declared shifts after registration, adding or removing availability.

- **FR6** (US4): A student can ask a question without choosing an office.

- **FR7** (US5, US6): The system suggests the best-matching FAQ answer for the submitted question, if a sufficiently relevant one exists, and the student can mark it as resolving their request.

- **FR8** (US7): If the student is not satisfied by the suggested answer, they can browse the available time slots of the office the answer belongs to, or of an office they pick from the list if no answer was found, and book one, with the original question attached.

- **FR9** (US8): The system prevents two students from being assigned the same employee for the same time slot, even under simultaneous requests.

- **FR10** (US9): The system distributes incoming appointment requests evenly among the employees of the target office who are available in the requested slot.

- **FR11** (US10): A booked appointment can be cancelled, before it starts, by the student who booked it, by the employee assigned to it, or by an administrator.

- **FR12** (US11): An employee can mark a booked appointment as completed once the corresponding meeting has taken place.

- **FR13** (US12): The system notifies the student by e-mail confirming that their appointment has been booked.

- **FR14** (US12): The system notifies the assigned employee by e-mail when a new appointment is booked with them, including the FAQ answer the student was suggested and found unsatisfactory, if any.

- **FR15** (US12): When an appointment is cancelled, the system notifies by e-mail whoever did not cancel it: the employee if the student cancelled, the student if the employee cancelled, and both if an administrator did.

### Non-functional requirements

- **NFR1:** The booking mechanism must guarantee that no employee ever holds two active appointments on the same time slot, even under simultaneous requests.

- **NFR2:** Under normal load, FAQ matching, slot search and appointment booking should each take at most a few seconds.

- **NFR3:** Passwords are never stored in clear text; authenticated requests carry a token issued at login and revoked at logout.

- **NFR4:** The FAQ knowledge base, derived from real historical helpdesk data, is anonymised before being imported, removing any information that could identify the original requester.

- **NFR5:** The development, test and CI environments must be reproducible across machines, so that the observed behaviour does not depend on who runs it or where.

- **NFR6:** Both backend and frontend codebases are covered by automated static analysis (type-checking, linting) and automated tests.

### Implementation requirements

- **IR1**: The backend is implemented in Python with Django and Django REST Framework (DRF).

    *Reason: Economic*. Django's ORM and migrations, authentication, the e-mail framework and the admin interface come with the framework, while DRF adds serialisation, authentication and permissions for the REST API consumed by the frontend.

- **IR2**: Data is persisted in PostgreSQL, accessed through the Django ORM, with the schema managed by Django migrations. Automated tests run on an in-memory SQLite database by default, and on PostgreSQL when `TEST_DATABASE_URL` is set, as CI does.

    *Reason: Economic*. Consistency is best tested against a real DBMS with real transactional isolation, so that correctness issues do not surface for the first time in production, where they cost more to fix. SQLite keeps local test runs free of any database server, while the tests that need PostgreSQL, such as those of the full-text FAQ matching, run on it in CI.

- **IR3**: The backend, its PostgreSQL database and the Chroma vector store (IR4) are containerized with Docker and orchestrated with Docker Compose: the database runs on the `postgres:16` image and Chroma on the `chromadb/chroma` image, both with a health check, and the backend starts, applying the migrations, only once both report healthy. The frontend is not part of the backend's Compose setup. In CI, the tests run against a PostgreSQL service container based on the same image, while the tests of semantic matching replace the Chroma server with an in-memory Chroma client and the embedding model with a small stand-in.

    *Reason: Economic*. Containerization keeps the database used in development, test, and CI identical, cutting the time spent chasing environment-specific bugs.
- **IR4**: FAQ matching combines two techniques in cascade, behind a common matching interface, over the published FAQs of the anonymised Q&A dataset:
    1. **PostgreSQL full-text search** goes first. The question is analysed with the `italian` or `english` text search configuration, according to its language, and ranked against each FAQ with the FAQ's question weighted above its answer; a FAQ is suggested if its rank reaches a configurable threshold (`FAQ_MATCH_MIN_RANK`, 0.45).
    2. **Semantic search with Chroma** is asked only when full-text search finds nothing relevant enough. The question is turned into an embedding by a local multilingual model (`paraphrase-multilingual-MiniLM-L12-v2`, run through `fastembed` on ONNX Runtime) and compared, by cosine similarity, with the embeddings of the FAQs' questions in the same language, stored in the Chroma vector store; a FAQ is suggested if the similarity reaches its own configurable threshold (`FAQ_SEMANTIC_MIN_SIMILARITY`, 0.7).

    The two scores live on different scales and are never compared with each other; every recorded question keeps which of the two techniques found its answer. The cascade runs on the FAQs of every office; when the client passes an office, it runs on that office first and on all offices only if it finds nothing there. Pronto's frontend never passes one. Both thresholds were calibrated on the real FAQ dataset. The vector store is kept in sync with the published FAQs whenever one is saved or deleted, once the transaction commits, and can be rebuilt from the database with the `rebuild_faq_index` command; if Chroma cannot be reached, the error is logged and matching falls back to full-text search alone, so the student still gets an answer or the way to book.

    *Reason: Economic*. Both PostgreSQL full-text search and Chroma are free/open-source, and full-text search reuses the existing database infrastructure; computing the embeddings locally avoids the cost of a paid embedding service, and keeps students' questions on the project's own infrastructure. `fastembed` runs the model without PyTorch, keeping the backend's Docker image small.
- **IR5**: Authentication uses DRF token authentication (`TokenAuthentication`): at login the backend issues a random token, stored in the database, which the client sends with every request and which is deleted at logout. Passwords are hashed with Django's password hashing (PBKDF2 by default) and checked against Django's password validators at registration.

    *Reason: Economic*. Both mechanisms ship with Django and DRF, so no additional library is needed. A token sent in a request header suits a REST API consumed by a separate single-page frontend, and well-tested framework code keeps implementation and maintenance effort low.
- **IR6**: The frontend is implemented with Vue.js. 

    *Reason: Political*. The team already knows its tooling (Vite, Vue Router, Pinia). This is an internal team decision; no technical constraint required Vue.js.
- **IR7**: E-mail notifications are sent through Django's e-mail framework: by default e-mails are printed to the console (development), and they are delivered via SMTP when configured through environment variables. Booking e-mails are sent only after the database transaction commits, on a best-effort basis: a failed e-mail is logged and does not undo the booking.

    *Reason: Economic*. The e-mail framework is part of the backend, so no separate notification service has to be deployed or integrated. The console backend lets developers try the flows without an SMTP account.
- **IR8**: Automated testing relies on `pytest`/`pytest-django`/`coverage` on the backend, and on an equivalent `Vitest`/`ESLint`/`Prettier` toolchain on the frontend. 

    *Reason: Administrative*. Quality assurance requires automated tests on both the frontend and the backend.

- **IR9**: Source control follows Conventional Commits and a Gitflow-inspired branching model: a stable branch (`master` in the backend repository, `main` in the report repository), an integration branch `develop`, and short-lived branches named after the kind of change (`feature/`, `fix/`, `refactor/`, `test/`, `docs/`, `chore/`), each merged into `develop` through a pull request. CI runs via GitHub Actions on every push and pull request; once a change lands on `develop` and the tests pass, CI also applies the pending migrations to the database the team shares (hosted on Neon), so that its schema always matches `develop`.

     *Reason: Administrative*. The goal is to keep a clear, reproducible development history, and eventually to automate the release.

## Acceptance criteria

- **AC1** (FR1): given valid, unique registration data, a new student account is created. Registration is rejected if the e-mail does not belong to the `@studio.unibo.it` domain or is already registered.
- **AC2** (FR2): given valid, unique registration data with an e-mail belonging to the `@unibo.it` domain, a new employee account is created; registration is rejected otherwise. Choosing an office a second time is rejected.
- **AC3** (FR3): a student cannot access employee-only actions (e.g. declaring availability), and vice versa; a student or employee only sees their own appointments, and someone else's appointment is reported as not found; unauthenticated requests to protected endpoints are rejected.
- **AC4** (FR4): given a declared shift (e.g. Monday 9:00–12:00) and a fixed slot duration, browsing that office's availability yields a sequence of non-overlapping, contiguous candidate slots covering the whole shift.
- **AC5** (FR5): since time slots are derived from shifts rather than stored, updating a shift immediately changes the availability computed for subsequent browsing.
- **AC6** (FR6): given a free-text question, the question is immediately compared against the FAQs of every office and recorded together with the suggested FAQ, if any, under that FAQ's office, without any link to the student who asked it.
- **AC7** (FR7): given a question with at least one sufficiently similar FAQ entry, the corresponding answer is shown to the student together with the office it belongs to; no suggestion is shown if no sufficiently relevant match is found. The API also accepts an optional office: when one is given, its FAQs are searched first, and those of every office only if none of them is relevant enough.
- **AC8** (FR8): after booking, the appointment is created with an initial "booked" status, the attached question, and exactly one employee of the office assigned to handle it; the slot keeps being offered to other students as long as at least one employee of the office is still free in it, and disappears from the office's availability once all of them are booked. No appointment is created if the student marks the suggested answer as satisfactory.
- **AC9** (FR9): when several booking requests for the same office and time slot are issued at the same time, each either succeeds with a distinct employee assigned, or is rejected; the number of successful bookings never exceeds the number of employees of that office available in that slot.
- **AC10** (FR10): given N employees of the same office sharing the same declared shifts, after a sequence of consecutive bookings on that office the difference between the number of active appointments held by any two of them never exceeds one. Under simultaneous requests the distribution may be briefly unbalanced.
- **AC11** (FR11): given a booked appointment that has not started yet, cancelling it transitions the appointment to the "cancelled" status and frees the corresponding slot; cancelling an appointment that has already started, or that is not in the "booked" status, is rejected.
- **AC12** (FR12): given a booked appointment whose slot has started, the handling employee can mark it completed, transitioning it to the "completed" status; marking it before the slot starts is rejected. Completing an appointment sends no e-mail.
- **AC13** (FR13): booking an appointment sends exactly one e-mail to the student, with the office, date/time and question, in the language the question was asked in.
- **AC14** (FR14): booking an appointment sends exactly one e-mail, in Italian, to the assigned employee, with the office, date/time, question and suggested FAQ answer, if any.
- **AC15** (FR15): cancelling an appointment sends exactly one e-mail to each recipient listed in FR15, and none to whoever cancelled it; the student receives it in the language of the question, the employee in Italian.
- **AC16** (NFR1): no employee is ever assigned two active appointments on the same slot, a rule enforced by the database itself and verified by tests issuing simultaneous bookings.
- **AC17** (NFR2): under normal load, FAQ matching, slot search and booking each answer within a few seconds.
- **AC18** (NFR3): passwords are stored only as hashes; a request without a valid token is rejected, and a token stops working after logout.
- **AC19** (NFR4): the `import_faqs` command replaces with placeholders e-mail addresses (except those on the institutional `@unibo.it` domain, which belong to the university, not to the requester), phone numbers, student ID numbers (matricole), Italian tax codes, and person names introduced by a cue (an honorific, a self-introduction or a closing sign-off). Name detection is heuristic, so imported FAQs stay unpublished until a staff member has reviewed them in the Django admin, unless the import is explicitly run with `--publish`.
- **AC20** (NFR5): the Poetry version is pinned in a single place (the backend's `Dockerfile`), from which CI reads it; dependencies are locked in `poetry.lock`; the Docker image and CI both run Python 3.12; Docker Compose and CI both use PostgreSQL 16; and the test suite can run on PostgreSQL, the same engine as production, not only on the default SQLite.
- **AC21** (NFR6): every push runs type-checking, linting and the automated tests of both codebases, and reports test coverage.

## Glossary

| Term | Meaning |
|---|---|
| Student | A user who needs information or assistance from a university office and can book appointments. |
| Employee | A staff member of a university office who declares availability and answers booking requests. |
| Administrator | A member of the helpdesk staff who oversees all appointments and maintains users, offices and FAQs; the role cannot be obtained through registration. |
| Office | An administrative unit of the Cesena Campus (guidance, student administration office, international relations, internships) that students can book appointments with. |
| Shift | A stretch of a weekday during which an employee is on duty, repeated every week and declared by the employee. |
| Time slot | A fixed-duration window of time that a student can book with an office, derived from the shifts declared by the office's employees. |
| Assignment | The binding of a booked appointment to the specific employee who will handle it, decided by the system rather than chosen by the student. |
| Appointment | The booking of a time slot by a student, together with the attached question and its lifecycle status (booked, cancelled, or completed). |
| Matricola | The Italian term for a student's university identification number. |
| FAQ | An anonymised question/answer pair from the historical helpdesk dataset, used to automatically suggest answers. |
| Inquiry | A question asked by a student, recorded with the suggested FAQ and whether it resolved the request, without any link to the student. |
| Match | The outcome of comparing a student's question against the FAQ collection, together with a relevance score. |

## Use-case diagram

<p align="center"><img src="assets/use-case-diagram.svg" alt="Use case diagram"></p>
