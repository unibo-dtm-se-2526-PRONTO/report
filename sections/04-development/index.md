---
title: Development
has_children: false
nav_order: 5
---

# Development

## DVCS

Pronto lives in three Git repositories hosted under the `unibo-dtm-se-2526-PRONTO` GitHub organisation: the backend, the frontend and this report. All three follow the same conventions.

**Branches.** We adopted a Git Flow model:

- `master` (`main` in the frontend and in the report) holds released code only, and every merge into it produces a release;
- `develop` is the integration branch, where finished work is collected and kept working;
- work branches start from `develop` and are named after the kind of change: `feature/<topic>` for new functionality (e.g. `feature/booking`, `feature/faq-matching`, `feature/shifts`), `fix/<topic>` for bugs, `refactor/<topic>`, `test/<topic>` and `chore/<topic>` for the other kinds of work;
- `hotfix/<topic>` branches carry small, urgent corrections (e.g. `hotfix/typecheck-faq`).

Each branch covers a single topic, so the branch list mirrors the features of the system.

**Commit messages.** Every commit follows [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): description`, where the type is one of `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `ci`, `build` and `style`, and the scope names the Django app or the frontend module involved (e.g. `fix(booking): lock the appointment when cancelling it`). The description is in the imperative mood and says what changes for the user or the developer. We enforce the convention with commitlint, which a GitHub Actions workflow runs on every pull request. The convention is not only stylistic: semantic-release reads the commit types to choose the next version number and to write the changelog (see the Release chapter).

**Pull requests and code reviews.** No one pushes directly to `develop` or to `master`: every change reaches them through a pull request. A pull request is merged only when the CI checks pass (static checks, formatting, test suite, commit messages) and the other team member has reviewed it. We merge with a merge commit, so the history keeps track of which commits came in together. Releases follow the same path: a pull request from `develop` to `master`, which commitlint skips because it only carries commits already checked. Before the first release we also ran a whole-codebase review on a dedicated `refactor/code-review` branch, which produced a series of small `fix` and `refactor` commits.

## Implementation details

**Network protocols.** Clients talk to the backend over **HTTP** (HTTPS in production), following the REST style. Every interaction in Pronto is a synchronous request/response exchange (see the Design chapter), which is exactly what HTTP offers; nothing has to be pushed to the client in real time, so WebSockets or a message broker would only add complexity. HTTP also works from any browser without plugins, and Django and Django REST Framework (DRF) support it natively. The backend reaches PostgreSQL through its own wire protocol over TCP, Chroma through its HTTP API, and the mail server through **SMTP**.

**Data representation.** Data in transit is encoded as **JSON**. It is the native format of both ends: DRF serializers turn models into JSON and validate incoming JSON, and the frontend reads it with `fetch` with no extra library. The text is readable, which made debugging the API easy. The language of stored texts (office names, FAQs) is chosen with the `?lang=it|en` query parameter rather than with separate endpoints. Dates and times travel as ISO 8601 strings. The identifier of an inquiry is a UUID, so that it cannot be guessed (see Design).

**Database queries.** The application data is relational and strongly linked (appointments reference students, employees, offices and FAQs), so we store it in **PostgreSQL** and query it through the **Django ORM**, without hand-written SQL. The ORM keeps queries in Python, protects against SQL injection and generates the schema migrations. Two features of PostgreSQL are used directly through the ORM:

- **full-text search** (`SearchVector`, `SearchQuery`, `SearchRank`) with the `italian` and `english` text-search configurations, which is the first step of FAQ matching; the question is weighted more than the answer;
- **constraints and row locks**: a partial unique constraint on (employee, start) among booked appointments makes concurrent bookings safe, and `select_for_update` locks the rows involved when an appointment is booked or cancelled.

The second step of FAQ matching uses **Chroma**, a vector database queried through its Python client. Embeddings are computed inside the backend with the multilingual model `paraphrase-multilingual-MiniLM-L12-v2`, run by fastembed. Chroma holds only derived data: a Django signal updates it after each committed change to a FAQ, and the `rebuild_faq_index` command rebuilds it from PostgreSQL.

