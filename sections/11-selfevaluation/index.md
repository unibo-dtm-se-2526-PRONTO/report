---
title: Self-Evaluation
has_children: false
nav_order: 12
---

# Self-Evaluation

Each member wrote their own section and is responsible only for it.

## Giulio Salotti

Together with Sara, I defined the problem that Pronto addresses, wrote the requirements and the user stories, and designed the system. After this shared phase I was responsible for the frontend: the Vue.js application with the student flow from the question to the booked appointment, the employee screens for the queue, the shifts and the profile, sign-in and registration with routing by role, and the bilingual interface. I also wrote most of its test suite, 218 tests that cover all the lines of the code and run on Linux, Windows and macOS. On the backend I worked on the parts that come before a booking: the import of the FAQs with the anonymisation of personal data, the first version of the FAQ matching based on PostgreSQL full-text search, the API that records the questions of the students, and the user and office models. Finally, I packaged the application in Docker, wrote the compose file that starts it from the released images, and set up the frontend release on npm and GitHub Container Registry.

During the implementation, I used Claude Code to support the writing of the code. I reviewed every part it produced before accepting it, and when something did not behave as expected I debugged it myself.

Some aspects of the project were new to me and needed some practice. Testing a frontend was one of them: I wrote most of the tests after the screens rather than before, so they confirmed the code more than they guided its design. The release pipeline was also harder than I expected. Publishing the frontend to npm took several fixes before it worked, and a dry run of the whole pipeline at the start would have saved most of them.

From a teamwork perspective, I kept the frontend aligned with the backend API that Sara was building, and agreed with her on the changes that touched both sides.

Pronto has some clear strengths: a student first receives the closest answer from real past questions, bookings stay correct even when they are made at the same time, and the whole application is tested and can be started with a single command. It also has some weaknesses: it is not ready for production, since e-mails are only written to the log and there is no HTTPS; e-mail addresses are never verified and login tokens never expire; and shifts cannot record holidays or days off. Some of these gaps come from choices I made, such as removing the e-mail verification to make the application easier to try.

Overall, Pronto allowed me to build a complete user interface on top of a real API, and to see how packaging and automated releases turn a working project into something other people can install and run.

## Sara Ladisa

Together with Giulio, I defined the problem that Pronto addresses, wrote the requirements and the user stories, and designed the system. After this shared phase I was responsible for the architecture and for the backend implementation: the Django project and its apps, the REST API, the relational database and the integration with Chroma used by the semantic part of FAQ matching. I followed a test-driven approach, writing each test before the code it covers, and I took care of the test suite. I also handled the releases on TestPyPI, PyPI, GitHub Container Registry and GitHub, and set up semantic-release and Dependabot.

During the implementation, I used Claude Code to support the writing of the code. I reviewed every part it produced before accepting it, and when something did not behave as expected I debugged it myself.

Some aspects of the project were new to me and needed some practice. Before Pronto I had little experience with a branching workflow on Git and GitHub. Connecting the software to external services was also unfamiliar: the shared Neon database, Chroma, the package registries and the SMTP server each have their own credentials and configuration.

From a teamwork perspective, I coordinated the work of the group, splitting the tasks between the backend and the frontend and keeping track of the deadlines.

Overall, Pronto allowed me to follow a software system from the analysis of the problem to its release, and to see in practice how tests, automated pipelines and dependency updates keep a project maintainable.
