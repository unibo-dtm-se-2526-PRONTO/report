# Pronto — Report

Software Engineering project work report for **Pronto**, a web application that
automates the phone helpdesk of the Campus of Cesena (University of Bologna).
Pronto lets students book appointments with university offices and get
FAQ-based answers before confirming a booking.

## Abstract

Student support at the Campus of Cesena is handled through a phone
helpdesk, where many recurring questions could be answered without human
intervention. Pronto is a web application that automates this process: it lets
students book appointments with the relevant university offices and, before
confirming, answers to frequently asked questions so that simple
requests are resolved immediately. This repository contains the full
engineering report, documenting the decisions behind the implementation.

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

## Contributing to the report

The report content lives in the `sections/` directory as Markdown files. The
website is generated automatically by GitHub Actions on every push to `main`,
so committing changes is enough to update the published site.

To preview the site locally you need **Ruby** installed
([instructions](https://jekyllrb.com/docs/installation/)). Then, from the root
of the repository:

```bash
# 1. install the Jekyll dependencies
bundler install

# 2. serve the site locally
bundler exec jekyll serve
```

The preview will be available at <http://127.0.0.1:4000> (the exact URL is
printed by the command). Any change to the `.md` files is reflected live until
you stop the server with `Ctrl+C`.

<!-- References -->

[template-repo]: https://github.com/unibo-dtm-se/template-project-work
[template-site]: https://unibo-dtm-se.github.io/template-project-work
[course-site]: https://www.unibo.it/en/study/phd-professional-masters-specialisation-schools-and-other-programmes/course-unit-catalogue/course-unit/2024/466765
[general-forum]: https://virtuale.unibo.it/mod/forum/view.php?id=1885625
[project-forum]: https://virtuale.unibo.it/mod/forum/view.php?id=1885626
[markdown-cheatsheet]: https://www.markdownguide.org/cheat-sheet
[jeckyll-home]: https://jekyllrb.com/
