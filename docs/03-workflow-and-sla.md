# Workflow and SLA

## 1. Purpose

This document defines the business workflows and Service Level Agreement (SLA) rules for the Helpdesk & SLA Ticketing System.

It covers:

- Ticket intake and Admin review.
- Customer-proposed Priority and SLA targets.
- Admin confirmation, adjustment, and rejection decisions.
- Agent claim requests and ticket assignment.
- Ticket processing status transitions.
- First Response SLA and Resolution SLA.
- SLA timing, pausing, deadlines, and breach tracking.
- Activity history, notifications, and data integrity.

The system separates **ticket intake review**, **ticket assignment**, and **ticket processing**. These are distinct stages and must not be treated as a single workflow.

## 2. Status Definitions

### 2.1. Ticket Intake Status

The intake status records whether a submitted ticket has been reviewed by an Admin.

| Status           | Description                                                                       |
| ---------------- | --------------------------------------------------------------------------------- |
| `PENDING_REVIEW` | The ticket has been submitted by a Customer and is waiting for an Admin decision. |
| `ACCEPTED`       | The Admin has accepted the ticket into the active processing workflow.            |
| `REJECTED`       | The Admin has rejected the ticket. The ticket does not enter normal processing.   |

The intake status is separate from the ticket processing status.

### 2.2. Ticket Processing Status

The processing status describes the progress of an accepted ticket.

| Status                 | Description                                                                                                               |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `OPEN`                 | The ticket has been accepted and is waiting to be picked up for processing. It may be unassigned or assigned to an Agent. |
| `IN_PROGRESS`          | An assigned Agent is actively handling the ticket.                                                                        |
| `WAITING_FOR_CUSTOMER` | The Agent needs additional information or action from the Customer before continuing.                                     |
| `RESOLVED`             | The issue has been solved, and the Agent has marked the ticket as resolved.                                               |
| `CLOSED`               | The ticket has been formally closed and its normal processing workflow is complete.                                       |

A ticket's processing status is not a substitute for its intake status or assignment state.

A ticket cannot enter normal processing before it has been accepted by an Admin.

### 2.3. Claim Request Status

An Agent may request to claim an accepted, unassigned ticket.

| Status      | Description                                                                                                                                                  |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `PENDING`   | The claim request is waiting for an Admin decision.                                                                                                          |
| `APPROVED`  | The Admin has approved the request and the ticket has been assigned to the requesting Agent.                                                                 |
| `REJECTED`  | The Admin has rejected the claim request.                                                                                                                    |
| `CANCELLED` | The request is no longer eligible for approval, for example because another valid assignment has already been completed or the ticket is no longer eligible. |

A claim request is not an assignment. An Agent must not begin handling a ticket merely because a claim request is pending.

## 3. Overall Ticket Workflow

The overall workflow consists of three separate stages:

1. **Intake review:** The Customer submits a ticket, and the Admin decides whether to accept or reject it. The Admin also confirms or adjusts the proposed Priority and SLA targets.
2. **Assignment:** After acceptance, the Admin can assign the ticket directly or approve an Agent's claim request.
3. **Processing:** The assigned Agent handles the ticket and updates its processing status until it is resolved and closed.

The high-level workflow is:

`Customer Submission → Admin Intake Review → Accepted or Rejected`

For an accepted ticket:

`Accepted → Assignment → Processing → Resolved → Closed`

For a rejected ticket:

`Pending Review → Rejected`

A rejected ticket does not proceed to assignment or normal processing.

## 4. Ticket Intake Review

### 4.1. Customer Submission

When creating a ticket, the Customer provides the information required by the ticket form, including:

- Ticket subject and description.
- Priority requested by the Customer.
- Proposed First Response SLA target.
- Proposed Resolution SLA target.
- Any other required ticket information.

The available Priority and SLA target options must be restricted to values supported by the company's configured policies. Customers must not be allowed to enter arbitrary SLA durations that bypass those policies.

