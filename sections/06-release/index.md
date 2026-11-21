---
title: Release
has_children: false
nav_order: 7
---

# Release

## Artefacts and repositories

Each release produces two artefacts per repository, four in total:

| Artefact | Format | Repository | Registry | Install |
|---|---|---|---|---|
| Python package `pronto-helpdesk` | wheel (`.whl`) and sdist (`.tar.gz`) | backend | TestPyPI, then PyPI | `pip install pronto-helpdesk` |
| Backend Docker image | OCI image, `linux/amd64` and `linux/arm64` | backend | GitHub Container Registry (GHCR) | `docker pull ghcr.io/unibo-dtm-se-2526-pronto/pronto-helpdesk:<version>` |
| npm package `pronto-helpdesk-frontend` | tarball (`.tgz`) | frontend | npm | `npm install pronto-helpdesk-frontend` |
| Frontend Docker image | OCI image, `linux/amd64` and `linux/arm64` | frontend | GHCR | `docker pull ghcr.io/unibo-dtm-se-2526-pronto/pronto-helpdesk-frontend:<version>` |

PyPI and npm are the standard registries for Python and JavaScript, so the packages install with plain `pip` and `npm`. Each image is tagged with the version of the release and with `latest`.

**Backend.** The Python package contains the Django project, its four apps and the `pronto-helpdesk` command, and declares its dependencies, which pip installs with it. Each release goes first to TestPyPI, a separate instance of PyPI meant for trials, and reaches PyPI only if that upload succeeds and the repository variable `PUBLISH_TO_PYPI` is `true`, so a missing variable can never publish to the real index by mistake. The wheel and the sdist are also attached to the GitHub release. The image contains the same code, its dependencies and Python 3.12, and applies the migrations when it starts.

**Frontend.** The npm package contains only the built web app, the static files that `vite build` writes to `dist/`, with no source code and no dependencies to install: Vue, Vue Router and Pinia are already inside the bundle. It is meant to be served by any web server that also forwards `/api/`, `/admin/` and `/static/` to the backend, because the build calls the API on its own origin. The image serves the same files with nginx, which also does the forwarding: the backend address comes from the `BACKEND_URL` environment variable, by default the `backend` service of a compose file.

The Docker images ship each component together with its runtime, so it runs the same way on any machine with Docker; `docker-compose.release.yml`, in the backend repository, starts both images together with PostgreSQL and Chroma (see [Deployment](../07-deployment/)). Each repository builds and releases its own image. We chose GHCR because the release job authenticates with the token GitHub Actions creates for each run, so no extra account or secret is needed, and the images are listed next to the repositories in the organisation.

## Choice of the license

Both code repositories, backend and frontend, are released under the Apache License 2.0, a permissive license that lets any institution adopt and adapt Pronto and also covers contributors' patents.

### Code

We chose Apache 2.0 for three reasons. It is permissive: anyone can use, modify and redistribute Pronto, also inside closed-source software, as long as they keep the copyright and license notices. Pronto was built for a university office, and a copyleft license such as the GPL would oblige anyone adapting it for another institution to release their changes under the same terms, which could discourage administrations from adopting it. Unlike MIT and BSD, it also includes an explicit patent grant: every contributor licenses to users any patent of theirs that covers their contribution, with no separate agreement, and the grant is revoked for anyone who sues over those patents. Finally, whoever redistributes a modified version must mark the files they changed, so it stays clear which parts come from the original project.

### Artefacts

The artefacts built from the codebase use the same license as the code. Apache 2.0 distinguishes the *Source form* (the code in the repository) from the *Object form* (anything obtained by building or packaging it) and grants the same rights on both. Third-party code inside an artefact keeps its own license, and Apache 2.0 covers only our part:

- **Python package**: only our own code. The libraries it needs are declared as dependencies and installed separately, each under its own license.
- **Backend image**: also the code of the operating system (Debian, from the `python:3.12-slim` base image), of Python and of the installed libraries, each with its own license file.
- **npm package and frontend image**: the bundle contains the code of Vue, Vue Router and Pinia. Vite would strip their license comments, so the build is configured (`build.license` in `vite.config.ts`) to write them to `dist/.vite/license.md`, which ships in both artefacts. The image also contains Alpine Linux and nginx.

Those licenses are compatible with Apache 2.0. In the backend, Django, Django REST framework, python-dotenv, django-cors-headers and openpyxl are under BSD or MIT, chromadb and fastembed under Apache 2.0, and psycopg under the LGPL 3.0, which allows software under any license to use it as an unmodified dependency. In the frontend, Vue, Vue Router and Pinia are under MIT.

## Choice of the versioning schema

Both repositories follow Semantic Versioning (SemVer). We chose it because the version number tells users what kind of change a release contains and whether they can upgrade safely.

Each of the three numbers covers one kind of change:

- `MAJOR` increases when a release breaks backwards compatibility, so code written for the previous version may stop working.
- `MINOR` increases when a release adds functionality without breaking anything, such as a new endpoint.
- `PATCH` increases for backwards-compatible bug fixes, which change the behaviour only where it was wrong.

Version `1.0.0` was the first release with a stable API. The rules that map our commits to each kind of increment are described below.

Artefacts built from the same repository share the version number: the Python package and the backend image of a release have the same version, and so do the npm package and the frontend image. The two repositories instead keep their own pace, because semantic-release runs separately in each and a change to one component leaves the other untouched. Tags carry the bare version (`1.2.0`, not `v1.2.0`).

## Creating a new release

As described in [CI/CD](../08-cicd/), the release job updates the version number and the changelog, publishes the new version, and creates its Git tag and its GitHub release, so nobody creates tags or releases by hand. In both repositories it runs on a push to the release branch, in practice every merge of `develop`, once the tests have passed. The two repositories differ in what happens elsewhere:

- **Backend**: the release job runs after the tests on every push and every pull request. Only on `master` is the release real; on any other branch it is a dry run, in which semantic-release loads its configuration and plugins and stops, since it publishes only from `master`, so a broken release configuration shows up before it reaches `master`.
- **Frontend**: the release job is called only by a push to `main`; on other branches the workflow stops after the tests. Started by hand on another branch, it performs a dry run.

The version is written back to the repository by the release commit, `chore(release): <version> [skip ci]`: to `pyproject.toml` in the backend and to `package.json` in the frontend, together with `CHANGELOG.md`. The frontend also compiles the version into the app, which shows it in the user menu.

A push does not always produce a release. semantic-release reads the commits added since the last tag and, because they follow Conventional Commits, derives the increment from their types. When the commits call for different levels, the highest one applies:

- a breaking commit, marked with `!` after the type (`feat!: ...`) or with a `BREAKING CHANGE:` footer, increments `MAJOR`;
- a `feat` commit increments `MINOR`;
- a `fix`, `perf`, `docs` or `revert` commit increments `PATCH`.

Other commits, such as `test`, `ci`, `refactor` or `chore`, do not trigger a release on their own and ship with the next one. Dependency updates follow the same rules through the type their bot gives them:

- **Backend**: Dependabot writes `chore(deps)`, `chore(deps-dev)` and, for the GitHub Actions, `ci(deps)`, so an update never makes a release by itself.
- **Frontend**: Renovate writes `fix(deps)` for the libraries that end up in the bundle (Vue, Vue Router, Pinia, a rule in `renovate.json`), which makes a `PATCH` release because the published app changes, `chore(deps)` for the development tools and `ci(deps)` for the GitHub Actions.

