# GALOR Universal Sales Automation V1 — Design

Date: 2026-08-16
Status: Approved architecture

## Objective
Build a reusable n8n-based sales automation engine by adapting proven workflows from the imported awesome-n8n-templates library rather than rebuilding common automation logic from scratch.

## V1 Sales Cycle
Apollo → Clay → AI research/qualification → HubSpot → human approval gate → personalized Gmail → automated follow-up sequence → Retell AI phone call → HubSpot logging.

The system must stop outreach automatically when a prospect replies, opts out, bounces, books, or converts.

## Architecture

### 1. Lead Acquisition — Apollo
- Source target accounts and decision-makers.
- Capture stable lead/company identifiers and source metadata.
- Deduplicate before enrichment.

### 2. Enrichment — Clay
- Enrich contact, company, website, role, and other campaign-relevant fields.
- Verify/enhance contact data where configured.
- Pass normalized lead records to qualification.

### 3. AI Research and Qualification
- Research the prospect using the available enriched data.
- Produce a qualification score, reason, personalization facts, offer angle, and outreach context.
- Reject leads that do not meet campaign rules before paid downstream actions where practical.

### 4. CRM System of Record — HubSpot
- Exact-match/dedupe before contact/company creation or update.
- Store source, enrichment, qualification, outreach status, reply state, call state, booking/conversion state, and suppression state.
- HubSpot is the durable sales record; n8n is the orchestration layer.

### 5. Human Approval Gate
- No first outbound email or first outbound phone call without explicit approval in V1.
- Approved leads enter outreach.
- Rejected leads remain logged but do not contact the prospect.

### 6. Personalized Email — Gmail
- Generate concise individualized outreach from verified prospect facts.
- Send only after approval.
- Follow-ups may proceed automatically after first-contact approval unless a stop condition occurs.

### 7. Follow-up State Machine
States: NEW → ENRICHED → QUALIFIED → PENDING_APPROVAL → APPROVED → CONTACTED → FOLLOW_UP → REPLIED/BOOKED/CONVERTED/SUPPRESSED.

Each transition must be idempotent so retries do not duplicate emails, calls, or CRM records.

### 8. Voice Outreach — Retell AI
- Call only approved, eligible leads.
- Respect campaign timing and suppression rules.
- Write call outcome, summary, disposition, and next action to HubSpot.

### 9. Global Stop/Suppression Rules
Immediately stop future outreach for:
- reply received;
- unsubscribe/opt-out;
- hard bounce or invalid address;
- booked meeting when campaign rules say outreach is complete;
- converted/won lead;
- manual suppression;
- do-not-contact/compliance condition.

## Reuse Strategy
Start from the imported template library, especially workflows covering AI lead management and AI cold-email generation. Extract useful patterns for triggers, normalization, AI prompting, routing, CRM updates, retries, and email composition. Do not blindly activate imported workflows or copy embedded credentials. Replace integrations and assumptions with GALOR-owned configuration.

## Modularity
Implement V1 as small workflows/modules with stable payload contracts:
1. lead-ingest
2. enrich
3. qualify
4. crm-upsert
5. approval
6. email-outreach
7. follow-up-orchestrator
8. voice-outreach
9. reply-and-stop-handler
10. audit/error-handler

Each module must be independently testable and replaceable. Apollo, Clay, Gmail, Retell, HubSpot, and the AI provider must not be hard-wired across unrelated modules.

## Data Contract
Every lead carries at minimum:
- lead_id
- source/source_id
- first_name/last_name
- company/domain
- role
- email/email_status
- phone/phone_status
- enrichment metadata
- qualification score/reason
- personalization facts
- approval status
- campaign/status
- HubSpot IDs
- last_contact_at/next_action_at
- suppression flag/reason
- idempotency keys for outbound actions

## Reliability and Cost Controls
- Deduplicate before expensive enrichment/AI actions.
- Cache enrichment/research results when safe.
- Use bounded retries with error routing, never uncontrolled retry loops.
- Require idempotency for sends/calls.
- Log every material transition and external action.
- Keep credentials in n8n credential storage/environment secrets, never workflow JSON or Git.
- Process only the fields needed for the active campaign.

## Compliance/Safety
- Honor opt-outs immediately and persist suppression.
- Do not call or email records marked do-not-contact.
- Apply configured calling windows and applicable campaign rules.
- Keep an audit trail of approval and outbound actions.

## Testing
Before production activation:
- fixture tests for normalized lead payloads;
- duplicate lead tests;
- qualification pass/fail tests;
- approval gate tests;
- email/call idempotency tests;
- reply/opt-out/bounce stop tests;
- HubSpot upsert tests;
- provider failure/retry tests;
- end-to-end dry run with outbound sends/calls disabled;
- limited approved pilot before scale.

## Success Criteria
V1 is complete when a qualified Apollo lead can move through Clay enrichment, AI qualification, HubSpot, owner approval, personalized Gmail outreach, automated follow-up, optional Retell call, and HubSpot outcome logging without duplicate contact and with reliable global stop conditions.

## Explicitly Out of Scope for V1
- Fully autonomous first contact without human approval.
- Replacing HubSpot with a custom CRM.
- Building custom equivalents of Apollo, Clay, Gmail, or Retell.
- Unrelated website or GALOR product redesign.
