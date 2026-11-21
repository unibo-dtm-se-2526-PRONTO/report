---
title: Future work
has_children: false
nav_order: 13
---

# Known issues and future work

# Known Issues and Future Work

## What is missing

### E-mail verification

An account is active as soon as it is registered, and the address it was registered with is never checked.

### Token expiry

Authentication uses Django REST Framework's token, which never expires: logging out deletes it, and nothing else does. Expiring tokens, or a periodic rotation, would limit the damage of a stolen one.

### Exceptions to the shifts

Shifts repeat every week, and there is no way to record holidays, sick leave or a single day off. Employees would need to add a dated exception to a shift, which Pronto would check against the appointments already booked, as it does when a shift is removed.

## Future developments

### Production deployment

The release stack is meant for a trial on a single machine. Opening PRONTO to the campus would need:

- a server with a public host name;
- nginx as the single entry point, with an HTTPS certificate, forwarding the API to the backend and serving the admin's static files;
- gunicorn in place of Django's development server, with `DEBUG` off and a real `SECRET_KEY`;
- an SMTP server for the e-mails, which today are only printed in the log.

### University single sign-on

Users could log in with their university account through federated single sign-on. In Django this can be done with a library such as `djangosaml2`.

### Online appointments

An appointment could take place in a video call as well as in person. The appointment would need a *mode* field, chosen by the student when booking. For an online appointment, Pronto would create a Microsoft Teams meeting through the Microsoft Graph API (Teams is the tool the university already uses) and add its link to the confirmation e-mails sent to the student and the employee.

### A conversational assistant

The FAQ matching could grow into a conversational assistant in two steps.

In the first, when no FAQ answers the question, the assistant asks the student a few follow-up questions and passes a summary to the employee together with the appointment. Today the employee receives the question as the student wrote it and, if one was suggested, the FAQ that did not help. A summary would let them prepare before meeting the student. This step can reuse the multilingual embeddings and the Chroma index that already exist.

In the second, the assistant answers open questions about calls for applications, deadlines, study plans or enrollment procedures, retrieving the information from the university website (Retrieval-Augmented Generation).
