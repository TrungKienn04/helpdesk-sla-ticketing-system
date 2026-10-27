# Helpdesk & SLA Ticketing System

A full-stack portfolio project for managing IT support requests, ticket assignment, customer communication, service-level agreements (SLAs), and operational audit history.

The system is designed around a controlled ticket lifecycle: Customers submit support requests, Admins review and confirm intake details, Agents request or receive assignments, and the application tracks response and resolution deadlines throughout ticket processing.

> **Project status:** Planning and foundation stage. The documentation defines the MVP requirements, authorization rules, business workflows, SLA behavior, and proposed relational data model. Application implementation and automated verification are separate work items; planned features must not be considered complete until implemented and tested.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [User Roles](#user-roles)
- [Core Business Workflow](#core-business-workflow)
- [Ticket Statuses](#ticket-statuses)
- [SLA Rules](#sla-rules)
- [MVP Features](#mvp-features)
- [Technology Stack](#technology-stack)
- [Architecture and Data Model](#architecture-and-data-model)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Configuration](#environment-configuration)
- [Testing and Quality Goals](#testing-and-quality-goals)
- [Security and Data Integrity](#security-and-data-integrity)
- [Scope Exclusions](#scope-exclusions)
- [Development Roadmap](#development-roadmap)
- [Project Documentation](#project-documentation)
- [Portfolio Objectives](#portfolio-objectives)

---

## Overview

The Helpdesk & SLA Ticketing System is intended for a small IT or customer-support team that needs a reliable way to receive, review, assign, process, and track support requests.

The MVP uses a **single-workspace model** with three primary roles:

- **Customer:** Submits support requests and follows the progress of their own tickets.
- **Agent:** Processes assigned tickets, communicates with Customers, and records public replies or internal notes.
- **Admin:** Reviews incoming requests, manages assignments, administers users and SLA policies, and oversees operational reporting.

The primary engineering goals are to demonstrate:

- Role-based authorization and ticket-level access control.
- A well-defined ticket state machine.
- A review-and-approval workflow for ticket intake and assignment.
- Separate tracking for First Response SLA and Resolution SLA.
- Transactional data updates with consistent history records.
- A relational data model that preserves important business decisions.
- Automated tests and API documentation.

## Problem Statement

Support requests can become difficult to manage when ticket ownership, status updates, customer conversations, and time-based service commitments are handled informally.

This project provides one structured workflow for recording each request, deciding whether it can enter normal processing, establishing the confirmed Priority and SLA targets, assigning an eligible Agent, and tracking the ticket until it is resolved and closed.

The system also records important operational decisions so that users can understand what happened, who performed an action, and why a ticket's confirmed values or assignment changed.

---

## User Roles

### Customer

Customers can:

- Create support tickets for themselves.
- Propose a Priority and supported SLA targets during intake.
- View the tickets for which they are the requester.
- Add permitted public replies to their own tickets.
- View permitted ticket events, confirmed SLA information, and their own in-app notifications.
- See the final intake decision and any explanation for an Admin's adjustments.

Customers cannot:

- View another Customer's tickets.
- Create internal notes or read staff-only notes.
- Assign Agents or directly change ticket-processing statuses.
- Override confirmed Priority or SLA targets through an API request.
- Approve intake decisions or Agent claim requests.
- Reopen or comment on a `CLOSED` ticket in the MVP.

### Agent

Agents can:

- View tickets within the single workspace, subject to the access rules.
- View accepted, unassigned tickets eligible for claiming.
- Request to claim an eligible ticket.
- Process a ticket only after its assignment is confirmed.
- Add public replies and internal notes.
- Perform permitted status transitions.
- Resolve tickets and reopen resolved tickets when the workflow allows it.

A pending claim request does not give an Agent ownership of a ticket or permission to process it.

Agents cannot manage user roles, make intake decisions, approve claim requests, or change company-wide SLA policies.

### Admin

Admins can:

- Review incoming tickets.
- Accept an intake proposal unchanged, adjust its Priority or SLA values and accept it, or reject it.
- Provide mandatory explanations when rejecting a ticket or adjusting proposed values.
- Approve or reject Agent claim requests.
- Assign, reassign, or unassign Agents through authorized operations.
- Manage users and their roles.
- Manage supported company SLA policies.
- View operational dashboards, reports, ticket history, and administrative audit records.

Admin actions must follow valid workflow and data-integrity rules; administrative access does not justify bypassing the required history or SLA records.

---

## Core Business Workflow

### 1. Ticket creation and intake review

1. A Customer creates a ticket and submits a proposed Priority and supported SLA targets.
2. The system creates the ticket with `PENDING_REVIEW` as its intake status.
3. The original proposal is stored separately from any values that will later be confirmed.
4. An Admin reviews the ticket and chooses one of the following outcomes:
   - **Accept unchanged:** Confirm the submitted values.
   - **Adjust and accept:** Confirm revised values and provide a mandatory explanation for the changes.
   - **Reject:** Provide a mandatory rejection reason.
5. If accepted, the ticket enters normal processing with status `OPEN`. The confirmed Priority, SLA policy, target-minute snapshots, acceptance timestamp, and initial SLA cycles are recorded.
6. If rejected, the ticket remains outside normal processing. It cannot be assigned, and processing SLA cycles are not started.
7. The Customer receives an in-app notification of the decision. If the Admin adjusts the proposal, the notification must show the original proposal, final confirmed values, the reason, and the decision details.

The Customer does not need to approve an Admin's adjustment as a separate step, but must be informed immediately.

### 2. Agent claim requests and assignment

An accepted ticket can remain unassigned.

**Agent claim flow:**

1. An Agent requests an eligible, accepted, unassigned ticket.
2. The claim request is recorded as `PENDING`.
3. The Admin approves or rejects the request.
4. On approval, the system persists the assignment, marks the claim request `APPROVED`, creates assignment history, and records the relevant events and notifications.
5. Until approval and assignment are complete, the requesting Agent cannot process the ticket.

**Direct Admin assignment:**

An Admin may assign or reassign an accepted ticket without an Agent claim request.

At most one Agent can be the current assignee. Assignment updates must be concurrency-safe. Competing pending claim requests that are no longer eligible must be closed consistently.

### 3. Ticket processing and communication

After assignment, an authorized Agent can process the ticket using the permitted status transitions.

Public replies are visible to the requester. Internal notes are restricted to authorized staff. The system records important status changes, assignment changes, messages, intake decisions, and SLA events in the ticket timeline.

A ticket being assigned does not remove the requirement to validate the requested operation against the workflow.

### 4. Resolution, reopening, and closure

- `RESOLVED` means the issue has been marked as solved.
- `CLOSED` means the ticket has been formally closed and normal processing is complete.
- A resolved ticket can be reopened through the permitted `RESOLVED → IN_PROGRESS` transition.
- Reopening starts a new Resolution SLA cycle while preserving previous resolution and SLA history.
- `CLOSED` is final in the MVP. Customers cannot reopen or comment on a closed ticket.

---

## Ticket Statuses

The database maintains **intake status** separately from **processing status**. A ticket that is awaiting Admin review has not yet entered the normal processing lifecycle.

### Intake status

| Status           | Meaning                                                                               |
| ---------------- | ------------------------------------------------------------------------------------- |
| `PENDING_REVIEW` | Waiting for Admin intake review.                                                      |
| `ACCEPTED`       | Intake has been approved and processing may begin.                                    |
| `REJECTED`       | Intake has been rejected; normal processing and processing SLA tracking do not start. |

### Processing status

| Status                 | Meaning                                                | Resolution SLA          |
| ---------------------- | ------------------------------------------------------ | ----------------------- |
| `OPEN`                 | Accepted and waiting for an Agent to begin processing. | Running                 |
| `IN_PROGRESS`          | Being actively processed.                              | Running                 |
| `WAITING_FOR_CUSTOMER` | Waiting for information from the Customer.             | Paused                  |
| `RESOLVED`             | The issue has been marked as solved.                   | Current cycle completed |
| `CLOSED`               | Formally closed; final state in the MVP.               | No active cycle         |

### Allowed processing transitions

| From                   | To                     | Authorized actor |
| ---------------------- | ---------------------- | ---------------- |
| `OPEN`                 | `IN_PROGRESS`          | Agent or Admin   |
| `IN_PROGRESS`          | `WAITING_FOR_CUSTOMER` | Agent or Admin   |
| `IN_PROGRESS`          | `RESOLVED`             | Agent or Admin   |
| `WAITING_FOR_CUSTOMER` | `IN_PROGRESS`          | Agent or Admin   |
| `WAITING_FOR_CUSTOMER` | `RESOLVED`             | Agent or Admin   |
| `RESOLVED`             | `IN_PROGRESS`          | Agent or Admin   |
| `RESOLVED`             | `CLOSED`               | Agent or Admin   |

The backend must reject other transitions. Customers do not directly change processing status.

---

## SLA Rules

The MVP uses a **24/7 SLA clock**. Business-hour calendars, holidays, and advanced time-zone rules are excluded from the initial scope.

### First Response SLA

- Starts when an Admin accepts the ticket's intake.
- Completes when the assigned Agent creates the first qualifying public reply.
- Customer replies, internal notes, assignment changes, status changes, and automated notifications do not satisfy this SLA.
- The first-response result must be preserved and must not be overwritten by later replies.

### Resolution SLA

- Starts at intake acceptance.
- Pauses when the ticket enters `WAITING_FOR_CUSTOMER`.
- Resumes when the ticket returns to `IN_PROGRESS`.
- Completes when the ticket becomes `RESOLVED`.
- If a resolved ticket is reopened, a new Resolution SLA cycle begins; previous cycles remain in history.

### SLA policies and snapshots

Admins manage supported SLA policies. Customers propose supported values during ticket intake, and Admins confirm or adjust them.

Once intake is accepted, the ticket retains its confirmed target-minute snapshots. Subsequent edits to company policy affect future decisions and tickets, not existing tickets silently.

Intake waiting time is measured separately from processing SLA time. Rejected tickets never start processing SLA cycles.

### Example SLA defaults for demonstration

The following are **sample values for development and testing**, not universal service commitments.

| Priority | First Response target | Resolution target |
| -------- | --------------------: | ----------------: |
| `LOW`    |               8 hours |          72 hours |
| `MEDIUM` |               4 hours |          24 hours |
| `HIGH`   |                1 hour |           8 hours |
| `URGENT` |            15 minutes |           4 hours |

The application uses the policy actually proposed and confirmed for a ticket. The table above does not imply that Priority alone must determine SLA targets.

---

## MVP Features

The planned MVP includes:

- **Authentication and user management:** Accounts for Customer, Agent, and Admin with server-side authorization.
- **Ticket management:** Ticket creation, detail views, lists, search, filters, sorting, and pagination.
- **Intake review:** Admin acceptance, adjustment with a mandatory reason, or rejection with a mandatory reason.
- **Assignment workflow:** Agent claim requests, Admin approval/rejection, and direct assignment or reassignment.
- **Ticket communication:** Public replies and staff-only internal notes with enforced visibility rules.
- **Workflow enforcement:** Valid state transitions and restrictions on closed tickets.
- **SLA tracking:** First Response and Resolution targets, deadline calculations, Resolution SLA pause/resume, breach tracking, and reopening cycles.
- **History and audit:** Ticket events, assignment history, intake decisions, and system-wide administrative changes.
- **In-app notifications:** Notifications for relevant ticket and intake events, including explanations of Admin adjustments.
- **Basic reporting:** Operational dashboard and SLA summaries.
- **Automated tests and API documentation:** Tests for workflow, authorization, SLA calculations, and data integrity, plus OpenAPI documentation.

The list describes the intended MVP scope. A feature is complete only after its implementation and relevant tests have been verified.

---

## Technology Stack

| Layer              | Technology             | Responsibility                                                |
| ------------------ | ---------------------- | ------------------------------------------------------------- |
| Frontend           | React                  | User interface for Customers, Agents, and Admins.             |
| Frontend language  | TypeScript             | Typed application code and API data contracts.                |
| Styling            | Tailwind CSS           | Responsive interface styling.                                 |
| Backend runtime    | Node.js LTS            | Server-side JavaScript runtime.                               |
| Backend framework  | Express 5              | REST API routes and middleware.                               |
| Backend language   | TypeScript             | Business logic, validation types, and service contracts.      |
| Database           | MySQL 8.x with InnoDB  | Relational storage, foreign keys, transactions, and indexes.  |
| ORM and migrations | Prisma                 | Typed database access and schema migrations.                  |
| Input validation   | Zod                    | Validation for request bodies, parameters, and query strings. |
| Testing            | Vitest and Supertest   | Unit and API/integration testing.                             |
| API documentation  | OpenAPI and Swagger UI | Documenting and manually testing API endpoints.               |
| Version control    | Git and GitHub         | Source control, collaboration, and portfolio presentation.    |

Dependency versions and executable scripts will be pinned in the project manifests when the client and server are scaffolded.

---

## Architecture and Data Model

The application is planned as a modular full-stack monolith with a separate frontend and backend. The backend owns authorization, workflow validation, SLA calculations, transactions, and database access.

```text
React + TypeScript + Tailwind CSS
                |
                | REST API / JSON
                v
Node.js + Express 5 + TypeScript
                |
                | Zod validation
                | Authentication and authorization
                | Ticket workflow and SLA services
                | Prisma transactions
                v
          MySQL 8.x / InnoDB
```

### Core database entities

The proposed relational model includes:

| Entity                      | Responsibility                                                           |
| --------------------------- | ------------------------------------------------------------------------ |
| `users`                     | Customer, Agent, and Admin accounts.                                     |
| `sla_policies`              | Company-approved SLA target combinations.                                |
| `tickets`                   | Current ticket state, requester, confirmed values, and current assignee. |
| `ticket_intake_reviews`     | Original proposal and Admin intake-decision details.                     |
| `ticket_claim_requests`     | Requests from Agents to take unassigned tickets.                         |
| `ticket_assignment_history` | Assignment, reassignment, and unassignment history.                      |
| `ticket_messages`           | Public replies and internal notes.                                       |
| `ticket_sla_cycles`         | First Response and Resolution SLA tracking cycles.                       |
| `ticket_sla_pauses`         | Resolution SLA pause/resume intervals.                                   |
| `ticket_events`             | Ticket-level business timeline.                                          |
| `user_notifications`        | Per-user in-app notifications and read state.                            |
| `audit_logs`                | Administrative changes to system-wide data.                              |

Key integrity principles:

- Intake review and processing state are separate.
- Original proposed values are preserved even when Admin changes the confirmed values.
- Claim requests are distinct from actual assignments.
- Current assignment is stored on the ticket; assignment history is retained separately.
- SLA cycles and pause intervals preserve the history needed to explain deadlines.
- Critical business updates and their event records must be committed atomically.
- Foreign keys, uniqueness constraints, indexes, and service-level checks jointly protect data integrity.

For the complete entity definitions and relationships, see [`docs/04-data-model.md`](docs/04-data-model.md).

---

## Project Structure

The intended repository structure is:

```text
helpdesk-sla-ticketing-system/
├── client/                         # React + TypeScript + Tailwind CSS
├── server/                         # Express 5 + TypeScript API
├── docs/
│   ├── 01-project-scope.md
│   ├── 02-roles-and-permissions.md
│   ├── 03-workflow-and-sla.md
│   ├── 04-data-model.md
│   └── DAY-1-CHECKLIST.md
├── .env.example                    # Safe environment-variable template
├── .gitignore
└── README.md
```

This is the **target structure**, not a claim that all folders and source files already exist. The final structure may be refined as the implementation is created, but changes must remain consistent with the documented architecture and business rules.

---

## Getting Started

### Current status

The project is in its planning and foundation stage. The requirements and data model are being documented before the application skeleton is implemented.

Because the client and server package manifests and database migrations have not yet been verified as implemented, this README does not present unverified commands as if they were already executable.

### Planned local setup

Once the application skeleton is available, the setup process will be:

1. Clone the repository.
2. Install the client and server dependencies from their respective package manifests.
3. Install and start a compatible MySQL 8.x instance.
4. Copy `.env.example` to the appropriate local environment file and configure the database connection and required application secrets.
5. Apply Prisma database migrations.
6. Seed development data, including safe demo accounts and representative ticket scenarios.
7. Start the backend and frontend using the scripts defined in the package manifests.
8. Verify the health endpoint, authentication, intake review, assignment workflow, and SLA behavior.
9. Run the automated test suite and review the generated API documentation.

Exact commands will be added once the package scripts, environment-variable names, migration setup, and seed process have been established.

### Environment configuration

The implementation is expected to require at least a database connection setting and server-specific configuration. Authentication secrets, allowed frontend origins, and other settings will be defined explicitly during backend setup.

- Never commit `.env` files containing real secrets or credentials.
- Keep `.env.example` limited to safe placeholders.
- Do not use production credentials or real Customer data in development seeds.
- Keep local environment configuration separate for the client and server when necessary.

---

## Testing and Quality Goals

The test suite should verify behavior, not merely whether individual routes return successful responses.

### Authorization tests

- A Customer cannot access another Customer's ticket by changing an ID in an API request.
- A Customer cannot read internal notes or staff-only event metadata.
- An Agent cannot approve a claim request or decide ticket intake.
- An Agent cannot process a ticket while their claim remains pending.
- A user cannot read or update another user's in-app notifications.

### Workflow tests

- Pending intake cannot be assigned or processed.
- Rejected intake does not create processing SLA cycles.
- Admin acceptance creates confirmed ticket values and initial SLA cycles.
- An adjustment preserves the original proposal, requires an explanation, and generates the required notification.
- Claim approval and direct assignment preserve the single-current-assignee rule.
- Invalid state transitions return an appropriate conflict response.
- `CLOSED` is final in the MVP.

### SLA tests

- Processing SLA starts at acceptance rather than ticket creation.
- A qualifying public reply from the assigned Agent completes First Response SLA.
- Customer replies and internal notes do not complete First Response SLA.
- Resolution SLA pauses and resumes correctly.
- Pause durations extend the Resolution deadline correctly.
- Breaches are preserved in history.
- Reopening a resolved ticket creates a new Resolution SLA cycle without overwriting previous cycles.

### Data-integrity tests

- Ticket creation and intake review are atomic.
- Assignment updates and assignment-history records are atomic.
- Ticket events are recorded consistently with successful business changes.
- Competing claim requests cannot produce multiple current assignments.
- Changes to an SLA policy do not silently change existing tickets' confirmed target snapshots.

---

## Security and Data Integrity

The backend is the authoritative enforcement layer. Frontend route guards and hidden buttons are useful for user experience but are not security controls by themselves.

The implementation should follow these principles:

- Authenticate every protected request.
- Apply role authorization and ticket-level access checks on the server.
- Validate request bodies, parameters, and query strings using Zod.
- Reject invalid state transitions and unsupported Priority/SLA values.
- Use Prisma transactions for multi-record operations.
- Handle concurrent assignment and claim-approval attempts safely.
- Protect internal notes and staff-only events from Customer responses.
- Record important changes with actor, timestamp, and relevant before/after values.
- Store timestamps consistently in UTC.
- Deactivate referenced users and SLA policies instead of deleting records in ways that erase ticket history.
- Store password hashes, not plain-text passwords, and never expose credentials in logs or API responses.

The database model and application services must work together: a foreign key ensures a reference exists, while the service validates whether the referenced user, ticket, policy, or workflow state is eligible for the action.

---

## Scope Exclusions

The following are intentionally outside the initial MVP:

- Multi-tenant SaaS or multiple independent workspaces.
- Microservices, Kubernetes, Kafka, or RabbitMQ.
- Redis caching, BullMQ background workers, and WebSocket real-time updates.
- Outbound email, SMS, Slack, WhatsApp, and other external notifications.
- File attachments and dedicated file-storage infrastructure.
- AI ticket classification or chatbot support.
- Knowledge bases, live chat, and omnichannel intake.
- Advanced SLA business calendars, holidays, and complex time-zone rules.
- Dynamic workflow builders and configurable permission engines.

These enhancements may be evaluated after the core ticket lifecycle, assignment integrity, SLA calculations, and authorization behavior are implemented and tested.

---

## Development Roadmap

### Phase 1 — Requirements and foundation

- [x] Define the product scope and initial MVP.
- [x] Define the primary roles and authorization rules.
- [x] Define the ticket workflow and SLA behavior.
- [x] Draft the relational data model.
- [ ] Review the documentation for cross-file consistency.
- [ ] Initialize the Git repository and project skeleton.

### Phase 2 — Backend foundation

- [ ] Initialize Node.js, Express 5, and TypeScript.
- [ ] Configure environment variables and centralized error handling.
- [ ] Add authentication and server-side authorization.
- [ ] Configure Prisma and MySQL.
- [ ] Create the initial migrations and development seed data.
- [ ] Add a health-check endpoint.

### Phase 3 — Core ticket workflow

- [ ] Implement ticket creation and intake review.
- [ ] Implement Admin acceptance, adjustment, and rejection.
- [ ] Implement Agent claim requests and Admin assignment operations.
- [ ] Implement public replies and internal notes.
- [ ] Enforce valid ticket status transitions.
- [ ] Record ticket events, assignment history, and required notifications.

### Phase 4 — SLA and operational visibility

- [ ] Implement First Response SLA.
- [ ] Implement Resolution SLA pause/resume and reopening cycles.
- [ ] Implement SLA breach tracking.
- [ ] Build in-app notification views.
- [ ] Add basic operational dashboards and SLA reports.
- [ ] Document endpoints with OpenAPI and Swagger UI.

### Phase 5 — Frontend integration and verification

- [ ] Build the Customer, Agent, and Admin interfaces.
- [ ] Integrate the frontend with the REST API.
- [ ] Add unit and API/integration tests.
- [ ] Test concurrent assignment attempts and critical transaction failures.
- [ ] Add safe demo data and verify the full business workflow.
- [ ] Prepare a portfolio-ready deployment and demonstration guide.

A roadmap item should be marked complete only after the relevant code, verification, and documentation are in place.

---

## Project Documentation

The `docs/` directory is the main reference for the MVP's intended behavior.

| Document                                                               | Purpose                                                                                     |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| [`docs/01-project-scope.md`](docs/01-project-scope.md)                 | Product objectives, users, MVP features, acceptance criteria, and exclusions.               |
| [`docs/02-roles-and-permissions.md`](docs/02-roles-and-permissions.md) | Role capabilities, ticket access, message visibility, and server-side authorization rules.  |
| [`docs/03-workflow-and-sla.md`](docs/03-workflow-and-sla.md)           | Intake review, claim and assignment flow, allowed status transitions, and SLA calculations. |
| [`docs/04-data-model.md`](docs/04-data-model.md)                       | Database entities, relationships, constraints, indexes, and transactional requirements.     |

When a requirement changes, review all affected documents instead of changing only one file. In particular, changes to ticket statuses, assignment rules, SLA behavior, or data retention may affect the scope, permission model, workflow, and database schema simultaneously.

---

## Portfolio Objectives

This project is intended to demonstrate practical full-stack engineering skills through a coherent business application rather than a collection of disconnected CRUD screens.

The key outcomes to demonstrate are:

- Business rules translated into a consistent relational data model.
- Server-side authorization and data ownership enforcement.
- Transactional workflows for intake and assignment.
- SLA deadline calculations with pause/resume and reopening behavior.
- Reliable history and audit records.
- Clear API contracts and useful automated tests.
- Maintainable separation of frontend, backend, business services, and data access.
- A reproducible local setup and a demonstrable end-to-end workflow.

The primary measure of success is not the number of technologies used. It is whether the application correctly enforces the documented business rules, preserves data integrity, and explains the lifecycle of a ticket from intake through closure.
