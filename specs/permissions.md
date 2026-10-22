# Permissions

**Status:** Draft · **Prefix:** PERM

## Purpose

Team roles (`@specfork/teams:TEAM-2`) applied to inboxes, conversations, contacts and settings.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| PERM-1 | Resource rows (`@specfork/teams:TEAM-10`). viewer+ (`@specfork/teams:TEAM-11`): read inboxes they belong to, accessible conversations and their messages, contacts, labels, groups, canned responses and custom attribute definitions; add private notes; own saved filters. member+: reply, label, assign, set status and priority on accessible conversations; bulk and macro runs on them; contacts create, update, notes, labels; canned responses create and own edits; own personal macros; own availability. admin+: inboxes create, update, delete, members, secrets; groups and members; labels; custom attribute definitions; global macros; others' canned responses and notes; contact delete, merge, import, export; conversation delete; message delete of others ([MSG-11](messages.md)); others' availability; workspace support settings (auto-resolve, reporting time zone, inbound email domain); reports and surveys; webhook subscriptions. owner: nothing beyond `@specfork/teams:TEAM-6`. | member deletes contact → 403; viewer replies → 403 |
| PERM-2 | Every inbox-scoped and conversation-scoped check uses the member's current role and memberships at request time, for sessions and API keys alike (`@specfork/auth:AUTH-20`); removal from an inbox or group ends access on the next request. | removed from inbox → next read 403 |
| PERM-3 | Conversation access: admin+ all; a member when they belong to the conversation's inbox or to its group. The same rule governs reading by id, lists, counts, search, filters, bulk, macros, mentions, participants, notifications and live updates. | member of group Billing, not of inbox → sees Billing conversations in lists |
| PERM-4 | Contacts are workspace-wide: any member reads all contacts, even those whose conversations they cannot access; conversation lists inside a contact show only accessible ones. | contact page → only accessible conversations |
| PERM-5 | Personal resources (saved filters, personal macros) are visible and editable only by their owner, whatever their role. | admin reads another's saved filter → 404 |
| PERM-6 | Public endpoints (widget, public inbox API) never accept agent credentials and reach only the identity's own data ([CHN-11](channels.md), [WID-3](widget.md)); the workspace's `require_2fa` (`@specfork/teams:TEAM-12`) applies to agents only, never to them. | widget token reads other visitor's conversation → 404 |

## Out of scope

- Custom roles and per-permission grants (enterprise-only in the source).

## Decisions

- Fixed roles from `@specfork/teams`; the source's agent maps to member and administrator to admin+. `viewer` (not in the source) reads and adds private notes but never replies, assigns or changes status, for supervisors and auditors.
- Group membership grants full conversation access, consistently; the source granted it for opening by id only.
- Labels are readable one by one by members; the source allowed listing but not showing a single label.