After successful submission:

- The ticket's intake status is set to `PENDING_REVIEW`.
- The ticket is not yet eligible for normal processing or assignment.
- The submission timestamp is recorded.
- The requested Priority and SLA targets are recorded as the Customer's proposals.
- Processing SLA timers do not start.
- The time spent waiting for Admin review is tracked separately from processing SLA time.

The system must preserve the original Customer proposals even if the Admin later changes the applicable values.

### 4.2. Admin Review and Decision

The Admin reviews the submitted ticket and its proposed Priority and SLA targets.

The Admin may perform one of the following actions:

1. Accept the ticket without changing the proposed values.
2. Adjust one or more proposed values and accept the ticket using the confirmed values.
3. Reject the ticket with a mandatory rejection reason.

The Admin may adjust:

- Priority.
- First Response SLA target.
- Resolution SLA target.

Any adjustment must comply with the company's configured policies and supported options.

The system must distinguish between the Customer's proposed values and the final values confirmed by the Admin. The original proposals must not be overwritten or lost.

### 4.3. Confirmation and Adjustment Rules

If the Admin accepts the ticket without changes, the system records the final applicable values and communicates the acceptance decision to the Customer.

If the Admin adjusts any proposed value, the system must immediately communicate the decision to the Customer. The response must clearly show:

- The original value proposed by the Customer.
- The final value confirmed by the Admin.
- The reason for each adjustment.
- The Admin who made the decision.
- The decision timestamp.
- Whether the ticket has been accepted or rejected.

The system must not silently change the Customer's proposed Priority or SLA targets.

For an accepted ticket, the final confirmed values become the applicable Priority and SLA targets. A separate Customer approval step is not required in this workflow. However, the Customer must be able to view the final values and the Admin's explanation.

The system must persist the decision and final values before reporting the result to the Customer. If the decision cannot be saved successfully, the system must not report it as successfully completed.

### 4.4. Acceptance Rules

When an Admin accepts a ticket:

- Intake status changes from `PENDING_REVIEW` to `ACCEPTED`.
- Processing status changes to `OPEN`.
- The final Priority is saved.
- The final First Response SLA target is saved.
- The final Resolution SLA target is saved.
- The acceptance timestamp is recorded.
- The processing SLA start timestamp is recorded.
- The ticket becomes eligible for assignment.
- The Customer is notified of the acceptance and the final confirmed values.

The processing SLA starts from the recorded acceptance time, not from the original submission time.

If the Admin adjusted any proposed values, the acceptance notification must include those changes and their reasons.

### 4.5. Rejection Rules

When an Admin rejects a ticket:

- Intake status changes from `PENDING_REVIEW` to `REJECTED`.
- A rejection reason is mandatory.
- The decision maker and decision timestamp are recorded.
- The Customer is notified of the rejection and its reason.
- The ticket does not become eligible for assignment or normal processing.
- Processing SLA timers never start for that rejected ticket.

The system must not treat a rejected ticket as an accepted ticket with an ordinary processing status.

### 4.6. Intake Transition Rules

| Current Intake Status | Action                                    | New Intake Status                    | Processing SLA       |
| --------------------- | ----------------------------------------- | ------------------------------------ | -------------------- |
| Not created           | Customer submits ticket                   | `PENDING_REVIEW`                     | Not started          |
| `PENDING_REVIEW`      | Admin accepts without changes             | `ACCEPTED`                           | Starts at acceptance |
| `PENDING_REVIEW`      | Admin adjusts proposed values and accepts | `ACCEPTED`                           | Starts at acceptance |
| `PENDING_REVIEW`      | Admin rejects with a reason               | `REJECTED`                           | Never starts         |
| `ACCEPTED`            | Attempt to accept or reject intake again  | Not permitted in the normal workflow | Unchanged            |
| `REJECTED`            | Attempt to accept or reject intake again  | Not permitted in the normal workflow | Never started        |

