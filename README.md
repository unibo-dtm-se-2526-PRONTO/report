# Pronto - Report

This repository contains the engineering report of Pronto, a web application that automates the phone helpdesk of the Campus of Cesena (University of Bologna).

## Abstract

"Pronto" is a web application that replaces the telephone helpdesk of the Cesena Campus of the University of Bologna with a self-service platform. A student picks a university office and types a question. The system first suggests the closest answer from a knowledge base of real, anonymised past questions; if that answer does not help, the student books one of the office's free time slots. Employees declare their working shifts, and each booking goes to an employee who is free in that slot, so the work is spread evenly among colleagues and nobody is booked twice for the same slot. Bookings and cancellations are confirmed by e-mail. The project was developed for the Software Engineering course at Unibo. It follows a 3-tier architecture built with Django and Vue.js, and matches FAQs by combining PostgreSQL full-text search with semantic search. Development follows Git Flow and Conventional Commits, and GitHub Actions runs the tests on every push.

## Project repositories

Pronto is split across three repositories:

- **Report** this repository
- **Backend** https://github.com/unibo-dtm-se-2526-PRONTO/artifact-backend.git
- **Frontend** https://github.com/unibo-dtm-se-2526-PRONTO/artifact-frontend.git

## Authors & context

- [Sara Ladisa](mailto:sara.ladisa@studio.unibo.it)
- [Giulio Salotti](mailto:giulio.salotti@studio.unibo.it)

Developed for the [Software Engineering course](https://www.unibo.it/) of the
Digital Transformation Management master's degree, University of Bologna. Supervisor: Prof. Giovanni Ciatto
([`gciatto`](https://github.com/gciatto)).

## Running the report locally

The report is a [Jekyll](https://jekyllrb.com/) website: its content lives in `index.md` and in the `sections/` directory as Markdown files.

1. Install Ruby 3.x ([instructions](https://jekyllrb.com/docs/installation/)); on macOS, e.g. `brew install ruby`.
2. From the root of the repository, install the dependencies and start the server:

   ```bash
   bundle install
   bundle exec jekyll serve
   ```

3. Open <http://127.0.0.1:4000> in the browser. Jekyll rebuilds the site whenever a `.md` file changes, so a page refresh shows the edit. Stop the server with `Ctrl+C`.
