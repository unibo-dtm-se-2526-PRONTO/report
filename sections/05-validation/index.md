---
title: Validation
has_children: false
nav_order: 6
---

# Validation

## Testing approach

We wrote the tests together with the code. The test layout of the backend was set up right after the Django project itself, before the first feature, and since then every feature commit carries the tests of the behaviour it adds: the booking service, full-text matching and the anonymiser, for instance, each arrived in the same commit as their tests. We did not follow strict TDD (writing each test before the code it checks), but we wrote the tests from the acceptance criteria of the [requirements](../02-requirements/index.md), and the docstring of each test file names the requirements it covers (e.g. `test_booking_notifications.py` covers FR13–FR15, `test_booking_concurrency.py` covers NFR1).

Each test is named after the behaviour it checks, written as a sentence: `test_a_slot_disappears_when_every_employee_is_booked`, `test_a_failed_email_does_not_undo_the_booking`. Read one after the other, the names of a test file describe the rules of the component, and a failing test says which rule broke without opening it.

A few rules hold across both suites:

- **No test depends on the date it runs.** Backend tests build their dates relative to today (`next_working_day()` in `conftest.py` returns the next Monday), so the suite does not start failing once a hard-coded date is in the past. The frontend suite runs in the `Europe/Rome` time zone on every machine and freezes the clock where a test needs a fixed "now".
- **No test reaches the network.** The frontend's `fetch` is always replaced by a mock, and a request that no test set up fails the test even if the application caught the error; the API base URL points to a reserved host that cannot resolve. In the backend, tests never reach a Chroma server or download the embedding model.
- **Behaviour that only PostgreSQL has is tested on PostgreSQL.** Full-text search does not exist on SQLite, and SQLite locks the whole database on every write, so a concurrency test there would pass for the wrong reason. These tests are marked `@pytest.mark.postgres` and are skipped, with that reason in the report, unless the suite runs on PostgreSQL (see [Integration testing](#integration-testing)).

**Frameworks.** The backend uses **pytest** with **pytest-django** and **coverage**. We preferred pytest to Django's own `unittest`-based runner because its fixtures compose: `conftest.py` defines once the cast that the booking tests share (an office, an employee working Monday to Friday from 9 to 17, a student, a future weekday), and each test asks only for what it needs. `parametrize` turns a list of inputs into separate tests (the anonymiser's 58 tests are mostly cases of a few rules), custom markers select the PostgreSQL-only tests, and pytest-django adds what a Django project needs: a test database, `settings` overrides, and `django_capture_on_commit_callbacks` to run the code that Django defers until a transaction commits.

The frontend uses **Vitest** with **Vue Test Utils** and **jsdom**. Vitest reuses the Vite configuration of the application, so single-file components, TypeScript and the `@/` import alias work in the tests with no separate build setup; jsdom gives the components a browser DOM, and Vue Test Utils mounts them and drives them as a user would (filling fields, clicking buttons).

## Testing (automated)

The figures below come from a run on 28 October 2026 (backend at commit `e86ae95` of `develop`, frontend at commit `99a56f0`). Coverage of the backend counts the application code only: tests and migrations are excluded.

| Suite | Tests | Passed | Skipped / expected failures | Coverage |
|---|---|---|---|---|
| Backend, on PostgreSQL 16 | 468 | 468 | 0 | 98% of statements (1253, 22 not run) |
| Backend, on SQLite | 468 | 439 | 29 skipped (marked `postgres`) | 97% of statements |
| Frontend | 221 | 219 | 2 expected failures (known bugs, see below) | 99.7% statements, 97.4% branches, 98.8% functions, 100% lines |

Both suites run in CI on every push and pull request (see [CI/CD](../08-cicd/index.md)). The backend's coverage is measured on the PostgreSQL run, the only one that reaches the FAQ matching. The frontend's CI fails if coverage drops below 95% of lines, statements and functions, or 90% of branches.

The two expected failures are tests that describe a known bug. They are written with Vitest's `it.fails`, so they pass while the bug is there and will fail, as a reminder to turn them into ordinary tests, once it is fixed:

- signing out when the logout request fails leaves the user on a protected page with an unhandled error (`App.spec.ts`), because `signOut()` awaits a call that rethrows the failure and never reaches the redirect to the login page;
- a registration error on a field the form does not show (such as DRF's `non_field_errors`) is stored as a field error, so the user sees no message at all (`RegisterView.spec.ts`).

The table below traces the requirements to the backend tests that check them. Some files cover more than one requirement; each is listed under its main one.

| Requirement | Test files (backend) | Tests |
|---|---|---|
| FR1, FR2, NFR3: registration, login, passwords | `test_accounts_register`, `test_accounts_auth`, `test_accounts_models`, `test_accounts_admin`, `test_accounts_migrations` | 112 |
| FR3: role-based access | role and ownership tests spread across `test_booking_appointments`, `test_booking_staff`, `test_faq_questions_api` (e.g. `test_only_students_ask`, `test_cancelling_someone_elses_appointment_is_not_found`) | (included in the rows of those files) |
| FR4, FR5: shifts and availability | `test_booking_shifts`, `test_booking_staff`, `test_booking_offices` | 63 |
| FR6, FR7, IR4: asking and FAQ matching | `test_faq_matching`, `test_faq_semantic`, `test_faq_questions_api`, `test_faq_api`, `test_faq_models` | 86 |
| FR8, FR10, FR11, FR12: booking, distribution, lifecycle | `test_booking_services`, `test_booking_appointments`, `test_booking_models` | 75 |
| FR9, NFR1: no double booking | `test_booking_constraints`, `test_booking_concurrency` (and the two race tests of `test_booking_services`, counted in the row above) | 7 |
| FR13–FR15: e-mail notifications | `test_booking_notifications` | 16 |
| NFR4: anonymisation | `test_faq_anonymisation`, `test_faq_import` | 95 |
| NFR5, IR2: environment and setup | `test_postgres`, `test_project_setup`, `test_health`, `test_offices_models`, `test_offices_seed` | 14 |

On the frontend, the same requirements are covered from the user's side: registration and login (`RegisterView`, `LoginView`, `auth-store`), role guards (`router`, 30 tests), asking and confirming an answer (`AskView`, `ConfirmView`), slots and bookings (`SlotsView`, `BookingsView`, `booking-store`), shifts and the employee queue (`AvailabilityView`, `ProfileView`, `QueueView`).

### Unit testing

The units we test in isolation are the domain rules and the pure functions:

- **Anonymisation** (`faq/anonymisation.py`, 58 tests). Each kind of personal data has its own tests: a personal e-mail address is replaced, an institutional office mailbox is kept, a subdomain of the institutional domain is not; matricole, phone numbers, tax codes, and names introduced by an honorific, a self-introduction or a sign-off are replaced. Two further tests check that ordinary text is left alone and that anonymising twice changes nothing more.
- **Booking rules** (`booking/services.py`, 31 + 30 tests in `test_booking_services` and `test_booking_shifts`). These tests call the services directly, with no HTTP and no serializers, and pin down when a slot exists, who gets a booking and when a transition is allowed: slots follow the office's slot duration and the employees' shifts, a slot stays available while one employee is free and disappears when all of them are booked, consecutive bookings are spread evenly (FR10), a booking outside a shift, in the past or off the slot grid is refused, and only a booked appointment that has not started can be cancelled.
- **Matchers** (`faq/matching.py`). `SemanticMatcher` and `CascadeMatcher` are tested one at a time: the first matcher that finds an answer wins, the next is asked only when one finds nothing, and the cascade finds nothing only when no matcher does.
- **Models and constraints**: uniqueness of e-mails and matricole, required names, what may and may not be deleted, and the partial unique constraint behind FR9 (`test_booking_constraints`: an employee cannot hold two booked appointments on one slot, while a cancelled one does not block it).

Model and service tests do use the test database. We chose not to mock the ORM there, because several rules (uniqueness, protected deletion, the booking constraint) are enforced by the database itself, and a mock would only repeat what the test assumes.

On the frontend, the unit tests cover the date and time formatting (`format`), the translation module (`i18n`), the HTTP client (`client`: base URL, token header, error handling), the endpoint functions (`endpoints`: each one sends the method, path, query and body the backend expects), the three Pinia stores and the two shared components.

### Integration testing

We tested the following components together.

**API, services and database.** The endpoint tests go through the whole backend stack with DRF's `APIClient`: URL routing, token authentication, permission classes, serializers, services and the ORM. They check what a client sees: status codes, response bodies, the language of the response, and that data is scoped to its owner (`test_an_employee_does_not_see_a_colleagues_appointments`). The booking rules themselves are tested once, in the services; the endpoint tests only check that the API applies them and maps their errors to the right response.

**Matching and PostgreSQL full-text search** (`test_faq_matching`, 14 tests). These run on a real PostgreSQL database with the `italian` and `english` configurations: Italian words are matched by their stem, a question made only of stop words matches nothing, an unpublished FAQ is never suggested, a good-enough match in the chosen office wins over a better one elsewhere, and a score equal to the threshold counts as a match. `test_postgres` checks that the PostgreSQL run really is one, with both text-search configurations installed.

**Concurrent bookings and PostgreSQL** (`test_booking_concurrency`, 5 tests). Several threads call the same service at the same moment, each on its own database connection and in its own transaction, as simultaneous HTTP requests would: a barrier releases them together so that their calls overlap. The tests use `transaction=True`, since the usual test transaction would hide each thread's writes from the others. They check that simultaneous bookings never give one employee two appointments on the same slot, that two overlapping shifts declared at once are not both accepted, that a shift withdrawn during a booking leaves no appointment outside a shift (and cannot be withdrawn under a booking not yet committed), and that two simultaneous cancellations send the e-mails only once.

**Booking and notifications** (`test_booking_notifications`, 16 tests). The e-mails go out once the transaction commits, so the tests capture Django's on-commit callbacks and read the sent messages from the test mail outbox. They check that each transition sends exactly one e-mail to each recipient listed in FR13–FR15, with office, time and question; that the student is written to in the language of their question and the employee in Italian; that a booking rolled back sends nothing; and that a failed e-mail neither undoes the booking nor stops the other e-mail.

**FAQ, vector index and Chroma** (`test_faq_semantic`, 28 tests). A published FAQ is indexed in both languages after the commit and not before, unpublishing or deleting it removes it, editing it updates its entry, a FAQ is still saved when the vector store is down, and `rebuild_faq_index` rebuilds the index from the database, keeping the entries it had if it fails halfway. The end-to-end cases of the cascade (full-text first, semantic search for what full-text misses) run on PostgreSQL.

**FAQ import** (`test_faq_import`, 37 tests). The `import_faqs` command is run on real `.xlsx` files written by the tests: questions and answers are anonymised in both languages, imported FAQs stay unpublished until reviewed, running the import twice creates nothing new, and a choice made in the admin (a published FAQ, an English text written by hand) survives a later import. A failure halfway leaves the database as it was.

**Frontend views, stores and router.** Each view is mounted with its real store and router and a mocked backend, and driven as a user would: the tests fill the form, submit it, and check both the request the view sent and what it shows for each possible response, errors included.

**Test doubles.** We replaced a component only where the real one is slow, external or cannot be made to fail on demand:

| Double | Kind | Replaces | Why |
|---|---|---|---|
| Django's in-memory mail backend (`mail.outbox`) | fake | the SMTP server | e-mails are read back and counted, nothing is delivered |
| `chromadb.EphemeralClient` | fake | the Chroma server | the real Chroma library, in memory, with no server to start |
| `fake_embed` | fake | the embedding model | a small dictionary of synonyms ("certificato", "attestato") is enough to tell a question worded differently from an unrelated one; how well the real model does that is a matter of calibration, checked on the real dataset |
| `store_down`, `model_unavailable`, `smtp_is_down` | stubs | the vector store, the model, `send_mail` | they raise the error of an unreachable service, to test the fallbacks |
| `lose_the_first_pick_to` | stub | the employee selection | it books the chosen employee for a rival just before the insertion, so the retry with a colleague can be tested on SQLite, one request at a time |
| `mockApi()` | fake and spy | the backend, behind `fetch` | it answers by method and path and records every request, so tests assert on what the application sent |
| `freezeTime()` | fake | the system clock | dates such as "today" and "already started" stay fixed |

PostgreSQL is never replaced by a double: the tests that need it run on a real PostgreSQL server.

### System testing

On the frontend, `flows.spec.ts` runs whole journeys through the real application: `App.vue`, the real router with its guard and lazy-loaded pages, and the real stores, with only the backend mocked. It covers three journeys that follow the acceptance criteria of the main user stories:

- a student signs in, asks a question, gets no answer, books a slot and finds it among their bookings (US2a, US2d, FR8);
- a student is kept out of the employee pages (FR3);
- an employee signs in to their queue and signs out again.

On the backend, the endpoint tests already treat the backend as a black box reached over HTTP, and the concurrency tests reproduce the situation of NFR1 with real concurrent transactions.

No automated test drives a browser against the real backend. The contract between the two is checked from both sides instead: the frontend types in `src/api/types.ts` mirror the backend serializers and are checked by `vue-tsc`, `endpoints.spec.ts` checks the requests the frontend sends, and the backend's endpoint tests check the responses those requests get. The full journey across the two runs in the manual acceptance tests below.

**Containers.** The backend's PostgreSQL tests run on the `postgres:16` image, the same one Docker Compose uses for the `db` service. In CI it is a service container started by GitHub Actions for the test job, with a health check, so every run starts from an empty database. Locally a developer starts the same container and points the suite at it:

```bash
docker compose up -d --wait db
TEST_DATABASE_URL=postgres://pronto:pronto@localhost:5432/pronto?sslmode=disable \
    poetry run poe test
```

Django creates its own `test_pronto` database on that server and drops it at the end, so the data of the development database is never touched. The frontend suite needs no container; CI runs it on Linux, Windows and macOS with Node 20, 22 and 24.

## Acceptance tests (manual)

The manual tests run the whole system as its users would, through the browser, and follow the acceptance criteria of the requirements.

**Setup.** From the backend repository, with the frontend repository next to it:

```bash
docker compose up --build
docker compose exec backend python manage.py seed_offices
docker compose exec backend python manage.py import_faqs <helpdesk-export>.xlsx --publish
docker compose exec backend python manage.py rebuild_faq_index
docker compose exec backend python manage.py createsuperuser
```

The application is at <http://localhost:5173> and the admin at <http://localhost:8000/admin/>. E-mails are printed by the backend, and `docker compose logs -f backend` shows them. The tests need one student account (`@studio.unibo.it`), two employee accounts (`@unibo.it`) working for the same office, and the administrator; the tests marked "two browsers" use a second browser or a private window, so that two users are signed in at once.

| # | Requirement | Steps | Expected result |
|---|---|---|---|
| AT1 | FR1, FR2 | Register a student, an employee, and a user with a `@gmail.com` address. | The first two accounts are created and can sign in at once; the third is refused. |
| AT2 | FR2, FR4 | Sign in as each employee, choose the office, declare a shift (e.g. Monday 9:00–12:00). Try to choose the office again. | The shift is listed; choosing the office a second time is refused. |
| AT3 | FR4, FR5 | As the student, open that office's availability for the Monday. As one employee, withdraw the shift, then reload the student's page. | The slots cover 9:00–12:00 in steps of the office's slot duration; after the withdrawal they are gone, unless the other employee still covers them. |
| AT4 | FR6, FR7 | As the student, ask a question taken from an imported FAQ, worded differently. | The matching answer is shown. |
| AT5 | FR7 | Ask a question that belongs to another office. | The answer is shown together with the office it belongs to. |
| AT6 | FR8 | Ask an unrelated question, or reject the suggested answer, then book a slot. | The appointment is listed as booked, with the question attached. |
| AT7 | FR8 | Ask a question and mark the answer as satisfactory. | The flow ends; no appointment is created. |
| AT8 | FR10 | Book two consecutive slots as the student. | Each employee of the office receives one appointment. |
| AT9 | FR9 (two browsers) | Sign in as two students and book the same slot at the same time; then book it a third time. | The first two bookings go to different employees; the third is refused, and the slot is no longer offered. |
| AT10 | FR13, FR14 | Check the backend log after AT6. | One e-mail to the student, in the language of the question, and one to the employee, in Italian, with the question and the FAQ that did not help. |
| AT11 | FR11, FR15 | Cancel a booking as the student, another as the employee, and a third as the administrator. The administrator has no page in the application, so they sign in through the API (`POST /api/auth/login/`) and call `POST /api/appointments/<id>/cancel/` with the token they receive. | Each is cancelled and its slot is offered again; the e-mail goes to the employee, to the student, and to both, respectively. |
| AT12 | FR12 | As the employee, try to complete an appointment that has not started; complete one whose slot has started. | The first is refused; the second becomes completed, and no e-mail is sent. |
| AT13 | FR3 | As the student, open an employee page by typing its URL. Sign in as the administrator in the application. | The student is sent back to their own home page; the administrator is sent to the Django admin. |
| AT14 | NFR4 | In the admin, open the FAQs imported without `--publish`. | They are unpublished, and the personal data in their text is replaced by placeholders. |
| AT15 | IR4 | Stop the Chroma container (`docker compose stop chroma`) and ask the question of AT4 again. | The student still gets an answer from full-text search, or the way to book; the backend logs the error. |

<!-- Results: fill in the date of the last run and the outcome of each test (e.g. "15/15 passed on DD/MM/2026"). -->