Any later reopening or re-intake process for rejected tickets is outside the current MVP workflow and must not be implemented implicitly.

## 5. Agent Claiming and Ticket Assignment

Ticket assignment is separate from intake review. Only accepted tickets are eligible for assignment.

### 5.1. Agent Claim Request

An Agent may submit a claim request for an accepted, unassigned ticket.

When a claim request is created:

- The claim request status is set to `PENDING`.
- The request timestamp and requesting Agent are recorded.
- The ticket remains unassigned until an Admin approves the request.
- The Agent cannot begin processing the ticket based on the pending request alone.

The system must validate that the ticket is accepted, eligible for assignment, and not already assigned to another Agent when the request is submitted.

### 5.2. Admin Approval or Rejection

The Admin may approve or reject a pending claim request.

If approved:

- The claim request status changes to `APPROVED`.
- The ticket is assigned to the requesting Agent.
- The assignment timestamp and approving Admin are recorded.
- The Agent becomes authorized to process the ticket, subject to the ticket's current status and other permissions.

If rejected:

- The claim request status changes to `REJECTED`.
- The decision maker, timestamp, and any applicable explanation are recorded.
- The ticket remains unassigned unless it has been assigned through another valid action.

A pending claim request must never be treated as an approved assignment.

### 5.3. Direct Admin Assignment

The Admin may directly assign an accepted ticket to an eligible Agent without requiring a claim request.

The system must record:

- The assigned Agent.
- The assigning Admin.
- The assignment timestamp.
- Any relevant assignment or reassignment reason, where required by the configured rules.

The Admin may reassign an accepted ticket when necessary. Reassignment must update the current assignment without erasing the historical assignment records.

### 5.4. Assignment Integrity

The system must enforce the following rules:

- A ticket can have at most one current assigned Agent.
- Only accepted tickets may be assigned.
- A pending claim request does not grant processing permission.
- An Agent may process a ticket only when the Agent is currently assigned to it.
- A ticket that is closed or otherwise ineligible cannot receive a new assignment through ordinary workflow actions.
- Concurrent claim approvals and direct assignments must be handled safely.

For example, if two Agents request the same unassigned ticket, or an Admin assigns it while a claim request is pending, the system must prevent conflicting assignments.

If another valid assignment wins first, an outstanding claim request that is no longer eligible must be marked `CANCELLED` or otherwise closed according to the implemented rules. The system must not leave an apparently actionable pending request that could create a second assignment.

Assignment changes and claim decisions must be recorded in the activity history.

## 6. Ticket Processing Statuses

### 6.1. `OPEN`

The ticket has been accepted and is ready for handling.

It may remain unassigned while waiting for an Agent assignment. An Agent must be assigned before processing actions that require Agent ownership can be performed.

### 6.2. `IN_PROGRESS`

An assigned Agent is actively working on the issue.

The Agent may perform authorized processing actions, add public comments, add internal notes, and update the ticket as permitted by the role and workflow rules.

### 6.3. `WAITING_FOR_CUSTOMER`

The Agent needs information or action from the Customer to continue.

The Agent should provide a clear public explanation of what is required. The system records the status change and its timestamp.

Whether this status pauses the Resolution SLA is governed by the applicable SLA policy and the rules in Section 8.

### 6.4. `RESOLVED`

The issue has been solved, and the Agent has marked the ticket as resolved.

The system records the resolution timestamp and evaluates the Resolution SLA against the applicable deadline.

`RESOLVED` is not simply a status for waiting for further information. It means the Agent considers the reported issue solved.

### 6.5. `CLOSED`

The ticket has been formally closed and its normal processing workflow is complete.

The system records the closure timestamp. A closed ticket cannot be processed through ordinary status transitions in the MVP.

Reopening a closed ticket is outside the current MVP workflow unless a separate reopening policy is explicitly defined.

## 7. Processing Status Transitions

The normal processing transitions are:

