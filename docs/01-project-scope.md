# 01 — Project Scope

## 1. Problem Statement

IT support teams need a centralized system to receive requests, determine priority, assign ownership, communicate with requesters, and track response and resolution deadlines.

The system must make it clear:

- Who submitted a request and who is responsible for handling it.
- Whether a ticket is awaiting intake review or is being processed.
- Which priority and SLA targets currently apply.
- Whether a ticket has been resolved or closed.
- Which important decisions and changes occurred during the ticket lifecycle.

In particular, the system must distinguish the time a ticket spends waiting for Admin review from the time spent on SLA-governed processing. These two periods must not be combined.

## 2. Product Objectives

Build a **Helpdesk & SLA Ticketing System for a single workspace**, focusing on controlled support workflows, clear access permissions, SLA tracking, and auditable activity history.

The MVP aims to demonstrate the following technical and business capabilities:

- Designing a relational database that supports real-world workflows.
- Enforcing access control in the backend.
- Managing ticket statuses through explicit transition rules.
- Separating ticket intake review, Agent assignment, and ticket processing.
- Calculating SLA deadlines using confirmed targets.
- Recording important decisions and changes so they can be reviewed and explained when necessary.

## 3. Users and Roles

### 3.1. Customer

Customers can:

- Create tickets and track their own tickets.
- Propose a Priority when creating a ticket.
- Select First Response SLA and Resolution SLA targets from company-supported options.
- View the Admin's intake decision, including the final values applied and the reasons for any adjustments.
- Add public comments to their own tickets according to workflow rules.
- View permitted notifications and SLA information.

Customers cannot approve their own tickets, assign Agents, or directly modify confirmed Priority/SLA values.

### 3.2. Agent

Agents can:

- View tickets in the workspace according to the access policy.
- Submit a claim request for an accepted ticket that has no assigned Agent.
- Process tickets after being assigned directly by an Admin or after their claim request has been approved.
- Submit public replies and add internal notes.
- Perform permitted status transitions.
- View SLA information and activity history relevant to their work.

Agents cannot approve claim requests, accept or reject new tickets during intake review, or modify company-wide SLA policies.

### 3.3. Admin

Admins are responsible for:

- Managing users and roles.
- Managing supported Priority options, SLA targets, and applicable policies.
- Reviewing new tickets and deciding whether to accept or reject them.
- Confirming or adjusting the Priority, First Response SLA, and Resolution SLA proposed by Customers.
- Providing a reason when adjusting Customer-proposed values.
- Immediately notifying Customers of intake decisions and adjustments.
- Approving or rejecting Agent claim requests.
- Assigning and reassigning Agents.
- Monitoring dashboards, reports, and SLA performance.

Admins have the operational permissions required to manage tickets, but all actions must follow the workflow and be recorded according to the audit rules.

## 4. MVP Features

The MVP is the minimum usable, testable, and demonstrable version of the system.

### 4.1. Authentication and User Management

- Authentication for Customer, Agent, and Admin roles.
- User and role management by authorized Admins.
- Backend access control.

### 4.2. Ticket Creation and Management

- Create tickets and view ticket lists and details.
- Search, filter, sort, and paginate ticket lists.
- Track intake status, processing status, and assigned Agent.
- Support public replies and internal notes with different visibility rules.
- Maintain activity history and important ticket events.

### 4.3. Ticket Intake and Review

When a Customer submits a ticket:

- The ticket is recorded with intake status `PENDING_REVIEW`.
- The Customer provides a proposed Priority, First Response SLA target, and Resolution SLA target.
- All selections must be within the options supported by the company.
- The ticket is not yet assigned and does not enter normal processing.
- Processing SLA timers have not started; intake review waiting time is tracked separately.

The Admin reviews the ticket and its proposals and may:

- Accept the ticket without changing the proposed values.
- Adjust one or more Priority/SLA values and then accept the ticket.
- Reject the ticket and provide a mandatory reason.

If the Admin adjusts Priority or SLA, the system must preserve the Customer-proposed values, the final values, the reason, the actor, and the timestamp. The Customer must be notified immediately of the changes and the intake decision. The system must not silently overwrite the original proposals.

When a ticket is accepted:

- Intake status changes to `ACCEPTED`.
- Processing status becomes `OPEN`.
- The final applicable Priority and SLA targets are saved.
- Processing SLA timing starts at the acceptance timestamp.
- The ticket becomes eligible for assignment.

When a ticket is rejected:

- Intake status changes to `REJECTED`.
- The rejection reason and decision details are recorded.
- The Customer is notified.
- The ticket does not enter assignment or normal processing.
- Processing SLA timing never starts for that ticket.

### 4.4. Agent Claim Requests and Assignment

