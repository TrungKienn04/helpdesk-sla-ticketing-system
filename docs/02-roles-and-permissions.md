# 02 — Roles and Permissions

## 1. Authorization Principles

### 1.1. System Scope

The MVP supports a single workspace. The permissions defined in this document apply to that workspace and do not cover independent organizations or multi-tenant isolation.

### 1.2. Backend Authorization

Every operation must be authorized by the backend. Hiding buttons or blocking pages in the frontend improves user experience but does not replace access control in the API and business logic layers.

The system follows the **deny-by-default principle**: if an action is not explicitly permitted by the user's role, ticket access rights, and workflow conditions, the backend must reject it.

### 1.3. Role Permissions and Workflow State

Permission to perform an action depends on:

- The user's role.
- The user's access rights for the specific ticket.
- The ticket's intake status.
- The ticket's processing status.
- The Agent assignment state.
- The business conditions for the requested action.

Having a role does not automatically allow a user to perform every action on every ticket.

### 1.4. Customer Data Protection

- Customers may access only tickets they submitted.
- Customers cannot view internal notes or staff-only data.
- Public replies are visible to Customers who have access to the ticket.
- Internal notes can be viewed only by Agents and Admins with appropriate access.
- Activity history must be filtered by permission; access to a ticket does not grant access to all internal events.

## 2. Roles

### 2.1. Customer

A Customer is a person who submits a support request.

Customers are allowed to:

- Create tickets for themselves.
- Propose a Priority when creating a ticket.
- Select First Response SLA and Resolution SLA targets from valid system-provided options.
- View intake status, processing status, and permitted information for their own tickets.
- View the final Priority/SLA values applied.
- View the reasons for Admin adjustments or ticket rejection.
- Add public comments to their own tickets when permitted by the workflow.
- View their own in-app notifications.

Customers are not allowed to:

- Create tickets on behalf of another user by submitting an arbitrary `requesterId`.
- Accept or reject their own tickets.
- Assign Agents.
- Directly change processing status.
- Directly change Priority or SLA values that have already been confirmed by an Admin.
- View internal notes, administrative data, or other Customers' tickets.

### 2.2. Agent

An Agent is a support staff member responsible for handling assigned tickets.

Agents are allowed to:

- View tickets in the workspace according to the access policy.
- View unassigned tickets if the visibility policy permits it.
- Submit a claim request for an accepted ticket that has no assigned Agent.
- View the status of their own claim requests.
- Process tickets after being validly assigned.
- Add public replies and internal notes.
- Perform valid status transitions on tickets they are authorized to handle.
- View SLA information and activity history relevant to their work.
- Receive and view their own in-app notifications.

Agents are not allowed to:

- Approve or reject their own claim requests.
- Assign tickets to themselves or other Agents by bypassing the workflow.
- Process a ticket merely because a claim request is in `PENDING` status.
- Accept or reject tickets during intake review.
- Independently change Priority or SLA targets confirmed by an Admin through the intake workflow.
- Modify company-wide SLA policy configuration.
- Manage users or roles.

### 2.3. Admin

Admins are responsible for administration and workflow coordination.

Admins are allowed to:

- View tickets in the workspace.
- Accept or reject tickets awaiting intake review.
- View the Priority, First Response SLA, and Resolution SLA proposed by a Customer.
- Confirm proposals or adjust one or more values according to company policy.
- Provide a mandatory reason for each Priority/SLA adjustment.
- Confirm the intake decision and immediately notify the Customer.
- Reject tickets with a mandatory reason and notify the Customer.
- Approve or reject Agent claim requests.
- Assign and reassign Agents directly.
- Perform permitted operational actions under the workflow.
- Manage users and roles.
- Manage supported Priority options and SLA policies.
- View workspace dashboards, reports, and audit history.

Administrative permissions do not allow an Admin to bypass data-integrity requirements. Important decisions, especially Priority/SLA adjustments, must be recorded and communicated according to the workflow.

## 3. Permission Matrix

