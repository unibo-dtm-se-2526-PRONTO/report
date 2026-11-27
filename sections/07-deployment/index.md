---
title: Deployment
has_children: false
nav_order: 8
---

# Deployment

To install Pronto, run the following commands in your terminal. You can either install it with Docker, download it from PyPI or install it from source.

## User installation

With Docker, the installation takes five steps:

1. Download the compose file into an empty folder:

        mkdir pronto && cd pronto
        curl -O https://raw.githubusercontent.com/unibo-dtm-se-2526-PRONTO/artifact-backend/master/docker-compose.release.yml

2. Start the stack:

        docker compose -f docker-compose.release.yml up -d

   The first start downloads the images and the embedding model for the semantic search, so it needs an Internet connection and takes a few minutes. The backend then applies the migrations and loads the offices and the FAQs.

3. Create the administrator account, which asks for an e-mail address, a name and a password:

        docker compose -f docker-compose.release.yml exec backend python manage.py createsuperuser

4. Open <http://localhost:8080>. The administrator manages FAQs, offices and users at <http://localhost:8080/admin/>. Students register with an `@studio.unibo.it` address and employees with an `@unibo.it` address; an employee chooses an office and declares the weekly shifts before students can book.

5. Stop the stack with `docker compose -f docker-compose.release.yml down`, adding `-v` to delete its data too.

### From PyPI

The backend can also be installed without Docker from PyPI, with `pip install pronto-helpdesk`. The package contains the API and the admin site, but not the web app. The package provides the `pronto-helpdesk` command:

    pronto-helpdesk migrate
    pronto-helpdesk seed_offices
    pronto-helpdesk seed_faqs
    pronto-helpdesk createsuperuser
    pronto-helpdesk runserver

The web app is available separately from npm, as the static files in `dist/` of `pronto-helpdesk-frontend`. Any web server can serve them.

### From source

Both components can also be installed from their repositories. The backend needs Python 3.10 or later, Poetry and, as with the PyPI package, a PostgreSQL database:

    git clone https://github.com/unibo-dtm-se-2526-PRONTO/artifact-backend.git
    cd artifact-backend
    poetry install
    cp .env.example .env

The `.env` needs at least `SECRET_KEY`, `DEBUG=True` and the `DB_*` keys of the database. `poetry install` also installs the `pronto-helpdesk` command in the Poetry environment, so the commands are the same as for the PyPI package, run through Poetry:

    poetry run pronto-helpdesk migrate
    poetry run pronto-helpdesk seed_offices
    poetry run pronto-helpdesk seed_faqs
    poetry run pronto-helpdesk createsuperuser
    poetry run pronto-helpdesk runserver

The web app needs Node.js 22.18 or later in the 22 line, or 24.12 or later. The build writes the same static files that the npm package contains to `dist/`:

    git clone https://github.com/unibo-dtm-se-2526-PRONTO/artifact-frontend.git
    cd artifact-frontend
    npm ci
    npm run build

## Configuration

The backend reads its configuration from environment variables or from a `.env` file in the folder it runs from; `.env.example` in the backend repository lists every key.

With Docker, `docker-compose.release.yml` already sets all of them for a trial on the local machine. The only one to change is `SECRET_KEY`, whose default is not secret: set it in the shell or in a `.env` next to the compose file before exposing the installation to anyone else. Installations from PyPI or from source need a `.env` with at least `SECRET_KEY`, `DEBUG` and the `DB_*` keys.

| Key | Default | Meaning |
|---|---|---|
| `SECRET_KEY` | none | Required |
| `DEBUG` | `False` | Debug mode; `True` also serves the admin's static files |
| `ALLOWED_HOSTS` | empty | Comma-separated, required when `DEBUG` is off |
| `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT` | | PostgreSQL connection |
| `DB_SSLMODE` | `require` | `disable` for a local server without SSL |
| `CHROMA_HOST`, `CHROMA_PORT` | empty, `8000` | Semantic FAQ search; off when `CHROMA_HOST` is empty |
| `EMAIL_BACKEND` | console | The SMTP backend sends real e-mails (`EMAIL_HOST`, `EMAIL_PORT`, ...) |
