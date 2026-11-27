---
title: Developer guide
has_children: false
nav_order: 11
---

# Developer Guide
# Developer Guide

This section is for a developer who joins the team: how the work is organised, how to set up the backend and the frontend, and the conventions the code follows.

## Organisation

The team is made of two developers, [Sara Ladisa](mailto:sara.ladisa@studio.unibo.it) and [Giulio Salotti](mailto:giulio.salotti@studio.unibo.it). The project lives in three repositories of the `unibo-dtm-se-2526-PRONTO` GitHub organisation:

| Repository | Content | Stable branch |
|---|---|---|
| [`artifact-backend`](https://github.com/unibo-dtm-se-2526-PRONTO/artifact-backend) | Django REST API, admin site, `docker-compose.yml` for development and `docker-compose.release.yml` for users | `master` |
| [`artifact-frontend`](https://github.com/unibo-dtm-se-2526-PRONTO/artifact-frontend) | Vue web app | `main` |
| `report` | this report, a Jekyll site | `main` |

Bugs and requests are reported as GitHub issues in the repository they concern; a problem whose cause is unclear goes in the backend repository. To join, a new developer asks one of the team to be added to the organisation, which gives write access to the three repositories.

## Workflow

Every change, in every repository, follows the same path:

1. Start from an up-to-date `develop`: `git checkout develop && git pull`.
2. Create a branch named after the kind of change, following the GitFlow: `feature/`, `fix/`, `refactor/`, `test/`, `docs/`, `chore/` or `ci/`, followed by a short topic, such as `feature/booking-reminder`. A branch covers a single topic.
3. Commit with [Conventional Commits] messages, `type(scope): description`, in English and in the imperative mood, with the Django app or the frontend module as scope:

        feat(booking): send a reminder the day before the appointment
        fix(auth): reach the login page when the logout request fails
        test(faq): cover the fallback when Chroma is unreachable

   The type decides the next version: `feat` for new behaviour, `fix` for a bug, `docs` for documentation; a breaking change takes a `!` after the type. Only `feat`, `fix`, `perf`, `docs`, `revert` and breaking commits make a release.
4. Push the branch and open a pull request against `develop`. The CI runs the checks and commitlint checks the commit messages; in the backend GitHub refuses to merge until they pass.
5. Ask the other developer for a review, and merge once it is approved and the checks are green.

To release, open a pull request from `develop` to the stable branch. Once it is merged, the release job publishes the new version on its own: nobody bumps versions, edits `CHANGELOG.md` or creates tags by hand. The release commit, with the new version and changelog, lands only on the stable branch: after each release, open a pull request from the stable branch back to `develop`, so that the two do not drift apart.

Dependency updates arrive as pull requests against `develop` too, from Dependabot in the backend and from Renovate in the frontend, and are merged automatically when the checks pass. A Dependabot pull request for a major version needs a person: read the changelog of the library, run the tests and merge it by hand.

## Backend

### Setting up

The backend needs Docker with Docker Compose, Python 3.12 and [Poetry](https://python-poetry.org/) 2.2.1, the version pinned in the `Dockerfile` that the CI also reads. Clone the backend and the frontend side by side, because the development compose file builds the frontend from `../artifact-frontend`:

    git clone https://github.com/unibo-dtm-se-2526-PRONTO/artifact-backend.git
    git clone https://github.com/unibo-dtm-se-2526-PRONTO/artifact-frontend.git
    cd artifact-backend
    cp .env.example .env        # set SECRET_KEY; DEBUG=True is already there
    docker compose up --build

`docker compose` starts PostgreSQL, Chroma, the backend on <http://localhost:8000> and the frontend on <http://localhost:5173>; the backend applies the migrations once PostgreSQL and Chroma are healthy. The source folder is mounted into the container, so the development server reloads at each change. On a new database, load the data and create an administrator from a second terminal:

    docker compose exec backend python manage.py seed_offices
    docker compose exec backend python manage.py seed_faqs
    docker compose exec backend python manage.py createsuperuser

E-mails are printed in the backend log (`docker compose logs -f backend`) instead of being sent. `.env` is never committed: it holds the secrets, and `.env.example` documents every key.

For the tests and the editor, install the environment on the host as well with `poetry install`. Poetry creates it in `.venv/` inside the repository (`poetry.toml`), where the editor finds it.

### Tests and checks

The tasks are defined in `pyproject.toml` and run with [Poe the Poet](https://poethepoet.natn.io/):

| Command | What it does |
|---|---|
| `poetry run poe test` | pytest on an in-memory SQLite database |
| `poetry run poe coverage` and `poe coverage-report` | the same, measuring coverage |
| `poetry run poe static-checks` | Ruff and mypy |
| `poetry run poe format` / `poe format-check` | formats the code with Ruff, or only checks it |
| `poetry run poe compile` | checks that every file compiles |

The CI runs `compile`, `static-checks`, `format-check` and `test` in the first job, so running them before pushing avoids a red pull request. Tests that need PostgreSQL (full-text search and concurrent bookings) are marked `@pytest.mark.postgres` and skipped on SQLite. To run them on the `db` container of compose:

    docker compose up -d --wait db
    TEST_DATABASE_URL=postgres://pronto:pronto@localhost:5432/pronto?sslmode=disable \
        poetry run poe test

### Conventions

- The project `pronto/` holds settings, URLs, permissions, shared enums and the `pronto-helpdesk` command; each of the four apps (`accounts`, `offices`, `booking`, `faq`) holds its models, admin and, when it exposes endpoints, its serializers, views and URLs. Business logic goes in service modules, as functions (`booking/services.py`, `faq/services.py`, and `faq/matching.py` for the matchers); views only check the request, call a service and serialise the result. A new feature that does not fit an existing app gets a new app rather than a module in another one.
- Ruff formats and lints the code, and mypy checks the types. Docstrings and comments are in English.
- Every change to a model comes with its migration, created with `python manage.py makemigrations` and committed with the change. After a merge into `develop`, the CI applies the migrations to the shared Neon database, so a migration must not depend on data that exists only locally.
- Tests live in `tests/`, one file per app and topic, named `test_<app>_<topic>.py` (`test_booking_shifts.py`). Shared fixtures, such as users of each role and an office with its employees, are in `tests/conftest.py`. Each new endpoint is tested for its permissions as well: the wrong role, and someone else's object, which must answer `404`.
- New settings are read from environment variables in `pronto/settings.py`, with a safe default, and added to `.env.example` and to the configuration table of the README.

## Frontend

### Setting up

The frontend needs Node.js 22.18 or later in the 22 line, or 24.12 or later, and the backend running on <http://localhost:8000>, for instance with the backend's `docker compose`, which already starts the frontend on <http://localhost:5173>. To run it on the host instead:

    cd artifact-frontend
    npm ci
    npm run dev                 # http://localhost:5173, with hot reload

The backend accepts cross-origin requests only from <http://localhost:5173> (CORS), so the development server must keep that port. `.env` points the app at the local backend; to change a value on one machine only, create a `.env.local`, which is not committed. `VITE_*` values end up in the JavaScript the browser downloads, so they never hold secrets.

### Tests and checks

| Command | What it does |
|---|---|
| `npm run test:unit` | Vitest in watch mode; `-- --run` runs the suite once |
| `npm run test:coverage` | the suite once, with coverage; fails below 95% of lines, statements and functions or 90% of branches |
| `npm run lint` | oxlint and ESLint, fixing what they can |
| `npm run type-check` | vue-tsc on the whole project |
| `npm run format` | Prettier on `src/` |
| `npm run build` | type check and production build in `dist/` |

The CI runs `lint`, `type-check` and `test:coverage` first, then the build and the tests on three operating systems and three Node versions.

The tests live in `src/__tests__/`, one file per view, store or module, plus `router.spec.ts` for the navigation guard and `flows.spec.ts` for whole journeys through the real router. They follow a few rules, described in detail in the repository's README:

- Tests make no network calls: `fetch` is replaced by a mock before each test, and the helper `mockApi()` answers the requests a test expects; a request nobody mocked fails the test.
- Tests check behaviour through the DOM. They fill fields by their label, click buttons by their text and read errors from `role="alert"`; they assert on what the user sees, the requests sent and the route reached, not on component internals.
- `vitest.config.ts` fixes the time zone to `Europe/Rome`, the office's time zone, so dates and slot times are the same on every machine; instants in the tests are written with their offset.
- Known bugs are written as `it.fails`, with a comment saying what is wrong, so the test signals the day the bug is fixed.

### Conventions

- Pages are in `src/views/`, grouped by role (`auth/`, `student/`, `employee/`); a new page is also added to `src/router/index.ts`, with `meta.role` when only one role may open it. Shared state goes in a Pinia store in `src/stores/`.
- Views never build URLs or call `fetch`: every endpoint is a function in `src/api/endpoints.ts`, which goes through `apiFetch` in `src/api/client.ts` for the token and the errors. The types in `src/api/types.ts` mirror the backend serializers, so a change to a serializer is reflected there in the same pull request.
- Every text of the interface is in `src/i18n.ts`, in Italian and English, and is read with `t('key')`. The English dictionary has the type of the Italian one, so a missing translation fails the type check.
- Prettier formats the code (no semicolons, single quotes, lines up to 100 characters), and `.editorconfig` sets the indentation for the editors. Imports from `src/` use the `@/` alias.