| Current Status         | Allowed Action                                                        | New Status             | Main Rule                               |
| ---------------------- | --------------------------------------------------------------------- | ---------------------- | --------------------------------------- |
| `OPEN`                 | Assigned Agent starts handling the ticket                             | `IN_PROGRESS`          | Agent must be assigned                  |
| `IN_PROGRESS`          | Agent needs Customer input                                            | `WAITING_FOR_CUSTOMER` | A public explanation should be provided |
| `WAITING_FOR_CUSTOMER` | Customer provides the required information and the Agent resumes work | `IN_PROGRESS`          | Agent must remain authorized            |
| `IN_PROGRESS`          | Agent resolves the issue                                              | `RESOLVED`             | Resolution timestamp is recorded        |
| `RESOLVED`             | Authorized closure action                                             | `CLOSED`               | Closure timestamp is recorded           |

The following transitions are not allowed in the normal MVP workflow:

- `PENDING_REVIEW` directly to `IN_PROGRESS`.
- `PENDING_REVIEW` directly to `RESOLVED` or `CLOSED`.
- `REJECTED` to a processing status.
- `OPEN` to `RESOLVED` without following the permitted processing workflow.
- `CLOSED` back to an active processing status.

All status transitions must be validated by the backend. The frontend must not be trusted to enforce workflow rules by itself.

## 8. Service Level Agreement (SLA)

### 8.1. SLA Overview

The MVP supports two processing SLA targets:

1. **First Response SLA:** the target time for the first valid public response from the assigned Agent.
2. **Resolution SLA:** the target time for the issue to be resolved.

Priority and SLA targets are related but distinct:

- **Priority** indicates the urgency or importance of a ticket.
- **First Response SLA** defines the target time for the first qualifying Agent response.
- **Resolution SLA** defines the target time for resolving the issue.

A higher Priority may be associated with shorter SLA targets under company policy, but the system must validate the final combination against the configured rules.

### 8.2. Customer-Proposed SLA Targets

When submitting a ticket, the Customer selects a Priority and proposes First Response and Resolution SLA targets from the available company-supported options.

These values are proposals until the Admin completes intake review.

The system must retain:

- The Customer's original proposed Priority.
- The Customer's original proposed First Response SLA target.
- The Customer's original proposed Resolution SLA target.
- The final Priority and SLA targets confirmed by the Admin.
- Any adjustment reasons.
- The decision maker and decision timestamp.

The system must not allow the Customer to bypass company policy by submitting unsupported Priority values or SLA targets.

### 8.3. Admin Confirmation and Adjustment of SLA

During intake review, the Admin may confirm the Customer's proposed values or adjust one or more values to comply with company policy.

If an adjustment is made, the system must immediately notify the Customer of the changed value and the reason.

The final confirmed values, not the original proposals, govern SLA calculations after the ticket is accepted.

The system must not start processing SLA timers while the ticket is still `PENDING_REVIEW`.

### 8.4. When SLA Starts

The processing SLA starts only when the Admin accepts the ticket into the active processing workflow.

At acceptance, the system records the acceptance timestamp and uses it as the start time for the applicable processing SLA calculations.

Before acceptance:

- First Response SLA does not run.
- Resolution SLA does not run.
- Intake review waiting time is tracked separately.

If a ticket is rejected, its processing SLA never starts.

This separation prevents the time spent waiting for intake review from being incorrectly counted as processing time.

### 8.5. First Response SLA

The First Response SLA measures the time from ticket acceptance to the first qualifying public response by the assigned Agent.

A qualifying response must be a public comment or response that is visible to the Customer and constitutes a response to the ticket.

The following actions do not count as a qualifying first response by themselves:

- Internal notes.
- Assignment changes.
- Claim requests or claim decisions.
- Status changes without a qualifying public response.
- Automated system notifications.

The system records the first qualifying response timestamp and evaluates whether the target was met.

If no qualifying response is recorded before the deadline, the First Response SLA is breached according to the applicable policy.

### 8.6. Resolution SLA

