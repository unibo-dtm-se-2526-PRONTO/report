---
title: Home
layout: home
has_children: false
nav_order: 1
---

# Pronto

### Authors

- [Sara Ladisa](mailto:sara.ladisa@studio.unibo.it)
- [Giulio Salotti](mailto:giulio.salotti@studio.unibo.it)

## Abstract

"Pronto" is a web application that replaces the telephone helpdesk of the Cesena Campus of the University of Bologna with a self-service platform. A student picks a university office and types a question. The system first suggests the closest answer from a knowledge base of real, anonymised past questions; if that answer does not help, the student books one of the office's free time slots. Employees declare their working shifts, and each booking goes to an employee who is free in that slot, so the work is spread evenly among colleagues and nobody is booked twice for the same slot. Bookings and cancellations are confirmed by e-mail. The project was developed for the Software Engineering course at Unibo. It follows a 3-tier architecture built with Django and Vue.js, and matches FAQs by combining PostgreSQL full-text search with semantic search. Development follows Git Flow and Conventional Commits, and GitHub Actions runs the tests on every push.

## Disclaimer

During the preparation of this work, the authors used Claude Code (Anthropic) to:

- draft and revise parts of this report;
- generate part of the source code and of the automated tests;
- review the code.

The requirements, the design decisions and the calibration of the FAQ matching thresholds are the authors' own, and every output of the tool was checked by the authors before being included.

After using this tool, the authors reviewed and edited the content as needed
and take full responsibility for the content of the final report and artifact.
