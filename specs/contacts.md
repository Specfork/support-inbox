# Contacts

**Status:** Draft · **Prefix:** CON

## Purpose

The people a workspace talks to: one record per person across channels, with notes, labels and attributes.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| CON-1 | Contact: `{id, name (≤ 255), email?, phone?, identifier?, avatar?, blocked (default false), contact_type: visitor\|lead\|customer (default visitor), additional_attributes {city?, country_code?, company_name?, description?, social profiles?}, custom_attributes ([PRD-9](productivity.md)), labels[], last_activity_at?, created_at}`. Blank `email`, `phone` and `identifier` are stored as null. Avatar: jpeg, png, gif or webp, ≤ 15 MB, or fetched once from `avatar_url` when none is set. | create with email "" → email null |
| CON-2 | `email`: valid address, lower-cased, unique per workspace. `phone`: E.164 (`+`, then 2–15 digits, first digit 1–9), unique per workspace. `identifier` (the customer's own user id, ≤ 255): unique per workspace, case-sensitive. Clash → `taken` naming the field. | second contact with `ANN@x.com` → `taken` (email) |
| CON-3 | Type: a `visitor` becomes `lead` when saved with an email, phone, identifier or social profile, or when a conversation is created for it; types never go down automatically; `customer` is set only by an agent. Contacts created by agents or import start as `lead`. | visitor gives email → lead |
| CON-4 | Update merges `custom_attributes` and `additional_attributes` key by key; keys are removed only by a delete call listing them. Any message from the contact sets `last_activity_at`. | update `{plan: pro}` → other keys kept |
| CON-5 | Matching on inbound traffic and identify: find by `identifier`, then email (case-insensitive), then phone; never match a contact whose `identifier` differs from a non-empty given one; never match by phone when the given email differs from the contact's non-empty email. A match is updated with the given non-blank values (attributes deep-merged); no match → new contact (generated name when none given). | identify email of existing lead → same contact updated |
| CON-6 | List: 15 per page; sort by name, email, phone, city, country_code, company_name, created_at, last_activity_at (asc/desc, nulls last; default last_activity_at desc); filter by labels (any of). By default only contacts with email, phone or identifier are listed; `include_anonymous=true` lists all. | anonymous visitor → hidden by default |
| CON-7 | Search: `q` required (else 422); case-insensitive substring on name, email, phone and identifier; 15 per page with `has_more` instead of a total. | `q=ann` → Ann, Joanne |
| CON-8 | Filter: condition list ([CONV-10](conversations.md)) over name, email, identifier, phone, city, country_code, company_name, labels, blocked, created_at, last_activity_at and contact custom attributes. | email contains `@acme` → Acme people |
| CON-9 | Merge `{base_id, mergee_id}` (admin+): both in the workspace and different, else 422. In one transaction the mergee's conversations, messages, identities, notes and labels move to the base; fields: base value kept when present, else mergee's; attributes deep-merged with base winning; type = the higher; the mergee is deleted. Emits `contact.merged` then `contact.updated` with the final base. | base phone empty, mergee has one → base gets it |
| CON-10 | Delete (admin+) removes the contact with its conversations, messages, identities and notes in the background, and emits `contact.deleted` with the last state. | delete → conversations gone |
| CON-11 | Blocked contact: new conversations start `resolved`; incoming messages are stored but do not reopen ([CONV-4](conversations.md)); notifications only for mentions ([EVT-10](events.md)). Mute a conversation = block its contact and resolve it; unmute = unblock, no reopen. | blocked contact writes → conversation stays resolved |
| CON-12 | Notes: `{id, contact, author, content (1–20,000), created_at, updated_at}`, newest first; any agent adds; author or admin+ edits and deletes. | member edits another's note → 403 |
| CON-13 | Import (admin+): CSV with header row, UTF-8 (BOM stripped). Columns `name, email, phone, identifier, company_name, city, country_code, labels` (comma list); other columns become custom attributes only when a contact definition with that key exists, else ignored. Each row matches by identifier, email, phone (a phone missing `+` gets one); matched → updated, else created as `lead`. Status: pending\|processing\|completed\|completed_with_errors\|failed; rejected rows (invalid values, undefined labels) downloadable as CSV with an `errors` column. | row with label `ghost` → rejected, others imported |
| CON-14 | Export (admin+): CSV of the current list or filter, columns chosen from contact fields plus `labels` (default `id, name, email, phone, labels`), formula-safe (cells starting `= + - @` escaped), UTF-8 with BOM, delivered as a download link to the requester. | name `=SUM(1)` → exported as text |

## Out of scope

- Companies as records, contact enrichment, CRM sync, contact calls.

## Decisions

- Merge needs admin+ like delete; the source let any agent merge (and so delete) contacts.
- Unknown import columns are ignored unless defined; the source turned every unknown column into a custom attribute.
- Identifier search is case-insensitive like the other fields; the source was case-sensitive there only.
- `contact.updated` is emitted after the merge so receivers see the final base; the source emitted the event before updating it.
