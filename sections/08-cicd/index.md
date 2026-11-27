---
title: CI/CD
has_children: false
nav_order: 9
---

# CI/CD

Both code repositories, backend and frontend, are checked, tested and released by GitHub Actions workflows stored in `.github/workflows/`.

## Backend

The backend has four workflows: `check.yml`, `commitlint.yml`, `deploy.yml` and `dependabot-automerge.yml`.

### Automated actions

| Action | Workflow | Job | Triggers | Description | Reason |
|---|---|---|---|---|---|
| 1 | `check.yml` | `check` | Push to any branch except `dependabot/**` (pushes that only change files such as `README.md`, `CHANGELOG.md`, `LICENSE` or `.gitignore` are ignored), pull request, manual run | Checks that the code compiles, runs ruff, mypy and the format check, then runs the test suite on in-memory SQLite. | Finds syntax, type and style errors quickly, without a database server, and stops the pipeline before the slower jobs start. |
| 2 | `check.yml` | `test` | Same as action 1, once action 1 succeeds | Runs the test suite on a PostgreSQL 16 service container, measures coverage and uploads the HTML report as a build artifact. | The FAQ matching and concurrent booking tests need PostgreSQL and are skipped on SQLite, so only this run executes them. |
| 3 | `check.yml` | `migrate-neon` | Push to `develop` or manual run on `develop`, once action 2 succeeds | Prints the migration plan, applies the migrations to the shared Neon database and checks that none is left unapplied. | Keeps the shared Neon schema aligned with `develop` without manual `migrate` runs, and only with code that passed every test. |
| 4 | `deploy.yml` | `release` | Once action 2 succeeds, or manual run. Real release only on `master`; a dry run on other branches and on pull requests | Runs semantic-release, which reads the commits, computes the next version, writes it to `pyproject.toml` and updates `CHANGELOG.md`, commits and tags the release, publishes the package to TestPyPI and then to PyPI, builds the Docker image for `linux/amd64` and `linux/arm64` and pushes it to GHCR, and creates a GitHub release with the built files. | Versions and changelog follow from the commit types, with no manual steps; if TestPyPI fails, nothing reaches PyPI. Package and image come from the same commit and carry the same version. |
| 5 | `commitlint.yml` | `commitlint` | Pull request, except the release pull request from `develop` | Checks that each commit message follows Conventional Commits. | semantic-release derives the version from the commit types, so a wrong message would silently skip a release. |
| 6 | `dependabot-automerge.yml` | `automerge` | Pull request opened by Dependabot | For patch and minor updates, waits for the CI/CD run of the pull request, merges it into `develop` if it passes, then starts `check.yml` on `develop`. | Patch and minor updates should not break compatibility, so passing checks are enough; major updates still need a person. A merge made by a workflow starts no other workflow, so the run on `develop` has to be started explicitly. |

### GitHub Actions implementation

The workflows contain no secrets. GitHub stores them encrypted and passes them to the jobs as environment variables. The tokens for the two package indexes, `TEST_PYPI_TOKEN` and `PYPI_TOKEN`, are repository secrets that Poetry reads from `POETRY_PYPI_TOKEN_TESTPYPI` and `POETRY_PYPI_TOKEN_PYPI`, which keeps them out of commands and logs. The repository variable `PUBLISH_TO_PYPI` decides whether a release also goes to PyPI. Tags, GitHub releases and the image use the `GITHUB_TOKEN` that Actions creates for each run, so there is no personal token to renew. `deploy.yml` grants it the `packages: write` permission, and the job logs in to GHCR with it only on `master`, where the image is pushed.

### Dependency automation

Dependabot, a service built into GitHub and configured in `.github/dependabot.yml`, keeps the dependencies up to date. Every Monday it checks the Python packages managed by Poetry, the npm packages used by semantic-release, the GitHub Actions used in the workflows, and the Docker images of the Dockerfile and of `docker-compose.yml`, for which it proposes only patch versions. For each new version it opens a pull request against `develop`. Updates therefore follow the same path as any other change and reach `master` with the next release. The exception are security updates, which GitHub always opens against the default branch, `master`.

On these pull requests, the `dependabot-automerge.yml` workflow merges patch and minor updates once the CI/CD run of the pull request has passed. After the merge, the workflow therefore starts `check.yml` on `develop` as a manual run. Major updates may break compatibility, so they wait for a manual review.

## Frontend

The frontend has three workflows: `check.yml`, `deploy.yml` and `commitlint.yml`.

### Automated actions

| Action | Workflow | Job | Triggers | Description | Reason |
|---|---|---|---|---|---|
| 1 | `check.yml` | `check` | Push to any branch except `renovate/**`, pull request, manual run | Runs oxlint and ESLint, the type check with `vue-tsc` and the whole test suite with coverage, which fails below the thresholds set in `vitest.config.ts` (95% of lines, statements and functions, 90% of branches), and uploads the coverage report as a build artifact. | Catches style, type and logic errors in a single fast run, and keeps untested code from creeping in. |
| 2 | `check.yml` | `test` | Same as action 1, once action 1 succeeds | Builds the app and runs the tests on Linux, Windows and macOS with Node 20, 22 and 24, nine combinations in all. | The build and the date handling must not depend on a single operating system or Node version. |
| 3 | `deploy.yml` | `release` | Push to `main`, once action 2 succeeds, or manual run (a dry run on other branches) | Runs semantic-release, which computes the next version, writes it to `package.json` and updates `CHANGELOG.md`, builds the app in `dist/` and publishes it to npm as `pronto-helpdesk-frontend`, builds the image for `linux/amd64` and `linux/arm64` and pushes it to GHCR, then commits the version and creates the tag and the GitHub release. | As in the backend: version, changelog, package and image follow from the commits, with no manual steps. The app is built after the version is written, so the version it shows is the released one. |
| 4 | `commitlint.yml` | `commitlint` | Pull request, except the release pull request from `develop` | Checks that each commit message follows Conventional Commits. | As in the backend. |

### GitHub Actions implementation

The only secret is `NPM_TOKEN`, a granular access token from npmjs.com with read and write access to packages, which the npm plugin of semantic-release reads. GHCR, the tag, the release commit and the GitHub release use the `GITHUB_TOKEN` of the run, as in the backend. The release job builds the arm64 image on the amd64 runner through QEMU emulation, and runs one release at a time (`concurrency` group `deploy`).

### Dependency automation

Renovate, an external GitHub app configured in `renovate.json`, keeps the dependencies up to date. It opens a pull request against `develop` for each update, keeping major, minor and patch updates separate, and merges it automatically once the checks pass. Its Dependency Dashboard, an issue in the repository, lists the pending updates. Unlike Dependabot in the backend, Renovate merges major updates as well: the frontend test suite covers every view and the whole booking flow, so a breaking update fails the checks and waits for a person.