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

“Pronto” is a web application that replaces the telephone helpdesk of the Cesena Campus of the University of Bologna with a self-service platform. A student asks a question in free text: the system first suggests the closest answer from a knowledge base of real, anonymised past questions, and only if it does not help does the student book one of the related office’s available time slots. Employees declare their working shifts, and each booking is assigned to a free employee, spreading the work evenly and never giving two students the same employee in the same slot. E-mail notifications confirm bookings and cancellations. 
This project, developed for the Software Engineering course at Unibo, follows a 3-tier architecture built with Django and Vue.js, and combines PostgreSQL full-text search with Chroma semantic search for FAQ matching. Development follows Git Flow and Conventional Commits, with GitHub Actions running the tests on every push.

## Disclaimer

During the preparation of this work, Claude Code (Anthropic) was used to assist the implementation of the software, while all the design decisions were made by the authors.

After using this tool, the authors reviewed and edited the content as needed
and take full responsibility for the content of the final report and artifact.