**Authentication.** Clients authenticate with **DRF token authentication**. At login the backend issues an opaque token, stored in the database; the frontend keeps it in `localStorage` and sends it in the `Authorization: Token <key>` header of each request; logout deletes it on the server. We preferred it to JWT because Pronto has one server and one client: a token stored in the database can be revoked at any time, and we do not need tokens that can be verified without the database. Passwords are hashed with Django's default algorithm (PBKDF2). Registration is open only to university addresses: `@studio.unibo.it` addresses register as students, `@unibo.it` addresses as employees, all others are refused.

**Authorization.** Authorization is **role-based (RBAC)**. Each user has one role: student, employee or administrator. Students and employees get theirs from the e-mail domain at registration; administrators are created by the staff. Two DRF permission classes, `IsStudent` and `IsEmployee`, guard the endpoints, on top of the default `IsAuthenticated`: only students ask questions and book, only employees declare shifts and complete appointments. Roles alone are not enough for appointments, so the querysets are also scoped to the caller: a student only sees their own appointments, an employee only those assigned to them. Administrators work in the Django admin, which is protected by Django's staff flag. The frontend repeats the role checks in the router (`meta.role`) to hide the pages a user cannot use, but the backend remains the only real check.

## Technological details

**Languages and frameworks.**

- **Backend: Python 3.12, Django 5.2 and Django REST Framework.** Django gave us the ORM, migrations, authentication and, above all, the admin site, which the helpdesk staff use to review and publish FAQs without a dedicated interface. DRF adds serializers, token authentication and permission classes on top of it. Python also gave us easy access to the embedding and spreadsheet libraries we needed.
- **Frontend: TypeScript, Vue 3 and Vite.** Vue's single-file components are simple to learn and suited to an application of this size; TypeScript catches errors in the API payloads at build time, since the types in `src/api/types.ts` mirror the backend serializers. Vite gives fast reloads in development and builds the static bundle.
- **Database: PostgreSQL 16**, for its full-text search and its constraints; **Chroma 1.5** for semantic search.

**Libraries.** The backend depends on:

- `djangorestframework`, for the REST API;
- `django-cors-headers`, so that the frontend, served from another origin, can call the API;
- `psycopg`, the PostgreSQL driver;
- `chromadb` and `fastembed`, for semantic matching; fastembed runs the model with ONNX, without installing PyTorch;
- `openpyxl`, to import the FAQs from the helpdesk spreadsheet (`import_faqs`), which also anonymises them before storing them unpublished;
- `python-dotenv`, to read configuration and secrets from a `.env` file that is not versioned.

The frontend depends only on `vue`, `vue-router` (navigation and role guard) and `pinia` (stores for authentication, offices and bookings). We wrote the HTTP client on `fetch` and a small Italian/English translation module ourselves instead of adding axios and vue-i18n, because two languages and a dozen endpoints did not justify them.

**Development tools.**

| Purpose | Backend | Frontend |
|---|---|---|
| Dependencies | Poetry | npm |
| Tests | pytest, pytest-django, coverage | Vitest, Vue Test Utils, jsdom |
| Linting and formatting | Ruff | ESLint, oxlint, Prettier |
| Type checking | mypy | vue-tsc |
| Task runner | Poe the Poet | npm scripts |

Docker Compose starts the whole system locally (PostgreSQL, Chroma, backend and frontend), with the same database engine used in production. GitHub Actions runs the CI, semantic-release publishes the releases, and Renovate keeps dependencies up to date with automatic pull requests.

**External services.**

- **Neon**, a managed PostgreSQL service hosting the shared development database; the CI applies the migrations to it after each merge into `develop`;
- an **SMTP server**, which delivers the booking and cancellation e-mails; in development the backend prints them to the console instead;
- **GitHub** (repositories, Actions, Releases) and **PyPI/TestPyPI**, where the backend is published as the `pronto-helpdesk` package.

We use no external API at run time other than the SMTP server, so the system keeps working, with full-text search only, even when Chroma is unavailable.