- Agents can submit claim requests for accepted, unassigned tickets.
- A claim request has its own status and does not mean the ticket has already been assigned.
- Admins can approve or reject claim requests.
- Admins can also assign or reassign tickets directly without a claim request.
- Each ticket can have at most one currently assigned Agent.
- The backend must safely handle competing requests to prevent conflicting assignments.

### 4.5. Ticket Processing Statuses

The MVP supports the following processing statuses:

- `OPEN`
- `IN_PROGRESS`
- `WAITING_FOR_CUSTOMER`
- `RESOLVED`
- `CLOSED`

Only tickets accepted by an Admin can enter the processing workflow. Every status transition must be valid under the workflow and enforced by the backend.

`RESOLVED` means the issue has been solved and the Agent has marked the ticket as resolved. `CLOSED` means the ticket has been formally closed and its normal processing workflow is complete.

The MVP does not implicitly introduce a separate testing or approval stage between `RESOLVED` and `CLOSED`.

### 4.6. SLA and Deadline Tracking

The MVP supports two SLA targets:

- **First Response SLA:** the target time for the first qualifying public response from the assigned Agent.
- **Resolution SLA:** the target time for resolving the issue, ending when the ticket transitions to `RESOLVED`.

Customers propose both SLA targets when creating a ticket. Admins confirm or adjust them according to the company's supported options and policies.

Processing SLA timing starts only when the Admin accepts the ticket and confirms the final values. Time spent waiting for Admin review is measured separately.

The MVP uses a 24/7 clock. Resolution SLA timing pauses while the ticket is in `WAITING_FOR_CUSTOMER` and resumes when it returns to `IN_PROGRESS`. The system must record pause intervals so deadlines are calculated consistently.

The system must calculate deadlines, record the first-response and resolution events, and determine whether each SLA target was met or breached.

### 4.7. Notifications and Audit Trail

- In-app notifications for important ticket events.
- Immediate notifications when an Admin accepts or rejects a ticket or adjusts Priority/SLA values.
- Activity history for decisions, adjustments, assignments, status changes, and important SLA events.
- Clear separation between public content and internal notes.
- Consistent updates to business data and related activity history when operations succeed or fail.

### 4.8. Dashboard, Reporting, and Testing

- Basic dashboards for ticket status and workload.
- SLA summary reports.
- Automated tests for permissions, workflows, intake review, assignment, content visibility, and SLA calculations.
- API documentation using OpenAPI/Swagger UI.
- Setup instructions and demo data/accounts that contain no real sensitive information.

## 5. Acceptance Criteria

The MVP is considered acceptable when:

- A Customer cannot access another Customer's ticket, including by changing an ID in an API request.
- The backend enforces permissions; the frontend is not the only protection layer.
- A newly submitted ticket always enters `PENDING_REVIEW` before an Admin decision.
- Only an Admin can accept or reject a ticket.
- Rejecting a ticket requires a reason.
- Customers can propose a Priority and two SLA targets from valid options.
- Admins can confirm or adjust those proposals according to company policy.
- Every adjustment records the original value, final value, reason, actor, and timestamp.
- Customers are immediately notified of adjustments and the intake decision.
- Processing SLA timing starts only at ticket acceptance; intake review waiting time is tracked separately.
- Rejected tickets cannot be assigned, do not enter normal processing, and never start processing SLA timers.
- An Agent cannot process a ticket merely because a claim request is pending.
- Each ticket has at most one currently assigned Agent.
- First Response SLA is satisfied only by a qualifying public response from the assigned Agent, not by an internal note or status change.
- Resolution SLA ends when the ticket transitions to `RESOLVED` and follows the defined pause/resume rules.
- Invalid status transitions are rejected by the backend.
- Important decisions and changes are recorded in the activity history.
- The application includes setup instructions and can be demonstrated with sample data.

## 6. Features Outside the MVP

The following features are planned for later development after the core workflow is stable:

- SaaS multi-tenancy or multiple independent organizations.
- Microservices and complex orchestration infrastructure.
- Redis caching.
- Background workers using BullMQ/Redis.
- Real-time updates using WebSockets.
- Email/SMS and integrations with Slack, WhatsApp, or external systems.
- Automated email notifications.
- File attachments.
- AI-based ticket classification or chatbots.
- Knowledge Base, surveys, live chat, and omnichannel workflows.
- Business-hour calendars, holidays, and advanced working-time SLA rules.

These are planned enhancements, not limitations imposed by a particular development timeline.

## 7. Portfolio Value

The project focuses on demonstrating technical capabilities that can be verified through code and a working demo:

- Backend authorization and data ownership protection.
- Explicit ticket workflows and valid status transitions.
- Separation of intake review, assignment, and processing.
- Auditable proposal, confirmation, and adjustment of Priority/SLA values.
- First Response SLA and Resolution SLA calculations.
- Consistent SLA pause/resume rules.
- Reliable data transactions and activity history.
- Automated testing and API documentation.
- A runnable or deployable demo with safe sample data.
