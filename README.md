---
specfork: 1
name: "@specfork/support-inbox"
summary: Shared customer-support inbox with contacts, conversations, email and chat-widget channels, assignment, groups, labels, canned replies, business hours and reports.
license: CC-BY-4.0
requires:
  "@specfork/auth": "^0.3.0"
  "@specfork/teams": "^0.3.0"
  "@specfork/webhooks": "^0.3.0"
  "@specfork/notifications": "^0.1.0"
derived_from:
  repo: https://github.com/chatwoot/chatwoot
  commit: abf371d2c080d89f0f09a9751677543c1ad22f1e
  url: https://www.chatwoot.com
  license: MIT
---

# Support inbox

A support team answers its customers in one shared place. Messages arrive through inboxes (email,
a website chat widget, a generic API channel, or any channel adapter). They become conversations
with a contact, and agents assign, label, reply to and resolve them.
Behaviour derived from Chatwoot's open-source code (MIT), written fresh and stack-free.
Files under Chatwoot's enterprise licence (`enterprise/`: SLA, custom roles, capacity limits, AI
assistant, audit logs, SAML) were not used.
Not affiliated with or endorsed by Chatwoot Inc. The name is used only to credit the source.

| Area | Prefix | Covers |
|---|---|---|
| [inboxes.md](specs/inboxes.md) | INB | Inboxes, settings, members, business hours, greeting and away messages |
| [channels.md](specs/channels.md) | CHN | Channel adapter contract, contact identities, routing, delivery, API channel |
| [widget.md](specs/widget.md) | WID | Website chat widget: visitor tokens, identity check, pre-chat form, continuity |
| [email.md](specs/email.md) | EML | Email channel: inbound routing, threading, polling, outbound headers |
| [contacts.md](specs/contacts.md) | CON | Contacts, identity matching, merge, notes, block, import and export |
| [conversations.md](specs/conversations.md) | CONV | Conversation fields, statuses, snooze, auto-resolve, lists, filters, unread, bulk |
| [messages.md](specs/messages.md) | MSG | Message types, private notes, attachments, mentions, activity log, delivery status |
| [assignment.md](specs/assignment.md) | ASN | Assignees, groups, availability, round-robin auto-assignment |
| [productivity.md](specs/productivity.md) | PRD | Labels, canned responses, variables, macros, custom attributes, saved filters |
| [events.md](specs/events.md) | EVT | Webhook events, payloads, inbox filter, live updates, notification types and recipients |
| [reports.md](specs/reports.md) | RPT | Response and resolution metrics, business-hours values, summaries, satisfaction surveys |
| [permissions.md](specs/permissions.md) | PERM | Team roles (with `viewer`) applied to inboxes, conversations and settings |

A workspace is a team (`@specfork/teams`); agents are its accepted members; all data belongs to exactly one workspace.
A *group* is a named set of agents inside a workspace (the source calls it a team).
All times are stored in UTC; API names are `snake_case`.
