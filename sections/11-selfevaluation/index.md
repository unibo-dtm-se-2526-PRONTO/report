---
title: Self-Evaluation
has_children: false
nav_order: 12
---

# Self-Evaluation

Each member wrote their own section and is responsible only for it.

## Giulio Salotti

The first part of the project was shared work with Sara: we analysed the helpdesk of the campus, agreed on the requirements and the user stories, and sketched the design of the system. Once the implementation started, the frontend became my responsibility. I built the Vue.js application: the student flow, which goes from the question to the booked appointment, the screens where employees manage their queue, shifts and profile, sign-in and registration with routing by role, and the interface in Italian and English. Most of its test suite is mine too: 218 tests that cover every line of the code and run on Linux, Windows and macOS. On the backend I took care of what happens before a booking: the import of the FAQs and the anonymisation of personal data, the first version of the FAQ matching with PostgreSQL full-text search, the API that stores the questions of the students, and the user and office models. I also packaged the application in Docker, wrote the compose file that runs it from the released images, and set up the release of the frontend to npm and GitHub Container Registry.

Claude Code helped me write the code. I read and checked what it suggested before keeping it, and I tracked down myself the problems that came up when something did not work.

Not everything came easily. Testing a frontend was new for me, and I wrote most of the tests after the screens instead of before them, so they verified the code more than they shaped it. The release pipeline took more effort than I had planned: publishing the frontend to npm needed several fixes, and a dry run of the whole pipeline at the beginning would have avoided most of them. It was also the first time I developed a project inside a GitHub organisation, with a separate repository for each part of it: the backend, the frontend and the report. Keeping three repositories aligned, each with its own branches, pipeline and releases, was a very enriching experience.

As for the collaboration, I made sure that the frontend followed the backend API that Sara was building, and we discussed together every change that affected both sides.

Looking at the product, Pronto does several things well: a student first receives the closest answer from real past questions, bookings stay correct even when they are made at the same time, and the whole application is tested and starts with a single command. It also has limits: it is not ready for production, since e-mails are only written to the log and there is no HTTPS; e-mail addresses are never verified and login tokens never expire; and shifts cannot record holidays or days off. Some of these gaps come from my own choices, such as removing the e-mail verification so that the application was easier to try.

In the end, Pronto taught me how to build a complete user interface on top of a real API, and how packaging and automated releases turn a working project into something that other people can install and run.

I want to thank Sara for the work she did and for the dedication she put into the project from the first day to the last. Her care for the backend, the tests and the organisation of the group made a real difference to the result.

## Sara Ladisa

Together with Giulio, I defined the problem that Pronto addresses, wrote the requirements and the user stories, and designed the system. After this shared phase I was responsible for the architecture and for the backend implementation: the Django project and its apps, the REST API, the relational database and the integration with Chroma used by the semantic part of FAQ matching. I followed a test-driven approach, writing each test before the code it covers, and I took care of the test suite. I also handled the releases on TestPyPI, PyPI, GitHub Container Registry and GitHub, and set up semantic-release and Dependabot.

During the implementation, I used Claude Code to support the writing of the code. I reviewed every part it produced before accepting it, and when something did not behave as expected I debugged it myself.

Some aspects of the project were new to me and needed some practice. Before Pronto I had little experience with a branching workflow on Git and GitHub. Connecting the software to external services was also unfamiliar: the shared Neon database, Chroma, the package registries and the SMTP server each have their own credentials and configuration.

From a teamwork perspective, I coordinated the work of the group, splitting the tasks between the backend and the frontend and keeping track of the deadlines.

Overall, Pronto allowed me to follow a software system from the analysis of the problem to its release, and to see in practice how tests, automated pipelines and dependency updates keep a project maintainable.
