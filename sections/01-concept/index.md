---
title: Concept
has_children: false
nav_order: 2
---

# Concept

Pronto is a web application that digitizes and automates the telephone-based helpdesk of the University of Bologna's Cesena Campus. Instead of calling, students use a self-service platform to book appointments with the relevant university office. Where possible, they get an answer straight away from an automatic FAQ-matching engine built on real, anonymised past questions and answers, and skip the appointment altogether.

## Type of product

Pronto is a client-server web application, structured as a 3-tier system: a single-page frontend shared by students and employees, an API backend that handles booking, FAQ matching, authentication, and notifications, and a data tier made of a relational database for persistence plus a vector store for the semantic part of FAQ matching. No native mobile app, CLI, or library is planned; every actor uses the system through a web browser.

## Use case collection

### Actors

Two primary roles use the system:

- **Student**: needs information from a specific university office (e.g. the job orientation office, for internships) and wants either an immediate answer or an appointment.
- **Employee**: a staff member of a university office who manages their own working shifts/availability and handles incoming appointment requests.

### Where are the users

Students and employees access Pronto through a standard web browser from anywhere with internet access: on campus, at home, or on the move. The offices are on the Cesena Campus, but users do not need to be on site to use the system.

### When and how frequently do they interact with the system

- Students interact **sporadically and on demand**, whenever they have a specific question or need (e.g. around enrollment or internship-search periods). Usage is expected to come in bursts around office-specific deadlines.
- Employees interact **regularly, as part of their daily work routine**: they set up their availability once per period (e.g. weekly), then handle the resulting appointments as they arrive during working hours.

### How do they interact with the system, and with which devices

Both roles use the same responsive single-page web application, from desktop or mobile browsers. A student first asks a free-text question about a given office. The FAQ-matching engine suggests the closest answer from past questions, and the student decides whether it resolves their request. If it does, no appointment is created. Otherwise, the student browses that office's available time slots and books one, and the original question is attached to the appointment for the employee who handles it. E-mail notifications tell users when an appointment is booked or cancelled, so they do not need to keep the application open.

### Does the system need to store user data

Yes. Pronto persists:

- **Identity data**: for students, name, surname, student ID (matricola), course of study, and institutional e-mail; for employees, name, surname, office, and working shifts. Passwords are stored hashed.
- **Booking data**: appointments, each associated with a time slot, an office, the employee assigned to handle it, a free-text question/note from the student (with the FAQ answer they were suggested, if any), and a lifecycle status (booked, cancelled, or completed).
- **Question data**: every question asked, with the office it was asked about and the FAQ suggested for it. These records have no link to the student who asked and are used to find out what the knowledge base does not cover yet.
- **Knowledge base data**: a FAQ collection built from real, anonymised historical helpdesk questions and answers, used to suggest an answer to a student's question before any appointment is created.

All persistent data lives in a single relational database on the backend. The only other store is the vector store used by FAQ matching, which holds the embeddings of the published FAQs. That data is derived: nobody edits it by hand, and it can be rebuilt from the database at any time. Clients keep only transient UI/session state and store no sensitive data locally.

### Multiple roles

Pronto distinguishes between the **Student** and **Employee** roles at registration time, and the authenticated user's role decides which actions are available and which data is visible. A third role, **Administrator**, is reserved for the staff who run the helpdesk. It cannot be obtained through registration; administrators oversee all appointments and maintain offices and FAQs.