| Capability | Customer | Agent | Admin |
|---|---|---|---|
| Sign in | Yes | Yes | Yes |
| Create a ticket for themselves | Yes | No in the MVP | Yes |
| View their own tickets | Yes | If they are the requester or permitted by policy | Yes |
| View workspace tickets according to policy | No | Yes | Yes |
| Propose Priority when creating a ticket | Yes | No | May set valid values when creating a ticket |
| Propose First Response SLA and Resolution SLA when creating a ticket | Yes | No | May set valid values when creating a ticket |
| Accept or reject a new ticket | No | No | Yes |
| Confirm or adjust proposed Priority/SLA during intake review | No | No | Yes |
| Provide reasons for Priority/SLA adjustments | No | No | Yes; required when adjusting values |
| View intake decisions and adjustment reasons | Own tickets only | According to access rights | Yes |
| View ticket rejection reasons | Own tickets only | According to access rights | Yes |
| Submit a claim request for an unassigned ticket | No | Yes, if eligible | Not required |
| Approve or reject claim requests | No | No | Yes |
| Assign or reassign Agents | No | No | Yes |
| Directly change confirmed Priority/SLA values | No | No | Only through permitted workflows, with audit records |
| Change processing status | No direct changes | Yes, if assigned and permitted by the workflow | Yes, as permitted by the workflow |
| Add public replies | Own tickets only | Tickets they are authorized to handle | Tickets they are authorized to handle |
| Add internal notes | No | Yes, within access rights | Yes |
| Read internal notes | No | Yes, within access rights | Yes |
| View ticket activity history | Permitted events for own tickets only | According to access rights | Yes |
| View their own in-app notifications | Yes | Yes | Yes |
| Manage users and roles | No | No | Yes |
| Manage SLA policies and supported options | No | No | Yes |
| View workspace dashboards/reports | No | Limited operational reports | Yes |

This matrix defines general permissions. Ticket ownership, status conditions, and workflow requirements must still be checked for each operation.

## 4. Workflow-Specific Authorization Rules

### 4.1. When a Customer Creates a Ticket

- A Customer may create tickets only for themselves.
- A Customer may submit a proposed Priority, First Response SLA target, and Resolution SLA target from valid system-provided options.
- The backend must validate all submitted values against the supported options and policies.
- After successful submission, the ticket's intake status is `PENDING_REVIEW`.
- A Customer cannot set the intake status, processing status, SLA start timestamp, or SLA deadline.
- A Customer cannot change submitted proposals through the ordinary ticket-update API after submission. Adjustments during intake review must be made by an Admin through the defined workflow.

### 4.2. When an Admin Reviews a Ticket

Only an Admin may accept or reject a ticket.

If the Admin accepts the ticket:

- Intake status changes to `ACCEPTED`.
- Processing status starts at `OPEN`.
- The final Priority and SLA values are saved.
- Processing SLA timing starts at acceptance.
- The ticket becomes eligible for assignment.

If the Admin adjusts Priority or SLA:

- A reason is mandatory.
- The original Customer-proposed value must be preserved for comparison.
- The final value must be stored separately or through an equivalent auditable change history.
- The system must immediately notify the Customer of the old value, the new value, and the reason.
- The system must record the actor and adjustment timestamp.

If the Admin rejects the ticket:

- A rejection reason is mandatory.
- The ticket's intake status changes to `REJECTED`.
- The Customer is notified of the decision and reason.
- The ticket cannot be assigned or enter normal processing.
- Processing SLA timing does not start.

The Admin must not leave the Customer with the impression that the original proposed values remain official after they have been adjusted.

### 4.3. When an Agent Requests a Ticket

- An Agent may submit a claim request only for an accepted ticket that has no assigned Agent.
- The claim request is stored with its own status, initially `PENDING`.
- The Agent cannot process the ticket until the Admin approves the request and the assignment is successfully recorded.
- Only an Admin may approve or reject the claim request.
- If the ticket is assigned through another operation, any pending claim request must be handled so that it cannot create a conflicting assignment.

### 4.4. When an Admin Assigns a Ticket Directly

- An Admin may assign an accepted ticket without a claim request.
- An Admin may reassign a ticket according to the operational policy.
- A ticket may have at most one currently assigned Agent.
- Assignments, reassignments, and the acting user must be recorded in the audit history.
- Concurrent operations must not create two current assignees.

### 4.5. When an Agent Processes a Ticket

- An Agent must be validly assigned before performing processing actions that require ownership.
- An Agent may perform only status transitions permitted by the workflow.
- An Agent may add public replies and internal notes within their permissions.
- Internal notes must not be exposed to Customers.
- An Agent must not change confirmed Priority/SLA values by bypassing the intake workflow.
- The First Response SLA event may be recorded only when a qualifying public reply is submitted by the assigned Agent.

## 5. Data Access Rules

### 5.1. Customer

