# Inboxes

**Status:** Draft · **Prefix:** INB

## Purpose

An inbox is one entry point for customer messages (one channel), with its own agents, schedule and automatic messages.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| INB-1 | Inbox: `{id, name (1–255), channel ([CHN-1](channels.md)), time_zone (IANA, default the workspace's, `@specfork/teams:TEAM-13`), auto_assign (default true), greeting_enabled (default false), greeting_message? (≤ 10,000), working_hours_enabled (default false), out_of_office_message? (≤ 10,000), csat_enabled (default false), csat_rules ([RPT-10](reports.md)), allow_messages_after_resolved (default true), lock_to_single_conversation (default false), email_collect (default true), assign_limit (default 100, > 0), assign_window_seconds (default 3,600, > 0), sender_name_type: friendly\|professional (default friendly), business_name?}`. | create with timezone `Mars/Base` → refused |
| INB-2 | Create, update and delete inboxes: admin+. Create takes a channel type from those registered ([CHN-1](channels.md)); an unknown type → `invalid_channel_type`. Inbox, channel and schedule changes in one update are saved together or not at all. Channel secrets (identity-check key, API secret) are returned only to admin+; mail server passwords are write-only and never returned. | update with invalid working hours → inbox name unchanged too |
| INB-3 | Inbox members: `(inbox, user)` unique. Admins add, remove, or replace the whole set with `user_ids`; every id must be an accepted workspace member (`@specfork/teams:TEAM-2`), else the call fails with `invalid_user_ids` and nothing changes. | replace with [Ann, stranger] → `invalid_user_ids`, set unchanged |
| INB-4 | A member or viewer sees only inboxes they belong to; admin+ see all. Assignable agents of an inbox = its member+ members plus all admin+ members of the workspace. | member lists inboxes → only own |
| INB-5 | Deleting an inbox returns at once and removes, in the background, its channel, members, schedule, conversations, messages and contact identities ([CHN-2](channels.md)); contacts stay. | delete inbox → contact still listed, its conversations gone |
| INB-6 | Schedule: each inbox has exactly 7 rows `{day_of_week 0 (Sunday)–6, open_all_day (default false), closed_all_day (default false), open_hour, open_minute, close_hour, close_minute}`, created with the inbox: Monday–Friday 09:00–17:00, Saturday and Sunday closed. | new inbox → Monday 09:00–17:00 |
| INB-7 | Schedule rules: hours 0–23, minutes 0–59; close strictly after open (no overnight ranges); `open_all_day` and `closed_all_day` not both true; `open_all_day` stores 00:00–23:59; times are not required on closed days. A bulk update matches rows by `day_of_week` and is all-or-nothing. | open 17:00, close 09:00 → refused |
| INB-8 | Open now = today's row in the inbox time zone is `open_all_day`, or not closed and open time ≤ local time < close time. An inbox is *away* when `working_hours_enabled` is true and it is not open now. | Mon 17:00 local, close 17:00 → away |
| INB-9 | Away message: on a new incoming message, if the inbox is away and `out_of_office_message` is non-empty, post it as a `template` message ([MSG-1](messages.md)), unless an agent sent a public reply in the last 5 minutes or the conversation already got a template message today (inbox time zone). | 2 messages at night → one away reply |
| INB-10 | Greeting: on an incoming message in a conversation that has no outgoing or template messages yet, post `greeting_message` as a template message when `greeting_enabled` and the text is non-empty. The away check runs first; an away message suppresses the greeting. | first message in hours → greeting only |
| INB-11 | No automatic message (greeting, away, email collect [WID-12](widget.md)) is sent for email auto-replies ([EML-5](email.md)) or for conversations started by the workspace ([CHN-8](channels.md)). | auto-reply email at night → no away reply |

## Out of scope

- Outbound campaigns, agent bots and bot triage, per-agent capacity limits and SLA policies (SLA and capacity are enterprise-only in the source).

## Decisions

- Close time is exclusive; the source counted the whole closing minute as open.
- A failed bulk schedule update rolls back; the source could leave it half-applied.
- Mail passwords are never returned; the source returned them to admins in plain text.