The Resolution SLA measures the time from ticket acceptance to the time the issue is marked `RESOLVED`.

The system records the resolution timestamp and compares it with the applicable Resolution SLA deadline.

Changing the ticket to `WAITING_FOR_CUSTOMER` does not automatically resolve the issue.

Whether the waiting period pauses the Resolution SLA must be determined by the applicable SLA policy.

### 8.7. Pausing and Resuming the Resolution SLA

The MVP may support pausing the Resolution SLA while the ticket is in `WAITING_FOR_CUSTOMER`, according to the applicable policy.

If the policy enables pausing:

- The system records when the SLA pause begins.
- The elapsed paused duration is tracked.
- The SLA resumes when the ticket returns to `IN_PROGRESS`.
- The final deadline calculation accounts for the permitted paused duration.

If the policy does not enable pausing, the Resolution SLA continues to run while the ticket is waiting for the Customer.

The implementation must apply one consistent, explicit rule per applicable policy. It must not silently pause or extend an SLA.

The First Response SLA must not be incorrectly reset or restarted by ordinary status changes.

### 8.8. SLA Deadlines and Breaches

For each accepted ticket, the system must be able to determine:

- The applicable SLA target.
- The SLA start timestamp.
- The calculated deadline.
- The relevant qualifying event timestamp, if completed.
- Whether the target was met or breached.
- Any valid paused intervals, if supported by the policy.

SLA calculations must use a consistent time standard, such as UTC for persisted timestamps, while displaying times in the user's appropriate timezone.

The system must avoid inconsistent calculations caused by frontend clocks, duplicated events, or concurrent updates.

### 8.9. SLA Policy Management

The Admin manages the company-supported SLA options and policies.

The policy configuration must define valid Priority and SLA target combinations and the rules used to calculate deadlines.

At minimum, the system must validate the Customer's proposals and the Admin's final selections against the configured options.

Advanced SLA features, including business-hour calendars, public holidays, regional calendars, and more complex working-time rules, are future enhancements and are not required for the MVP.

## 9. Activity History and Notifications

The system must maintain an auditable activity history for significant workflow actions, including:

- Ticket creation and submission timestamp.
- Customer-proposed Priority and SLA targets.
- Intake review decisions.
- Admin adjustments to Priority or SLA targets and their reasons.
- Acceptance or rejection timestamps and decision makers.
- Customer notifications about intake decisions and adjustments.
- Claim requests and their decisions.
- Direct assignments and reassignments.
- Processing status changes.
- Public comments and internal notes, with appropriate access controls.
- First qualifying response timestamp.
- Resolution and closure timestamps.
- SLA pause/resume events, if supported.
- SLA deadline and breach information.

Activity records must identify the actor, action, timestamp, and relevant changed values where applicable.

Customer-visible notifications and internal activity records are different concepts. Internal notes and restricted administrative details must not be exposed to Customers.

For the MVP, intake decisions and Priority/SLA adjustments must be communicated through in-app notifications. Email notifications are a future enhancement.

The system must not report a decision as successfully completed if the underlying decision was not persisted.

## 10. Data Integrity and Error Handling

The backend must enforce workflow and SLA rules even if a client sends invalid or unauthorized requests.

Important integrity rules include:

- Only an Admin may accept or reject a submitted ticket.
- A rejection requires a reason.
- Only accepted tickets may be assigned.
- A pending claim request does not authorize ticket processing.
- Only one Agent may be currently assigned to a ticket.
- Only an authorized assigned Agent may perform restricted processing actions.
- Customer-proposed values and Admin-confirmed values must remain distinguishable.
- Admin adjustments must have a recorded reason and must be communicated to the Customer.
- Processing SLA timers must not start before acceptance.
- Rejected tickets must not receive processing SLA deadlines.
- SLA calculations must use the final confirmed targets.
- Ticket status transitions must follow the defined workflow.
- Significant decisions and changes must be auditable.

