---
title: Validation
has_children: false
nav_order: 6
---

# Validation

## Testing approach

### Backend

We followed test-driven development (TDD) since the first feature: the test layout and a first health-check test were set up before the `accounts` app, and from then on each feature was developed together with its tests, writing a failing test first and then the code that makes it pass. All tests live in the `tests/` folder at the root of the repository, one file per app and topic (`test_accounts_*.py`, `test_booking_*.py`, `test_faq_*.py`, `test_offices_*.py`), with shared fixtures in `conftest.py`.

#### Automated testing tools

The tools are development dependencies declared in `pyproject.toml`:

- **pytest**, with the **pytest-django** plugin, runs the tests: its fixtures make test data easy to share, and pytest-django gives each test a fresh database and the Django settings;
- **coverage** measures which lines of the code the tests cover.

They run with `poe test` and `poe coverage`.

#### Test databases

The tests run on two databases. On a developer's machine they use a SQLite database by default: it needs no server, credentials or network, so the suite works on any machine with Python alone and the unit tests run in about half a minute, which keeps the TDD cycle short. SQLite is not the production engine, though: it has no full-text search and no real row locking. The tests that depend on them, such as full-text FAQ matching and concurrent bookings, are marked `postgres` and skipped on SQLite. 
The CI runs the suite twice: first on SQLite, to check that it still passes with no database server, then on PostgreSQL 16.

The switch happens in `pronto/settings.py`. The settings recognise a test run (pytest sets the `PYTEST_VERSION` environment variable) and, only in that case, replace the default database: if `TEST_DATABASE_URL` holds a `postgres://` URL they point to that PostgreSQL server, otherwise to SQLite in memory.

#### Test doubles

A test double stands in for a real component during a test. All the doubles in Pronto are written by hand and installed with pytest's `monkeypatch`.

