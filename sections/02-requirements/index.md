---
title: Requirements
has_children: false
nav_order: 3
---

# Requirements

## User stories

Pronto is used by two personas: the **Student**, who needs information or assistance from a university office, and the **Employee**, a staff member of that office.

- **US0 (Student) — Registration.** As a student, I want to create a profile with my personal data (name, surname, student ID, course of study, institutional e-mail), so that I can be identified when booking an appointment.
    - For the scope of this project, the student's institutional e-mail domain (`@studio.unibo.it`) is used to automatically validate the student role at registration time, in place of a full integration with the university's identity provider (Shibboleth/SAML), which would require an official authorization out of scope for a course project.
- **US1 (Employee) — Registration and availability.** As an employee, I want to create a profile indicating my office and working shifts, so that the system can automatically generate the bookable time slots derived from them.
    - Symmetrically to US0, the employee role is validated through the institutional e-mail domain (`@unibo.it`), rather than through manual approval by an administrator.
    - Declaring availability is a separate step from registration, so that an employee can update their shifts afterwards without having to register again.
- **US2 (Student) — Booking an appointment.** As a student, I want to book an appointment with the appropriate office, choosing a day and time among the available slots, so that I can get help with my request. This is the core story of the system and is refined into the following sub-stories:
    - **US2a — Slot selection.** The student picks an office and selects one of its available time slots.
    - **US2b — Question attachment.** Before confirming, the student attaches a free-text question to the booking.
    - **US2c — FAQ matching.** As soon as the question is submitted, the system compares it against a knowledge base of real, anonymised past questions and answers and, if a relevant match is found, proposes it to the employee as a suggested answer.
    - **US2d — Employee response.** The employee sees the booking, the student's note, and the FAQ suggestion (if any), and can answer directly. If the student confirms that the answer resolved their request, the appointment is automatically cancelled; otherwise, it remains confirmed.
    - **US2e — Concurrency control.** Two students must never be able to book the same time slot at the same time.
    - **US2f — Equal distribution.** As a student, I only see the available slots of an office, not the individual employee handling them, so that the system is free to distribute incoming requests evenly among the office's staff.
    - **Notifications.** As a student or employee, I want to be notified by e-mail whenever the state of an appointment changes (question answered, appointment confirmed or cancelled, new booking received), so that I do not have to keep the application open to know the outcome of my request.

An additional story was considered and consciously **descoped**: a general-purpose chatbot answering open questions about calls for applications, deadlines, study plans, or enrollment procedures by retrieving information from the university website (a Retrieval-Augmented-Generation-based assistant). Given the already broad scope of Pronto (authentication, scheduling, concurrency control, FAQ matching, staff load-balancing, notifications) and the additional risk this story would introduce — dependency on scraping an external website, unpredictable answer quality, per-query API costs — it was left out of the current iteration in favor of a smaller, better-tested feature set.

## Requirements analysis

### Functional requirements

**Identity & access**

- **FR1**: A student can register, providing name, surname, student ID, course of study, and institutional e-mail, and can subsequently authenticate with e-mail and password.
    - *Acceptance criteria*: given valid, unique registration data, a new student account is created; the student can then log in with the same credentials. Registration is rejected if the e-mail does not belong to the `@studio.unibo.it` domain or is already registered.
- **FR2**: An employee can register, providing name, surname, and office, and can subsequently authenticate with e-mail and password.
    - *Acceptance criteria*: given valid, unique registration data with an e-mail belonging to the `@unibo.it` domain, a new employee account is created; registration is rejected otherwise.
- **FR3**: Authenticated users can only access the actions and data pertinent to their role (student or employee).
    - *Acceptance criteria*: a student cannot access employee-only actions (e.g. declaring availability), and vice versa; unauthenticated requests to protected endpoints are rejected.

**Availability management**

- **FR4**: An employee can declare their office and working shifts; the system automatically generates the corresponding bookable time slots.
    - *Acceptance criteria*: given a declared shift (e.g. Monday 9:00–12:00) and a fixed slot duration, the system generates a sequence of non-overlapping, contiguous slots covering the whole shift.
- **FR5**: An employee can update their declared shifts after registration, adding or removing availability.
    - *Acceptance criteria*: newly added shifts generate new bookable slots; removed shifts make the corresponding, not-yet-booked slots unavailable.

**Booking**

- **FR6**: A student can browse the available time slots of a given office.
    - *Acceptance criteria*: only slots that are within a shift and not already booked are shown to the student.
- **FR7**: A student can book an available slot, attaching a free-text question.
    - *Acceptance criteria*: after booking, the slot is no longer offered to other students, and the appointment is created with the attached question and an initial "booked" status.
- **FR8**: The system prevents two students from booking the same time slot concurrently.
    - *Acceptance criteria*: when two booking requests for the same slot are issued at the same time, exactly one succeeds and the other is rejected with a clear conflict response.

**FAQ matching**

- **FR9**: When a booking question is submitted, the system compares it against a knowledge base of anonymised past questions and answers and, if a relevant match exists, proposes it to the employee as a suggested answer.
    - *Acceptance criteria*: given a question sufficiently similar to an existing FAQ entry, the corresponding answer is shown to the employee alongside the booking; no suggestion is shown if no sufficiently relevant match is found.
- **FR10**: An employee can answer a student's question directly, optionally reusing the suggested FAQ answer.
    - *Acceptance criteria*: the employee's answer is recorded and associated with the appointment, moving it to an "answered" status.
- **FR11**: A student can confirm whether the received answer resolved their request; if confirmed, the appointment is automatically cancelled, otherwise it remains a confirmed appointment.
    - *Acceptance criteria*: confirming a satisfactory answer transitions the appointment to "cancelled" and frees the slot; declining it transitions the appointment to "confirmed".

