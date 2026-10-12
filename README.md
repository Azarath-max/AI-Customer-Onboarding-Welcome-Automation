# AI Customer Onboarding Welcome Automation

An n8n workflow that sends a personalized onboarding welcome email when a Salesforce Opportunity is marked **Closed Won** and needs implementation. Salesforce supplies the data, fixed rules make the decisions, Gemini only drafts the wording, and a human approves in Slack before anything is sent.

> Built as a portfolio project with fictional data in a Salesforce Developer Edition org. It is a prototype, not a production system.

## The problem

After a deal closes, an Account Manager or CSM has to look up the customer, products, main contact and onboarding owner in Salesforce, then write a welcome email by hand. That leads to late or missed emails, inconsistent messaging, repeated manual work, and mistakes from incomplete data.

## What the workflow does

1. Detects a Salesforce Opportunity that changed and loads its full record.
2. Continues only if Stage = **Closed Won** and **Implementation Required** is true.
3. Looks up the main contact and account.
4. Validates required fields (opportunity, account, contact, contact email, products, CSM).
5. Stops if Onboarding Status is already `Welcome Email Sent`.
6. Gemini drafts the email body from Salesforce data only. Code checks the draft.
7. Posts the draft to Slack and waits for **Approve** or **Disapprove**.
8. On approval, Gmail sends the email and Salesforce is updated to `Welcome Email Sent` with a timestamp.
9. Any failure ends in a Slack exception message. A Gmail failure also sets `Email Failed` in Salesforce.

## Tools

| Tool | Role |
|---|---|
| Salesforce | Source of truth for customer and deal data |
| n8n | Orchestration |
| Google Gemini | Drafts the email body (nothing else) |
| Slack | Human approval and exception alerts |
| Gmail | Sends the approved email |

No RAG, AI agent, vector database or extra database. Logging uses n8n's execution history, Slack messages and the Salesforce status fields.

## Workflow (18 nodes: 15 on the happy path, 3 for exceptions)

```
Opportunity Updated (Salesforce Trigger, polls every minute)
  -> Get Opportunity
  -> Closed Won Check
  -> Implementation Required Check
  -> Get Contact and Account (one SOQL query)
  -> Validate Required Data (Code)
  -> Data Valid?
  -> Not Already Sent?
  -> Draft Welcome Email (Gemini)
  -> Check AI Draft (Code)
  -> AI Draft OK?
  -> Slack Approval Request (send and wait)
  -> Approval Check
  -> Send Welcome Email (Gmail)
  -> Mark Welcome Email Sent (Salesforce)

Exceptions -> Build Exception Report (Code) -> Notify Exception (Slack)
Gmail error output -> Mark Email Failed (Salesforce) -> Build Exception Report
```

The trigger only returns a partial record, so `Get Opportunity` loads the full one. Slack's send-and-wait operation covers both the approval request and the wait in one node.

## Where decisions are made

| Decision | Made by |
|---|---|
| Is it Closed Won? Is implementation required? | n8n IF nodes on Salesforce fields |
| Is required data present? | Code node with fixed rules |
| Was the email already sent? | n8n IF node on Onboarding Status |
| Does the AI draft mention the contact, CSM and every product, and avoid amounts, percentages and links? | Code node with fixed rules |
| Wording of the email | Gemini |
| Final approval to send | A human, in Slack |

The subject line is built in code (`Welcome to <products> - Next Steps`), not by Gemini.

## Salesforce setup

Custom fields:

| Object | Field | Type |
|---|---|---|
| Opportunity | Implementation Required (`Implementation_Required__c`) | Checkbox |
| Opportunity | Products (`Products__c`) | Text(255), products separated by `+` |
| Opportunity | CSM (`CSM__c`) | Text(80) |
| Opportunity | Onboarding Status (`Onboarding_Status__c`) | Picklist: Not Started, Needs Review, Welcome Email Sent, Email Failed |
| Opportunity | Welcome Email Sent Date (`Welcome_Email_Sent_Date__c`) | Date/Time |
| Account | Customer Segment (`Customer_Segment__c`) | Picklist: Higher Ed, K-12, Corporate |

Connection: an OAuth2 **External Client App** (scopes `api` and `refresh_token`), used by an n8n Salesforce OAuth2 credential.

## Test scenarios

| Scenario | Data | Result |
|---|---|---|
| Normal onboarding | Closed Won, implementation required, CSM and contact present | Draft, Slack approval, email sent, status updated |
| Implementation not required | Closed Won, implementation not required | Stops at the implementation check |
| Missing CSM | Closed Won, implementation required, CSM blank | Slack exception, no email |
| Duplicate processing | Status already `Welcome Email Sent` | Stops at the duplicate check |
| Rejected draft | Reviewer clicks Disapprove | Slack message, no email |
| Gmail failure | Invalid recipient | Status set to `Email Failed`, Slack message |

## Error handling

- Missing or invalid data, failed AI checks and rejected drafts: Slack exception message, no email.
- Gemini and Gmail errors use n8n error outputs so they route to the same exception path.
- A failed Gmail send is never marked as sent, because the status update only runs after Gmail succeeds.
- Missing-data and AI-check exceptions do not write to Salesforce. A write would re-trigger the workflow and could loop. Fixing the record re-triggers it naturally.

## Security

- All secrets live in n8n's credential store. None are in this repository.
- Fictional data only. Test emails use plus-addressing so they reach the author's own inbox.
- The Gemini prompt contains only fields from Salesforce, and the draft is checked before a human sees it.
- Nothing is sent without human approval.

## Known limitations

- Polling means a delay of up to about a minute after a stage change.
- Two runs for the same Opportunity can both be waiting for approval at once, because a "pending" status is not written. A human sees both messages and can reject one.
- If several records are processed in one run, `Build Exception Report` looks at the whole execution, so an exception label could be wrong. Test one record at a time.
- If the Salesforce update fails after the email was sent, the status stays unchanged and the run fails visibly in n8n.
- Slack, Salesforce lookup and Salesforce update failures are not routed to the exception path.

## Ideas for a real deployment

- Use a Salesforce Platform Event or Flow instead of polling.
- Add a shared n8n Error Workflow for unrouted failures.
- Write a `Pending Review` status to block simultaneous approvals.
- Store the CSM as a Salesforce user lookup so the CSM's email can be used.
- Add approval rules by deal size or segment.
- Log events to a Salesforce custom object.

## Repo contents

- `workflow/onboarding-welcome-automation.json`: exported n8n workflow (no secrets)
- `docs/screenshots/`: full canvas, Slack approval, Slack exception, sent email, updated Opportunity
- `docs/demo.mp4` or a link to the demo recording