| Test double | Test file | Stands in for | Double type | Purpose |
|---|---|---|---|---|
| In-memory SQLite database | all, in local runs | PostgreSQL | Fake | a database with no server; what it cannot reproduce, full-text search and row locking, is tested on PostgreSQL in the CI (see [Test databases](#test-databases)) |
| `chromadb.EphemeralClient` | `test_faq_semantic.py` | Chroma server | Fake | indexing and similarity search with no server to run |
| `fake_embed` | `test_faq_semantic.py` | embedding model | Fake | no model to download, and similarities known in advance |
| `locmem` e-mail backend (`mailoutbox`) | `test_accounts_register.py`, `test_booking_appointments.py`, `test_booking_notifications.py` | SMTP server | Fake used as a spy | e-mails are kept in memory, so the test can check recipients, language and content |
| `StubMatcher` | `test_faq_semantic.py` | full-text and semantic matchers | Stub, also a spy | a fixed answer, and a count of the calls, to test the order of the cascade |
| `find_best_match` that finds nothing | `test_faq_semantic.py` | full-text matching | Stub | sends the question on to semantic search |
| `connection.vendor = "sqlite"` | `test_faq_matching.py` | database engine | Stub | checks that full-text matching refuses to run without PostgreSQL |
| disabled `UniqueValidator` | `test_accounts_register.py` | serializer uniqueness check | Stub | lets an invalid profile reach the database, to check that the user is not registered either |
| `smtp_is_down`, `refuse_the_student` | `test_booking_notifications.py` | `send_mail` | Stub (failure) | a failed e-mail neither undoes the booking nor stops the other e-mail |
| `store_down`, `model_unavailable` | `test_faq_semantic.py` | Chroma server, embedding model | Stub (failure) | the FAQ is saved and matching falls back when the vector store or the model fails |
| `anonymise_then_fail` | `test_faq_import.py` | anonymiser | Stub (failure) | an import that fails halfway leaves the database as it was |
| `no_model` | `test_faq_semantic.py` | embedding model | Dummy | fails the test if called, to check that nothing is embedded |


#### Static analysis tools

Static analysis checks the code without running it. Its tools are also declared in `pyproject.toml`:

- **mypy** checks the type annotations;
- **ruff** checks the code for errors and bad patterns (linting) and keeps its formatting consistent.

`poe static-checks` runs both checks, and `poe format-check` verifies the formatting.

## Testing (automated)

### Unit testing

#### Backend

Unit tests call models, services and commands directly, without going through the HTTP API.

The table reports the run on PostgreSQL, as in the CI. Coverage is measured by running the unit tests alone.

| Module | Tests | What it covers | Requirements | Success rate | Coverage |
|---|---|---|---|---|---|
| `accounts/models.py` | 31 | user model with e-mail login and roles, password hashing, student profile with matricola and degree programme | FR1, FR2, FR3, NFR3 | 31/31 | 98% |
| `accounts/admin.py` | 6 | creation of students and employees from the Django admin, search by matricola | FR3 | 6/6 | 97% |
| `accounts` migrations | 3 | data migration that reactivates the users left inactive by the removed e-mail verification | FR1 | 3/3 | n/a |
| `booking/models.py` | 17 | employee profiles, appointments, deletion rules, at most one booked appointment per employee and slot | FR8, FR9, NFR1 | 17/17 | 98% |
| `booking/services.py` | 61 | shifts declared and withdrawn, available slots, booking with even assignment, cancellation and completion | FR4, FR5, FR8, FR9, FR10, FR11, FR12 | 61/61 | 99% |
| `booking/notifications.py` | 16 | booking and cancellation e-mails, their language and content, failed e-mails that do not undo the booking | FR13, FR14, FR15 | 16/16 | 100% |
| `faq/models.py` | 10 | bilingual FAQs, publication, ordering, deletion rules | FR7 | 10/10 | 95% |
| `faq/matching.py` | 14 | full-text matching: stemming, language, threshold, search across offices | FR6, FR7 | 14/14 (13 on PostgreSQL only) | 99% |
| `faq/vector_index.py`, `faq/signals.py` | 28 | indexing after each commit, rebuild, semantic search, fallback when Chroma is down | FR7 | 28/28 (3 on PostgreSQL only) | 86%, 100% |
| `faq/anonymisation.py` | 58 | removal of e-mail addresses, student IDs, phone numbers, tax codes and names | NFR4 | 58/58 | 98% |
| `faq/management/commands/import_faqs.py` | 37 | import of the FAQ spreadsheet: offices, missing translations, anonymisation, repeated imports | NFR4 | 37/37 | 99% |
| `offices/models.py`, `offices/seed.py` | 8 | office codes and defaults, seeding of the offices | – | 8/8 | 100% |
| `pronto/settings.py` | 3 | installed apps | – | 3/3 | 92% |
| **Total** | **292** | | | **292/292** | **86%** |

### Integration testing

#### Backend

Integration tests go through the HTTP API with the test client of Django REST Framework: each request crosses the routes (URL, authentication, permission check, serializer), the service, the ORM and the database, and the test checks both the response and the state left in the database.

| Components | Tests | What it covers | Requirements | Success rate | Coverage |
|---|---|---|---|---|---|
| Routes → `accounts` → database | 72 | registration with the role given by the e-mail domain, login and logout with a token, the current user, passwords never returned | FR1, FR2, FR3, NFR3 | 72/72 | views 100%, serializers 100% |
| Routes → BookingService → NotificationService | 29 | booking, appointments listed by role, cancellation and completion through the API, e-mails sent after the commit | FR3, FR8, FR11, FR12, FR13, FR14, FR15 | 29/29 | views 100%, serializers 100% |
| Routes → offices and availability | 13 | office list in Italian and English, free slots of an office on a date | FR4, FR8 | 13/13 | views 100% |
| Routes → BookingService (staff) | 20 | an employee chooses their office once, declares and withdraws shifts | FR2, FR4, FR5 | 20/20 | views 100% |
| BookingService → PostgreSQL (concurrency) | 5 | simultaneous bookings, overlapping shifts, a shift withdrawn during a booking, a double cancellation, with real threads | FR9, FR15, NFR1 | 5/5 (PostgreSQL only) | services 93% |
| Routes → FAQ catalogue | 12 | public FAQ list and detail, language, filter by office | FR7 | 12/12 | views 100% |
| Routes → InquiryService → FaqMatchingService → PostgreSQL | 22 | a question is matched, recorded with its office and marked as resolved | FR6, FR7 | 22/22 (7 on PostgreSQL only) | services 100%, matching 82% |
| Health check and PostgreSQL setup | 3 | health endpoint and version, text-search languages available on PostgreSQL | – | 3/3 | – |
| **Total** | **176** | | | **176/176** | **77%** |

Run together, unit and integration tests (468) all pass on PostgreSQL and cover 98% of the backend code.

### System testing

#### Backend

We did not write automated end-to-end tests. The behaviour of the running application and the user experience are verified manually, as described in the next section.

What is verified automatically at system level is the environment in which the application must run, through the CI/CD workflow in `.github/workflows/check.yml`:

- the code is compiled and tested with Python 3.12 and the Poetry version pinned in the `Dockerfile`, the same ones that run the backend in its container;
- the whole suite runs on PostgreSQL 16, in a service container built from the same image as Docker Compose, so the `postgres` tests run too;
- on each push to `develop`, once both test runs pass, the migrations are applied to the shared Neon database with the runtime dependencies alone, and `migrate --check` confirms that none is left unapplied.

Only then does the workflow call `.github/workflows/deploy.yml`, which publishes the release.

Containers are what keep the test environment clean. In the CI, the PostgreSQL server is a service container created for each run and discarded at the end, so nothing is carried over from one run to the next. Locally, the same tests run against the `db` service of Docker Compose (`docker compose up -d --wait db`), on a `test_pronto` database that Django creates and drops at each run, apart from the development data.

## Acceptance tests (manual)

The acceptance criteria that describe what a user sees are checked by hand on the whole system.

| ID | Criterion | Steps | Expected result |
|---|---|---|---|
| MT1 | AC1 | A student registers at `/register` with a `@studio.unibo.it` address, then logs in. | The account is created and the student lands on `/ask`. |
| MT2 | AC6, AC7 | A student asks a question in the textbox. | The answer is shown with its office. |
| MT3 | AC8, AC13 | A student says the answer did not help, picks a free slot and confirms. | The appointment is booked and a confirmation e-mail is sent to the student (this can be checked in the log of `docker compose logs -f backend`). |
| MT4 | AC3 | A student opens `/bookings`. | Only the student's own appointments are listed. |
| MT5 | AC11 | A student cancels a future appointment from `/bookings`. | The appointment is shown as cancelled and the slot is free again. |
| MT6 | AC2 | An employee registers at `/register` with a `@unibo.it` address, logs in and chooses an office at `/profile`. | The account is created and the employee lands on `/queue`. |
| MT7 | AC4 | An employee declares a shift at `/availability`, e.g. Monday 9:00–12:00. | The shift is saved and students see 30-minute slots covering it. |
| MT8 | AC5 | An employee withdraws the shift. | The shift disappears and its slots are no longer offered to students. |
| MT9 | AC3, AC14 | An employee opens `/queue` after a student has booked with them. | The appointment is listed with the student's question, and a notification e-mail was sent to the employee. |
| MT10 | AC12 | An employee marks an appointment that has started as completed in `/queue`. | The appointment is shown as completed. |
| MT11 | AC9, AC16 | Two students, logged in in two browser windows, reach the confirmation of the same slot, which only one employee covers, and confirm at the same moment. | One booking succeeds; the other is rejected because the slot is no longer available. The employee has a single appointment in that slot. |
| MT12 | AC6, AC7 | A student asks two questions about the same FAQ: one with the FAQ's own words, one reworded with different words. An administrator then opens the questions in the Django admin (`/admin/faq/inquiry/`). | Both questions are matched to that FAQ; the column "found with" shows "full-text search" for the first and "semantic similarity" for the second. |

Success rate: 12/12.
