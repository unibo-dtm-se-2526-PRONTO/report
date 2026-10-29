---
title: CI/CD
has_children: false
nav_order: 9
---

# CI/CD

Both code repositories, backend and frontend, are checked, tested and released by GitHub Actions workflows stored in `.github/workflows/`. 

### BACKEND 

Actions are split into four workflows: `check.yml`, `commitlint.yml` , `deploy.yml`, `dependabot-automerge.yml` .

## Automated Actions

| Action | Workflow | Job | Triggers | Description | Reason |
|---|---|---|---|---|---|
| 1 | `check.yml` | `check` | Push to any branch except `dependabot/**` (pushes that only change `README.md`, `CHANGELOG.md` or `LICENSE` are ignored), pull request, manual run | Checks that the code compiles, runs ruff, mypy and the format check, then runs the test suite on in-memory SQLite. | Finds syntax, type and style errors quickly, without a database server, and stops the pipeline before the slower jobs start. |
| 2 | `check.yml` | `test` | Same as action 1, once action 1 succeeds | Runs the test suite on a PostgreSQL 16 service container, measures coverage and uploads the HTML report as a build artifact. | The FAQ matching and concurrent booking tests need PostgreSQL and are skipped on SQLite, so only this run executes them. |
| 3 | `check.yml` | `migrate-neon` | Push to `develop`, once action 2 succeeds | Prints the migration plan, applies the migrations to the shared Neon database and checks that none is left unapplied. | Keeps the shared Neon schema aligned with `develop` without manual `migrate` runs, and only with code that passed every test. |
| 4 | `deploy.yml`| `release` | Once action 2 succeeds, or manual run. Real release only on `master`; a dry run on other branches and on pull requests | Runs semantic-release, which reads the commits, computes the next version, writes it to `pyproject.toml` and updates `CHANGELOG.md`, commits and tags the release, publishes the package to TestPyPI and then to PyPI, and creates a GitHub release with the built files. | Versions and changelog follow from the commit types, with no manual steps; if TestPyPI fails, nothing reaches PyPI. |
| 5 | `commitlint.yml` | `commitlint` | Pull request, except the release pull request from `develop` | Checks that each commit message follows Conventional Commits. | semantic-release derives the version from the commit types, so a wrong message would silently skip a release. |
| 6 | `dependabot-automerge.yml` | `automerge` | Pull request opened by Dependabot | Enables auto-merge for patch and minor updates, so GitHub merges them into `develop` once the required checks pass. | Patch and minor updates should not break compatibility, so passing checks are enough; major updates still need a person. |

## GitHub Actions implementation

The workflows contain no secrets. GitHub stores them encrypted and passes them to the jobs as environment variables. The tokens for the two package indexes, `TEST_PYPI_TOKEN` and `PYPI_TOKEN`, are repository secrets that Poetry reads from `POETRY_PYPI_TOKEN_TESTPYPI` and `POETRY_PYPI_TOKEN_PYPI`, which keeps them out of commands and logs. The repository variable `PUBLISH_TO_PYPI` decides whether a release also goes to PyPI. Tags and GitHub releases use the `GITHUB_TOKEN` that Actions creates for each run, so there is no personal token to renew.

## Dependency automation

Dependabot, a service built into GitHub and configured in `.github/dependabot.yml`, keeps the dependencies up to date. It checks weekly the Python packages managed by Poetry, the npm packages used by semantic-release, the GitHub Actions used in the workflows, and the Docker images of the Dockerfile and of `docker-compose.yml`. For each new version it opens a pull request against `develop`. Updates therefore follow the same path as any other change and reach `master` with the next release.

On these pull requests, the `dependabot-automerge.yml` workflow enables auto-merge for patch and minor updates, and GitHub merges them once the required checks on `develop` pass. Major updates may break compatibility, so they wait for a manual review.

### Frontend

The frontend repository has three workflows in `.github/workflows/`. `check.yml` runs on every pull request and on every push, except to the branches Renovate opens. Its first job, `check`, runs the linters, the type check and the whole test suite with coverage, which fails below the thresholds set in `vitest.config.ts`, and uploads the coverage report. Once it passes, the `test` job builds the app and runs the tests on Linux, Windows and macOS with Node 20, 22 and 24, so the build does not depend on a single environment. On a push to `main`, a passing `test` job calls `deploy.yml`, where semantic-release computes the next version from the commits, writes it to `package.json`, updates `CHANGELOG.md`, and creates the tag and the GitHub release. The frontend is not published to npm (`npmPublish: false`), and the release uses only the `GITHUB_TOKEN` of the run, so the workflows need no secrets. `commitlint.yml` checks on every pull request, except the release pull request from `develop`, that commit messages follow Conventional Commits. Renovate, configured in `renovate.json`, keeps the dependencies up to date: it opens a pull request against `develop` for each update, with major, minor and patch updates kept separate, and merges it automatically once the checks pass.