Assignment and claim approval operations must be protected against concurrent requests. The database transaction or equivalent consistency mechanism must ensure that two conflicting operations cannot create multiple current assignments.

If an operation fails, the system must return an appropriate error and preserve data consistency. It must not partially apply an intake decision, assignment, or SLA change.

## 11. Important Test Cases

### 11.1. Ticket Intake

- A Customer submits a ticket with supported Priority and SLA proposals.
- A submitted ticket is created with intake status `PENDING_REVIEW`.
- Processing SLA timers remain inactive before acceptance.
- An Admin accepts a ticket without changing the proposed values.
- An Admin adjusts Priority only and accepts the ticket.
- An Admin adjusts First Response SLA only and accepts the ticket.
- An Admin adjusts Resolution SLA only and accepts the ticket.
- An Admin adjusts multiple values and accepts the ticket.
- Each adjustment records the original value, final value, reason, actor, and timestamp.
- Each adjustment is communicated immediately to the Customer.
- An Admin rejects a ticket without a reason; the operation is refused.
- An Admin rejects a ticket with a reason; the Customer is notified.
- A rejected ticket cannot be assigned or enter normal processing.
- A rejected ticket never starts processing SLA timers.
- An unauthorized user attempts to accept or reject a ticket; the request is refused.

### 11.2. Assignment

- An Agent requests to claim an accepted, unassigned ticket.
- The Agent cannot process the ticket while the claim request is pending.
- An Admin approves a claim request and the ticket is assigned to the requesting Agent.
- An Admin rejects a claim request and the ticket remains unassigned unless another valid assignment exists.
- An Admin directly assigns an accepted ticket.
- An Admin reassigns a ticket and the assignment history is preserved.
- Two Agents submit claim requests for the same ticket.
- A direct assignment conflicts with a pending claim request.
- Concurrent operations cannot create more than one current assigned Agent.
- A claim request that becomes ineligible is no longer actionable.

### 11.3. Processing Status

- An assigned Agent moves a ticket from `OPEN` to `IN_PROGRESS`.
- An Agent moves a ticket to `WAITING_FOR_CUSTOMER`.
- An Agent resumes work and moves the ticket back to `IN_PROGRESS`.
- An Agent resolves the issue and the resolution timestamp is recorded.
- An authorized closure action moves the ticket from `RESOLVED` to `CLOSED`.
- Invalid status transitions are refused by the backend.
- An unauthorized or unassigned Agent cannot perform restricted processing actions.

### 11.4. SLA

- First Response SLA starts at acceptance, not submission.
- Resolution SLA starts at acceptance, not submission.
- SLA targets use the final values confirmed by the Admin.
- An internal note does not count as the first public response.
- A status change alone does not count as the first public response.
- A qualifying public Agent response records the First Response SLA event.
- Resolution records the correct resolution timestamp.
- A ticket that exceeds its First Response SLA is marked as breached according to policy.
- A ticket that exceeds its Resolution SLA is marked as breached according to policy.
- Resolution SLA pause/resume behavior follows the applicable policy.
- Rejected tickets do not receive active processing SLA deadlines.
- SLA calculations remain consistent under concurrent updates.

### 11.5. Access Control and Auditability

- A Customer can view only their own tickets and permitted ticket information.
- A Customer cannot view internal notes or restricted administrative data.
- A Customer can see the final Priority/SLA values and explanations for adjustments to their own ticket.
- An Agent cannot approve their own claim request unless a separate explicit policy permits it.
- An Agent cannot accept or reject intake decisions.
- An Agent cannot change confirmed SLA targets outside the permitted workflow.
- Significant decisions and changes are recorded with actor and timestamp.
- Failed operations do not leave partially applied decisions or inconsistent SLA records.

## 12. Related Documentation

- `docs/01-project-scope.md`
- `docs/02-roles-and-permissions.md`
- `docs/04-database-design.md` (when created)
- `docs/05-api-design.md` (when created)