**Distribution**

- **FR12**: The system distributes incoming appointment requests evenly among the employees of the target office.
    - *Acceptance criteria*: given multiple employees of the same office with open slots, new bookings are assigned so that no employee accumulates significantly more active appointments than their colleagues.

**Notifications**

- **FR13**: The system notifies the relevant employee by e-mail when a new appointment request is created for their office.
- **FR14**: The system notifies the student by e-mail when their question has been answered.
- **FR15**: The system notifies the student by e-mail when their appointment is confirmed or automatically cancelled.
    - *Acceptance criteria* (FR13–FR15): each listed lifecycle transition results in exactly one e-mail being sent to the correct recipient, containing enough context (office, date/time, question) to act on it without opening the application.

### Non-functional requirements

- **NFR1 — Consistency**: the booking mechanism must guarantee that no two students ever hold a confirmed appointment on the same time slot, even under simultaneous requests.
- **NFR2 — Responsiveness**: slot search and booking confirmation should complete within an interactively acceptable time (in the order of seconds) under normal load.
- **NFR3 — Security**: passwords are never stored in clear text; authenticated sessions rely on signed, expiring tokens.
- **NFR4 — Privacy**: the FAQ knowledge base, derived from real historical helpdesk data, is anonymised before being imported, removing any information that could identify the original requester.
- **NFR5 — Reproducibility**: the development, test, and CI environments must be reproducible across machines, so that the observed behaviour (in particular around concurrency) does not depend on who runs it or where.
- **NFR6 — Code quality**: both backend and frontend codebases are covered by automated static analysis (type-checking, linting) and automated tests, with coverage tracked over time.

### Implementation requirements

- **IR1**: The backend is implemented in Python with FastAPI. *Technical*: an async-first web framework fits the I/O-bound booking/notification workload and provides interactive API documentation out of the box.
- **IR2**: Data is persisted in PostgreSQL, accessed through SQLAlchemy (async) with Alembic-managed migrations. *Technical*: NFR1 is best validated against a real DBMS with genuine transactional isolation; a lighter engine would weaken exactly the guarantee Pronto needs to demonstrate.
- **IR3**: The whole stack (backend, database, frontend) is containerized with Docker and orchestrated with Docker Compose, both locally and in CI. *Administrative*: mandated by the course, since it standardizes the environment and is the prerequisite for running integration tests against a real, disposable PostgreSQL instance.
- **IR4**: FAQ matching is implemented, as a baseline, with PostgreSQL full-text search over the anonymised Q&A dataset. *Scope decision*: a vector-based semantic search (e.g. Chroma) is considered only as an optional future enhancement, since it introduces an additional moving part (an embeddings pipeline) on top of an already broad scope.
- **IR5**: Authentication uses JWT, with passwords hashed via bcrypt. *Technical*: a standard, stateless mechanism that fits a REST API consumed by a separate single-page frontend.
- **IR6**: The frontend is implemented with Vue.js. *Team decision*: chosen for the team's familiarity with its tooling (Vite, Vue Router, Pinia), with no external constraint mandating it.
- **IR7**: E-mail notifications are sent through `fastapi-mail`. *Technical*: integrates directly with the async backend without introducing a separate notification service.
- **IR8**: Automated testing relies on `pytest`/`pytest-asyncio`/`httpx`/`pytest-cov` on the backend, and on an equivalent `Vitest`/`ESLint`/`Prettier` toolchain on the frontend. *Administrative*: explicitly required by the course, so that the frontend has a JavaScript-side counterpart to the backend's `pytest`/`mypy` checks.
- **IR9**: Source control follows Conventional Commits and a Gitflow-inspired branching model (`main` / `develop` / `feature/*`), with continuous integration enforced via GitHub Actions on every pull request. *Administrative*: required by the course guidelines to keep a legible, reproducible development history.

## Glossary

| Term | Meaning |
|---|---|
| Student | A user who needs information or assistance from a university office and can book appointments. |
| Employee | A staff member of a university office who declares availability and answers booking requests. |
| Office | An administrative unit of the Cesena Campus (e.g. the job orientation office) that students can book appointments with. |
| Shift | A period of time during which an employee is available to work, declared by the employee. |
| Time slot | A fixed-duration, bookable unit of time derived from an employee's shift. |
| Appointment | The booking of a time slot by a student, together with the attached question and its lifecycle status. |
| Matricola | The Italian term for a student's university identification number. |
| FAQ | An anonymised question/answer pair from the historical helpdesk dataset, used to automatically suggest answers. |
| Match | The outcome of comparing a student's question against the FAQ collection, together with a relevance score. |

## Use-case diagram

```plantuml
@startuml
left to right direction

actor Student
actor Employee

rectangle Pronto {
    usecase "Register / Login" as UC1
    usecase "Declare office & shifts" as UC2
    usecase "Update availability" as UC3
    usecase "Browse available slots" as UC4
    usecase "Book appointment" as UC5
    usecase "Attach question" as UC6
    usecase "Receive FAQ suggestion" as UC7
    usecase "Answer booking" as UC8
    usecase "Confirm / reject answer" as UC9
    usecase "Receive notifications" as UC10
}

Student --> UC1
Student --> UC4
Student --> UC5
Student --> UC9
Student --> UC10

Employee --> UC1
Employee --> UC2
Employee --> UC3
Employee --> UC8
Employee --> UC10

UC5 ..> UC6 : <<include>>
UC5 ..> UC7 : <<include>>
UC8 ..> UC7 : <<include>>
@enduml
```