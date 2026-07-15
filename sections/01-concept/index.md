---
title: Concept
has_children: false
nav_order: 2
---

# Concept

Pronto is a web application that digitizes and automates the telephone-based helpdesk of the University of Bologna's Cesena Campus. It replaces phone-in requests for information with a self-service platform: students can book appointments with the relevant university office, and, where possible, get their doubts resolved directly through an automatic FAQ-matching engine built on real, anonymised past questions and answers, without needing an appointment at all.

## Type of product

Pronto is a client-server web application, structured as a 3-tier system: a single-page frontend (shared by students and employees), an API backend encapsulating booking, FAQ-matching, authentication, and notification logic, and a relational database for persistence. No native mobile application, CLI, or library is planned: every actor accesses the system exclusively through a web browser.

## Use case collection

### Actors

Two primary roles use the system:

- **Student**: needs information from a specific university office (e.g. the job orientation office, for internships) and wants to either get an immediate answer or book an appointment.
- **Employee**: a staff member of a university office who manages their own working shifts/availability and handles incoming appointment requests.

### Where are the users

Students and employees are geographically distributed: they access Pronto from any location with internet access (on campus, at home, on the move) through a standard web browser. The domain itself is tied to the Cesena Campus and its offices, but there is no requirement for users to be physically on-site to use the system.

### When and how frequently do they interact with the system

- Students interact **sporadically and on demand**, whenever they have a specific question or need (e.g. around enrollment or internship-search periods), rather than routinely. Usage is expected to be bursty, concentrated around office-specific deadlines.
- Employees interact **regularly, as part of their daily work routine**: they set up their availability once per period (e.g. weekly), and then handle the resulting appointments as they arrive during working hours.

### How do they interact with the system, and with which devices

Both roles interact through the same responsive single-page web application, from desktop or mobile browsers. A student first asks a free-text question about a given office; the FAQ-matching engine suggests the best-matching answer drawn from past questions, and the student decides whether it resolves their request. If it does, no appointment is created. Otherwise, the student browses that office's available time slots, selects one, and books it, with the original question attached for the handling employee. E-mail notifications complement the web interface, informing users of state changes (a new appointment being booked, or an existing one being cancelled or completed) without requiring them to keep the application open.

### Does the system need to store user data

Yes. Pronto persists:

- **Identity data**: for students, name, surname, student ID (matricola), course of study, and institutional e-mail; for employees, name, surname, office, and working shifts. Passwords are stored hashed, never in clear text.
- **Booking data**: appointments, each associated with a time slot, an office, a free-text question/note from the student, and a lifecycle status (booked, cancelled, or completed).
- **Knowledge base data**: a FAQ collection built from real, anonymised historical helpdesk questions and answers, used to automatically suggest an answer to a student's question before any appointment is created.

All persistent data lives centrally, in a single relational database on the backend side; clients only keep transient UI/session state and never store sensitive data locally.

### Multiple roles

As described above, Pronto distinguishes between the **Student** and **Employee** roles at registration time, and every subsequent interaction — which actions are available, which data is visible — depends on the authenticated user's role.
