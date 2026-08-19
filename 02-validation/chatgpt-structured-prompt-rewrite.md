# CRM Rep Workflow Prototype

## CONTEXT

### The Internal Tool Nobody Uses

**User voice:**

* "It's faster to keep my deals in a spreadsheet than to fight the CRM's eight required fields." — Account Executive, 2 logins/month
* "Logging activity feels like data entry for management, not something that helps me sell." — Senior AE
* "If it could just tell me who to call next, I'd open it every morning." — SDR, occasional user

The product should reduce CRM maintenance work and make the rep's immediate priorities obvious.

## REFERENCE

Create a rep-facing workspace with two primary areas:

**Today's Actions**
Show meetings, calls, emails, follow-ups, and other actions that are scheduled or due today.

Reps can:

* Complete an action
* Reschedule it
* Edit it
* Dismiss it
* Launch the relevant workflow when appropriate

**Opportunity Updates**
Show meaningful changes to existing opportunities and newly identified opportunities.

For existing opportunities, show:

* Current value
* Proposed value, when explicitly supported by the source
* Current phase
* Proposed phase
* Current next action
* Proposed next action
* Source and timestamp
* Verbatim source evidence supporting the proposed change

For new opportunities, clearly distinguish them from updates to existing opportunities and provide a separate review and acceptance flow.

## CONSTRAINTS

Source evidence must be displayed verbatim. Do not summarize, rewrite, or alter the quoted source content.

Structured proposed updates may be extracted from source evidence, but must not introduce information that is not explicitly supported by the source.

Only surface an opportunity update when the source explicitly indicates a meaningful change to at least one of:

* Opportunity value
* Phase
* Next action

Use **phase** consistently throughout the product. The only valid phases are:

* New
* Discovery
* Qualify
* Negotiate
* Closed

Opportunity values must come from an explicitly stated numeric value in the source. Do not estimate, calculate, or infer opportunity value. Preserve the stated currency when available. If no numeric value is stated, do not propose a value change.

Clearly distinguish:

* Source evidence
* System-suggested changes
* Rep-entered changes
* Current opportunity record

All system-suggested changes remain pending until the rep reviews them.

When a rep **accepts** a suggested change, update the opportunity record.

When a rep **rejects** a suggested change, dismiss the suggestion without changing the opportunity record.

## OUTPUT

Produce a responsive, fully clickable, functional webpage that demonstrates the complete rep workflow for:

* Reviewing today's actions
* Completing, editing, dismissing, and rescheduling actions
* Reviewing existing-opportunity updates
* Reviewing newly identified opportunities
* Accepting or rejecting suggested changes
* Directly editing opportunity values, phases, and next actions
* Seeing source provenance and verbatim supporting evidence

Use realistic seeded data.

Persist changes within the prototype session so accepted updates, edits, completed actions, and rejected suggestions visibly change the interface.

Every visible control must work. Do not include decorative buttons, menus, filters, or actions that have no behavior.

The result should be responsive and ready to share via a public link.
