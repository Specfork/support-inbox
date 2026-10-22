# Conversations

**Status:** Draft · **Prefix:** CONV

## Purpose

A conversation is one support thread between a contact and the team in one inbox, moved through statuses until resolved.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| CONV-1 | Conversation: `{id, display_id, uuid, inbox, contact, identity ([CHN-2](channels.md)), status: open\|resolved\|pending\|snoozed (default open), priority?: low\|medium\|high\|urgent, assignee?, group?, labels[], participants[], snoozed_until?, waiting_since?, first_reply_at?, last_activity_at, status_changed_at, agent_last_seen_at?, assignee_last_seen_at?, contact_last_seen_at?, additional_attributes, custom_attributes, created_at}`. `display_id` is a per-workspace sequence from 1, never reused, and is the id in every API path; `uuid` is random and used in public links. `additional_attributes` (browser, referer URL, mail_subject, …) and `custom_attributes` are maps; string values ≤ 1,500 chars, integers ≤ 9,999,999,999; an invalid `referer` URL is stored as null. | 2nd conversation in a new workspace → `display_id` 2 |
| CONV-2 | Initial status: `resolved` for a blocked contact ([CON-11](contacts.md)); else `pending` when the creator asks for it (a bot or triage step), else `open`. `last_activity_at` = creation; `waiting_since` = creation for contact-started conversations. | conversation for blocked contact → resolved |
| CONV-3 | Any agent with access may set any status. `snoozed_until` (future time, optional: empty = until the next incoming message) is kept only while `snoozed` and cleared otherwise. Toggle without a target: open → resolved, any other → open. Each change sets `status_changed_at`; resolving clears `waiting_since`. | snoozed → open → `snoozed_until` null |
| CONV-4 | Incoming message: `snoozed` or `resolved` → `open`; `open` and `pending` unchanged; a blocked contact's conversation is not reopened. Outgoing messages and notes never change status. | message into resolved → open |
| CONV-5 | Snoozed conversations whose `snoozed_until` has passed are reopened within 5 minutes, however late the job runs. | scheduler down 4 days → reopened on restart |
| CONV-6 | `waiting_since` is set by an incoming message when empty and cleared by a human reply ([MSG-6](messages.md)); private notes, activity and template messages never clear it. The first human reply sets `first_reply_at` once. | note only → still waiting |
| CONV-7 | Auto-resolve, workspace settings: `auto_resolve_after_minutes` (null = off, 10–1,438,560), `auto_resolve_message?`, `auto_resolve_label?`, `auto_resolve_skip_waiting` (default false). At least every 5 minutes, `open` conversations with `last_activity_at` older than the limit (and, when skipping, with no `waiting_since`) get the message, then the label, then `resolved` with actor `system`. | 60 min limit, idle 61 min → resolved |
| CONV-8 | List: `status` (default open; `all` = any), `assignee_type` me\|unassigned\|assigned\|all (unassigned = no assignee), `inbox_id`, `group_id`, `labels` (any of), `conversation_type` mentioned\|participating\|unattended (no first reply, or waiting), `q` (case-insensitive substring of incoming and outgoing message text; ignores status). Only conversations the user may access ([PERM-3](permissions.md)). Every list response has counts `mine, unassigned, assigned, all` computed after all filters except `assignee_type`; 25 per page. | `assignee_type=me` → only mine; 3 mine, 2 unassigned → `all` 5 |
| CONV-9 | Sort: `last_activity_at` desc (default) or asc, `created_at` asc/desc, `priority` desc/asc (nulls last, then last activity desc), `waiting_since` asc/desc (nulls last), `unread` (unread count desc, then last activity desc); ties by id. Unknown sort → default. | priority desc → urgent first, none last |
| CONV-10 | Condition list (conversation and contact filters, saved filters): ordered `{attribute_key, filter_operator, values[], query_operator?: and\|or}` joining each condition to the next; the last has none. Operators: equal_to, not_equal_to, contains, does_not_contain, is_present, is_not_present, is_greater_than, is_less_than, days_before (integer 1–998, in the request timezone). Values are scalars and required except for is_present/is_not_present; text comparisons are case-insensitive; unknown key or disallowed operator → 422 naming it. | last condition with `and` → 422 |
| CONV-11 | Conversation filter keys: status, priority, contact_id, browser_language (equality); assignee_id, inbox_id, group_id, labels (equality, presence); display_id, referer, mail_subject (equality, contains); created_at, last_activity_at (greater, less, days_before); conversation custom attributes by their type ([PRD-9](productivity.md)). | `created_at` contains → 422 |
| CONV-12 | Unread = incoming messages after the user's `agent_last_seen_at` (all if null), counted without cap. Opening a conversation sets `agent_last_seen_at` (and `assignee_last_seen_at` for the assignee) and marks its notifications read; mark unread sets them to just before the last incoming message. | open → unread 0 |
| CONV-13 | When the contact views the conversation, `contact_last_seen_at` = now and its outgoing messages in `sent` or `delivered` up to then become `read`. | widget opened → agent replies `read` |
| CONV-14 | Participants: users who follow a conversation; the assignee and mentioned users are added automatically; agents add, remove or replace them; each must have access to the conversation, else 422. | add user without access → 422 |
| CONV-15 | Update: `priority` (value or null), `custom_attributes` (replace, or merge with `merge=true`; delete by key list); other fields change only through status, assignment and label endpoints. | priority `extreme` → 422 |
| CONV-16 | Delete (admin+): removes the conversation and its messages in the background; for email inboxes the Message-IDs are remembered so polling does not import them again. | delete, next poll → not recreated |
| CONV-17 | Bulk: ≤ 100 `display_id`s with any of `status, snoozed_until, assignee_id, group_id, labels {add[], remove[]}`; each change runs with the same rules, permissions, events and activity as the single call; conversations the user cannot access are skipped and reported. | 101 ids → 422 |

## Out of scope

- Agent bots and bot hand-off rules, AI assistants, conversation drafts, transcripts sent by email, translation.

## Decisions

- No implicit self-assignment when an agent reopens a conversation; the source made a reopening agent the assignee, taking it from a colleague.
- Snoozes past due always reopen; the source skipped snoozes more than 3 days late.
- Unread counts are uncapped; the source capped them at 10.
- Bulk changes run the normal rules and are capped at 100; the source bypassed assignment checks and had no cap.
- Agent-started conversations do not start waiting; the source set `waiting_since` on every new conversation.
