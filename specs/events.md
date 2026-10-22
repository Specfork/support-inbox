# Events

**Status:** Draft · **Prefix:** EVT

## Purpose

What the inbox tells the outside world (webhooks), open screens (live updates) and agents (notifications) when something happens.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| EVT-1 | Webhook subscriptions (`@specfork/webhooks:HOOK-1`) have scope = workspace, list event names or patterns (`@specfork/webhooks:HOOK-12`) from EVT-2, and are managed by admin+. Filter `inbox_ids` (list, `@specfork/webhooks:HOOK-13`): only events whose `inbox_id` is listed are sent; events without one (contact) are not. | filter inbox 1 → inbox 2 messages and contact events not sent |
| EVT-2 | Events: `conversation.created`, `conversation.updated`, `conversation.status_changed`, `conversation.typing_on`, `conversation.typing_off`, `message.created`, `message.updated`, `contact.created`, `contact.updated`, `contact.merged`, `contact.deleted`, `inbox.created`, `inbox.updated`, `widget.triggered`. A failed or rolled-back write emits nothing. | create conversation with first message → `conversation.created`, `message.created` |
| EVT-3 | `conversation.updated` fires only when one of assignee, group, status, snoozed_until, priority, labels, custom_attributes, waiting_since or first_reply_at changed; `conversation.status_changed` also fires on status change. Both carry `changed_attributes {field: {previous, current}}`; assignee changes carry `automatic` ([ASN-10](assignment.md)). | seen-time update only → no event |
| EVT-4 | Conversation `data`: `{display_id, uuid, inbox_id, status, priority, labels, assignee {id, name, email}?, group {id, name}?, contact {id, name, email, phone, identifier}, verified, snoozed_until, waiting_since, first_reply_at, last_activity_at, unread_count, custom_attributes, additional_attributes, last_message, created_at, workspace {id, name}}`. | payload has `display_id`, not internal id |
| EVT-5 | Message events are emitted for incoming, outgoing and template messages, not activity: `data {id, conversation (display_id, uuid), inbox_id, message_type, content_type, content, content_attributes, private, status, source_id, sender {type: user\|contact, id, name}, attachments [{id, file_type, url, size}], created_at}`. Private notes are delivered to workspace subscriptions with `private` true and never to channel endpoints ([CHN-10](channels.md)). | note → workspace hook gets `private: true`, API inbox URL gets nothing |
| EVT-6 | `contact.updated` and `inbox.updated` fire only when a stored field changed and carry `changed_attributes`; `contact.deleted` carries the last state; `contact.merged` carries `{base, mergee}`. | save contact unchanged → no event |
| EVT-7 | Live updates to agents, within 5 s: conversation and message events go to every user with access to that conversation ([PERM-3](permissions.md)); private notes and activity included; contact events go to all workspace members; typing goes to all except the typer. Each payload names the acting user (`performer`) when there is one. Typing state is never stored. | member outside the inbox → no live event |
| EVT-8 | Live updates to the contact's open widget or public API sessions: messages that are not private and not activity, status changes, agent typing that is not marked private, and read receipts. A verified contact receives them on all its verified sessions in the inbox; an unverified one only on its own. | agent note → visitor sees nothing |
| EVT-9 | Notification types (`@specfork/notifications:NTFY-1`) and recipients: `conversation_created` → inbox members; `conversation_assigned` → the new assignee; `conversation_mention` → each mentioned user ([MSG-13](messages.md)); `assigned_conversation_new_message` → the assignee; `participating_conversation_new_message` → participants; `mailbox_reauth_required` ([EML-9](email.md)) and `webhook_disabled` (`@specfork/webhooks:HOOK-7`) → admin+. New-message types fire only for public incoming and outgoing messages. | assign to Ann → Ann notified |
| EVT-10 | Recipient rules: no `conversation_created` or `conversation_assigned` while the conversation is `pending`; for a blocked contact only mentions; the recipient must have access to the conversation ([PERM-3](permissions.md)); one type per user per message (assignee type wins over participant). | Ann is assignee and participant → only `assigned_conversation_new_message` |
| EVT-11 | Subject `{kind: conversation, id: display_id}` (`mailbox_reauth_required`: the inbox; `webhook_disabled`: the subscription); `title` = `#<display_id> <contact name>`; `body` = message text. Opening a conversation marks the user's notifications for it read ([CONV-12](conversations.md)). | open conversation → its notifications read |
| EVT-12 | Preferences (`@specfork/notifications:NTFY-7`): in-app on for every type; email `instant` for `conversation_assigned`, `conversation_mention`, `mailbox_reauth_required` and `webhook_disabled`, `off` for the rest. | new user, assigned → email within 1 min |

## Out of scope

- Push and mobile notifications; automation-rule webhooks.

## Decisions

- Dotted event names like other Specfork specs; the source used underscores.
- Workspace webhooks are signed and retried by the base; the source sent once.
- Inbox webhooks become one workspace subscription filtered by `inbox_ids`; notification inbox, grouping and email come from `@specfork/notifications`.
- Users who can open a conversation by group membership also see it in lists and get its live updates; the source granted only opening it.