The backend must verify ticket ownership using the authenticated user, not an arbitrary `requesterId` supplied by the client.

Customers may receive:

- Information about their own tickets.
- Public replies.
- The final applicable Priority and SLA values.
- Permitted information about their original proposals and subsequent adjustments.
- Public events and relevant notifications.
- SLA information that is permitted to be displayed.

Customers must not receive:

- Internal notes.
- Other Customers' tickets.
- Authentication secrets or administrative data.
- Internal audit details outside the Customer-visible scope.
- Private information about Agents or other users that is not necessary for tracking the ticket.

### 5.2. Agent

Agents may view the data needed to perform their work within the workspace and according to the access policy.

Agents may view internal notes and operational history for tickets they are authorized to access. Permission to view a ticket does not automatically grant permission to perform every action on that ticket.

### 5.3. Admin

Admins may access administrative information and audit history within their responsibilities. Intake decisions, adjustments, rejections, assignments, and workflow changes must be recorded for review.

## 6. Public Replies, Internal Notes, and Activity History

### 6.1. Public Replies

Public replies may be viewed by Customers who have access to the ticket.

Customers may add comments only to their own tickets and must follow status restrictions. For example, the MVP does not allow new comments on a ticket that is already `CLOSED`.

### 6.2. Internal Notes

Internal notes are restricted to Agents and Admins with appropriate access. APIs must protect both the note content and related fields so that internal information cannot leak through responses or activity history.

### 6.3. Activity History

Important events must record the actor, timestamp, action type, and relevant changed values.

These events include:

- Ticket creation and submission.
- Original Priority/SLA proposals.
- Acceptance or rejection decisions.
- Priority/SLA adjustments and reasons.
- Assignment, reassignment, and claim request decisions.
- Status changes.
- Public replies and internal notes.
- SLA start, first response, SLA pause/resume, resolution, and closure events.

Customers may view only the history permitted to them. They must not access the complete internal audit log.

## 7. SLA and Notification Rules

- Customers may propose Priority and two SLA targets when creating a ticket.
- Admins confirm or adjust these values according to company policy.
- Every adjustment must have a reason and be communicated immediately to the Customer.
- Processing SLA timing starts only after the Admin accepts the ticket and the final values are confirmed.
- Intake review waiting time is tracked separately.
- Rejected tickets do not start processing SLA timers.
- First Response SLA is satisfied by the first qualifying public reply from the assigned Agent; internal notes do not count.
- Resolution SLA ends when the ticket transitions to `RESOLVED`.
- In the MVP, Resolution SLA pauses while the ticket is in `WAITING_FOR_CUSTOMER` and resumes when it returns to `IN_PROGRESS`.
- In-app notifications are used for intake decisions and Priority/SLA adjustments. Email notifications are a future enhancement and are not part of the MVP.

## 8. Error Handling Rules

The backend should return consistent errors so that clients can handle failures correctly.

| Scenario | Recommended HTTP Status |
|---|---|
| Unauthenticated request | `401 Unauthorized` |
| Authenticated user lacks the required permission | `403 Forbidden` |
| Customer requests a ticket they are not allowed to access | `404 Not Found` |
| Invalid input data | `400 Bad Request` or `422 Unprocessable Content`, according to the API convention selected |
| Status transition or operation conflicts with the current state | `409 Conflict` |

The system must not return a successful result when the business operation has not been persisted successfully.

## 9. Implementation and Testing Principles

- Use reusable middleware or authorization services, while still checking ticket ownership and workflow conditions in the relevant business service.
- Do not trust client-supplied `requesterId`, role, intake status, processing status, assignee, SLA timestamps, or deadlines.
- Validate Priority and SLA targets against the permitted options and policies in the backend.
- Persist intake decisions, before/after values, adjustment reasons, and audit events consistently.
- Ensure claim approval and assignment operations are safe under concurrent requests.
- Test that a Customer cannot access another Customer's ticket.
- Test that an Agent cannot process a ticket before being assigned.
- Test that an Admin cannot adjust Priority/SLA without a reason.
- Test that the Customer receives a notification about adjustments.
- Test that SLA timing does not start before ticket acceptance.
- Test that rejected tickets cannot be assigned and do not start processing SLA timers.

## 10. Related Documentation

- `docs/01-project-scope.md`
- `docs/03-workflow-and-sla.md`
- `docs/04-data-model.md` (when completed)
- `docs/05-api-design.md` (when completed)
