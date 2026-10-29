---
title: Release
has_children: false
nav_order: 7
---

# Release

- Which and how many artefacts are produced from your project's codebase?
- Onto which repositories (e.g. PyPI, Docker Hub, GitHub Packages, NPM etc.) are they released? Why?
- How are they released (e.g. manually, automatically, etc.)?
   + report the configuration steps and commands to run to release the artefacts

## Choice of the license

Both code repositories, backend and frontend, are released under the Apache License 2.0.

### Artefacts

The artefacts built from the codebase use the same license as the code. Apache 2.0 distinguishes the *Source form* (the code in the repository) from the *Object form* (anything obtained by building or packaging it) and grants the same rights on both. Two licenses would be needed if the built packages contained something the source does not, such as bundled third-party code, a trademark or a commercial edition. Pronto has none of these: the published package contains only our own code, and the libraries it needs are declared as dependencies and installed separately, each under its own license.

Those licenses are compatible with Apache 2.0. Django, Django REST framework, python-dotenv, django-cors-headers, openpyxl, Vue, Vue Router and Pinia are under BSD or MIT, and chromadb and fastembed under Apache 2.0. psycopg is under the LGPL 3.0, which allows software under any license to use it as an unmodified dependency.

### Code

We chose Apache 2.0 for three reasons. It is permissive: anyone can use, modify and redistribute Pronto, also inside closed-source software, as long as they keep the copyright and license notices. Pronto was built for a university office, and a copyleft license such as the GPL would oblige anyone adapting it for another institution to release their changes under the same terms, which could discourage administrations from adopting it. Unlike MIT and BSD, it also includes an explicit patent grant: every contributor licenses to users any patent of theirs that covers their contribution, with no separate agreement, and the grant is revoked for anyone who sues over those patents. Finally, whoever redistributes a modified version must mark the files they changed, so it stays clear which parts come from the original project.

## Choice of the versioning schema

Both repositories follow Semantic Versioning (SemVer). We chose it because the version number tells users what kind of change a release contains and whether they can upgrade safely. 

Each of the three numbers covers one kind of change:

- `MAJOR` increases when a release breaks backwards compatibility, so code written for the previous version may stop working. 
- `MINOR` increases when a release adds functionality without breaking anything, such as a new endpoint.
- `PATCH` increases for backwards-compatible bug fixes, which change the behaviour only where it was wrong.

Version `1.0.0` was the first release with a stable API. The rules that map our commits to each kind of increment are described below.

- In case of multiple artefacts, are the version numbers aligned or each artefact has its own versioning pace? Why?

## Creating a new release

As described in the CI/CD section, the release job updates the version number and the changelog, publishes the new version, and creates its Git tag and its GitHub release, so nobody creates tags or releases by hand. The job runs on every push to the release branch (`master` in the backend, `main` in the frontend), in practice every merge of `develop`, once the tests have passed. On any other branch it only performs a dry run.

A push does not always produce a release. semantic-release reads the commits added since the last tag and, because they follow Conventional Commits, derives the increment from their types. When the commits call for different levels, the highest one applies:

- a breaking commit, marked with `!` after the type (`feat!: ...`) or with a `BREAKING CHANGE:` footer, increments `MAJOR`;
- a `feat` commit increments `MINOR`;
- a `fix`, `perf`, `docs` or `revert` commit increments `PATCH`.

Dependency updates use the scope to choose the level: `chore(api-deps)` increments `MINOR` and `chore(core-deps)` increments `PATCH`. Other commits, such as `test`, `ci` or a plain `chore`, do not trigger a release on their own and ship with the next one.

