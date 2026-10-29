---
title: Future work
has_children: false
nav_order: 13
---

# Known issues and future work

## What is missing

### E-mail verification

An account is active as soon as it is registered, and the address it was registered with is never checked.

### Token expiry

Authentication uses Django REST Framework's token, which never expires: it stops working only when the user logs out, which deletes it. Expiring tokens, or a periodic rotation, would limit the damage of a stolen one.

### Exceptions to the shifts

Shifts repeat every week, and there is no way to record holidays, sick leave or a single day off. A dated exception to a shift would close this gap, checked against the appointments already booked in the same way as removing a shift.

## What does not work as it should

### The semantic index can fall out of sync with the database

We chose Chroma, a vector store running as a separate service next to the backend, to show within the project how a dedicated search component fits into the architecture.

In deployment, where the application is started with Docker Compose, the backend uses the database and the Chroma of the same stack, so each database has one index and the two stay aligned.

During development, the backend on each developer's machine connects to the shared database on Neon, but to the local Chroma of that machine. An FAQ that one developer adds or rephrases is then missing from the other indexes, or indexed there with its old question, until `rebuild_faq_index` is run on each machine.

One fix is to host a single Chroma server outside Docker Compose, on Chroma Cloud or on a server of our own exposed over HTTPS, and point every developer's backend at it through `CHROMA_HOST`. We left it out because it adds an external service to run, secure and pay for.

## Future developments

### University single sign-on

Users could log in with their university account through federated single sign-on. In Django this can be done with a library such as `djangosaml2`.

### Online appointments

An appointment could take place in a video call as well as in person. The appointment would need a *mode* field, chosen by the student when booking. For an online appointment, Pronto would create a Microsoft Teams meeting through the Microsoft Graph API (Teams is the tool the university already uses) and add its link to the confirmation e-mails sent to the student and the employee.

### A conversational assistant

The FAQ matching could grow into a conversational assistant in two steps.

In the first, when no FAQ answers the question, the assistant asks the student a few follow-up questions and passes a summary to the employee together with the appointment. Today the employee receives the question as the student wrote it and, if one was suggested, the FAQ that did not help; with the summary they could prepare before meeting the student. This step can reuse the multilingual embeddings and the Chroma index that already exist.

In the second, the assistant answers open questions about calls for applications, deadlines, study plans or enrollment procedures, retrieving the information from the university website (Retrieval-Augmented Generation).

Both steps involve a language model, and with it the risk of unpredictable answers. A model reached through an external API also brings per-query costs and sends what students write outside the system, so a model running locally would be preferable. The second step also depends on scraping a website that Pronto does not control.
