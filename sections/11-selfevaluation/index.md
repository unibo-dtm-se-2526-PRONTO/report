---
title: Self-evaluation
has_children: false
nav_order: 12
---

# Self-evaluation

## Sara Ladisa

Together with Giulio, I defined the problem that Pronto addresses, wrote the requirements and the user stories, and designed the system. After this shared phase I was responsible for the architecture and for the backend implementation: the Django project and its apps, the REST API, the relational database and the integration with Chroma used by the semantic part of FAQ matching. I followed a test-driven approach, writing each test before the code it covers, and I took care of the test suite. I also handled the releases on TestPyPI, PyPI, GitHub Container Registry and GitHub, and set up semantic-release and Dependabot.

During the implementation, I used Claude Code to support the writing of the code. I reviewed every part it produced before accepting it, and when something did not behave as expected I debugged it myself.

Some aspects of the project were new to me and needed some practice. Before Pronto I had little experience with a branching workflow on Git and GitHub. Connecting the software to external services was also unfamiliar: the shared Neon database, Chroma, the package registries and the SMTP server each have their own credentials and configuration.

From a teamwork perspective, I coordinated the work of the group, splitting the tasks between the backend and the frontend and keeping track of the deadlines.

Overall, Pronto allowed me to follow a software system from the analysis of the problem to its release, and to see in practice how tests, automated pipelines and dependency updates keep a project maintainable. 

## Giulio Salotti

Together with Sara, I defined the problem that Pronto addresses, wrote the requirements and the user stories, and designed the system. During the implementation I supported the work on the project in general and focused mainly on the frontend: the single-page application in Vue, its views for students, employees and administrators, and its connection to the REST API of the backend.

[NEEDS: how Giulio used Claude Code, the challenges he faced, his view of the teamwork and his overall reflection, written by Giulio]
