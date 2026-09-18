# CRM Source-of-Truth Sync

## The problem
HubSpot and a second operational system can disagree on the same
customer — most commonly billing/subscription status. Someone has to
decide, automatically, which value is correct, without guessing.

## How it works
A scheduled n8n workflow pulls records from both systems, compares them
field by field against a predefined ownership table, resolves conflicts
according to that table (falling back to most-recently-updated where no
natural owner exists), writes the resolved value back to both systems,
and logs every conflict — resolved or not — to an audit table.

## The actual answer to "who owns the data"
See `field-ownership-table.md`. The short version: ownership is assigned
per field based on which system generates that data first-hand. There is
no single global rule, and nothing resolves silently — every conflict is
logged even when the automation resolves it without a human.

## Tools
HubSpot (API) · Airtable (mock second system + audit log) · n8n
(Schedule Trigger, HTTP Request, Airtable node, Code, IF)

## Note on data
Fictional company and customer data, built for demonstration purposes.
