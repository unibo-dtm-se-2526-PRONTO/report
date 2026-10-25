---
title: Concept
has_children: false
nav_order: 2
---

# Concept

Pronto is a web application that aims to replace the telephone helpdesk of the University of Bologna's Cesena Campus. Instead of calling an office, students write their question on the platform. The system first looks for an answer among real, anonymised questions and answers from past helpdesk requests, and only when that answer is not enough, does the student book an appointment with the office that can help.

## Type of product

Pronto is a client-server web application with a 3-tier structure. A single-page frontend serves every role. The backend exposes a REST API for authentication, FAQ matching, booking and notifications. The data tier is a relational database, together with a vector store used by the semantic part of FAQ matching. There is no native mobile app: everyone uses Pronto from a desktop or mobile browser.

## Use case collection

Pronto has three actors. Students need information from one of the campus offices (guidance, the student administration office, international relations, internships) and want either an immediate answer or an appointment. Employees work in one of these offices: they declare their weekly shifts, from which the system derives the bookable slots, and handle the appointments assigned to them. Administrators run the helpdesk: they oversee all appointments and maintain users, offices and FAQs through the Django admin. Students and employees register by themselves, and their role follows from the domain of their institutional e-mail; an administrator account cannot be obtained through registration.

Students use Pronto sporadically and from anywhere, while Employees use it every working day: they declare their shifts once, change them when needed, and follow the appointments as they arrive.

A student writes a question in free text without choosing an office, since they often do not know which one is responsible. The system compares the question with the FAQs of every office and suggests the closest answer, together with the office it belongs to. If the answer resolves the request, the student says so and no appointment is created. Otherwise the student books a free slot with that office, or with one picked from the list when no answer was found. Students see the slots of an office, not of a person: the system assigns each booking to an employee who is free in that slot, so the work is spread among colleagues and nobody is booked twice for the same slot. 

The question and the suggested answer travel with the appointment, so the employee knows what the student has already read. An appointment can be cancelled before it starts by the student, the employee or an administrator, and the employee marks it as completed after the meeting. 

E-mails tell the student and the employee when an appointment is booked or cancelled, so neither has to keep the application open.

## Persistent data

All persistent data lives in a single relational database. For every user Pronto stores name, surname, institutional e-mail and role, with the password hashed; students also have their student ID (matricola) and degree programme, employees their office and weekly shifts. Each appointment records the student, the office, the assigned employee, the slot, the question with its language, the FAQ the student was shown (if any) and a status: booked, cancelled or completed. Every question asked is stored with its office, the suggested FAQ and whether that answer solved the request, but with no link to the student who asked it: these records show what the knowledge base does not cover yet. The FAQs, in Italian and English, come from real helpdesk requests and are anonymised before being imported. The vector store holds only the embeddings of the published FAQs, which are derived data and can be rebuilt from the database at any time. The browser keeps only the login token and the chosen language.
