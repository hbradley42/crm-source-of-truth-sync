# Field Ownership — CRM Source-of-Truth Sync

## The rule, in one line

Airtable is the source of truth for billing status specifically, because
that's the system where billing state actually changes first. HubSpot
remains the source of truth for everything else in this design — there is
no single system that wins across the board.

## Field ownership table

| Field | Source of truth | Reasoning |
|---|---|---|
| Email, name, company | HubSpot | First point of contact owns identity data |
| Billing/subscription status | Airtable (standing in for a support/billing system) | System of record for payment events — billing state changes there first |
| Support tier | Support system | Reflects actual usage, not sales assumptions |
| Anything with no defined owner | Most recently updated wins | No natural authority — timestamp is the fallback, and every fallback resolution is logged, never silent |

## How conflicts are actually resolved

A scheduled n8n workflow pulls records from both systems, normalises their
field names so nothing collides, matches records by email, and applies the
ownership table above. Every conflict — resolved or not — is written to a
Conflicts Log table with both original values and the resolved value, so
nothing resolves invisibly. The resolved value is then written back to
HubSpot via the API, closing the loop.

## Known gap: one-sided records and the PATCH step

The workflow currently flags a record as a conflict if it exists in only
one system — for example, a contact that's in Airtable but hasn't been
created in HubSpot yet. In that case there's no HubSpot contact ID to
write back to, so `hs_contact_id` comes through as `null`.

Right now, that null value can reach the PATCH step unguarded — if it did,
the request would target a broken URL rather than failing safely. I'm
flagging this rather than fixing it in this build: the correct fix is a
second IF node, checking `hs_contact_id` exists before the PATCH runs,
with the false branch logging the record to a "needs manual
reconciliation" list instead of attempting an update. It's a small
addition and I know exactly what it looks like — I'm choosing to document
it here rather than build it immediately because the core
conflict-resolution logic was the priority for this demo, and this edge
case doesn't occur in the current sample data. This is the first thing
I'd close before trusting this workflow with real records.

## What I'd build into V2

- The guard above, closing the one-sided-record gap properly.
- Filtering the initial HubSpot/Airtable pulls to only records updated
  since the last successful run, rather than re-comparing every record on
  every scheduled run — fine at this scale, wouldn't hold up on a real
  customer base.
- Extending the ownership table to more than one field — this build proves
  the pattern on billing status alone, but the same structure extends
  cleanly to support tier, contract dates, or anything else that lives in
  more than