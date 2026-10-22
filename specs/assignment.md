# Assignment

**Status:** Draft · **Prefix:** ASN

## Purpose

Who owns a conversation: one assignee, an optional group, and fair automatic distribution among available agents.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| ASN-1 | A conversation has at most one assignee. Assign `{assignee_id}`: the user must be an accepted workspace member with role member+ who is an inbox member or admin+, else 422 `invalid_assignee`; null unassigns. The change is made under a row lock; the last committed write wins. | assign non-member of the inbox → 422 |
| ASN-2 | Group: `{id, name (1–100, trimmed, unique per workspace case-insensitive), description?, allow_auto_assign (default true)}`; members `(group, user)` unique, each an accepted workspace member. Create, update, delete and membership: admin+; any agent lists groups and members. | duplicate name `Billing` vs `billing` → refused |
| ASN-3 | Set group `{group_id}` (null removes): the assignee is cleared when not a group member; then, if unassigned, the group has `allow_auto_assign` and the inbox has `auto_assign`, one agent is picked per ASN-6. Deleting a group clears it on its conversations; assignees stay. | move to Billing, assignee not in Billing → unassigned, then auto-picked |
| ASN-4 | Availability per workspace member: `availability: online\|busy\|offline (default online)`, `auto_offline (default true)`; set by the member, or by admin+ for others. Presence: clients ping every 20 s; a user is present if pinged in the last 60 s. Effective status = stored availability if `auto_offline` is false; else stored availability while present, `offline` when not. | auto_offline, no ping 2 min → offline |
| ASN-5 | Auto-assignment runs when the inbox has `auto_assign` and a conversation becomes `open` (created or reopened) with no assignee, or with an assignee who lost access, and the same change does not set an assignee. `pending`, `snoozed` and `resolved` conversations are never auto-assigned. | new open conversation → assigned |
| ASN-6 | Candidates: member+ inbox members whose effective status is `online` (not busy), limited to the group's members when the conversation has a group, and none when that group disallows auto-assign. Pick round-robin: the candidate assigned least recently in this inbox; a newly added inbox member counts as just assigned. | A got the last one, B online → B |
| ASN-7 | Fair limit per inbox: `assign_limit` (default 100) auto-assignments per agent per `assign_window_seconds` (default 3,600, sliding); agents at the limit are skipped. Manual assignment is not limited or counted. | limit 1, A got one → next goes to B |
| ASN-8 | No candidate → the conversation stays unassigned. A sweep at least every 30 minutes, and whenever a member of the inbox becomes online, retries open unassigned conversations of inboxes with `auto_assign` whose last activity is within 168 hours, oldest `created_at` first. | nobody online, A comes online → assigned to A |
| ASN-9 | Removing a user from an inbox unassigns them from that inbox's non-resolved conversations unless they are admin+; on leaving or removal from the workspace (`@specfork/teams:TEAM-15`) all their conversations are unassigned and their inbox and group memberships removed. Each unassignment writes activity and events like a manual one. | remove A from inbox → A's open conversations unassigned |
| ASN-10 | Every assignee change adds the new assignee as a participant ([CONV-14](conversations.md)), writes an activity message ([MSG-7](messages.md)) with actor = acting user or `system`, and emits `conversation.updated` ([EVT-3](events.md)) with `automatic` true when made by auto-assignment. | auto-assign → event `automatic` true |

## Out of scope

- Per-agent capacity limits, balanced or weighted assignment orders (enterprise in the source), agent bots as assignees, skills-based routing.

## Decisions

- Assignees must have access to the inbox; the source accepted any account user.
- Removing an agent unassigns their conversations at once and with normal events; the source left stale assignees in inboxes and unassigned silently on account removal.
- A group's auto-pick respects the inbox's `auto_assign`; the source ignored it.
- Presence window 60 s with a 20 s ping; the source used 20 s for both, so agents flapped offline.
- One assignment model with the fair limit and sweep always on; the source had two versions behind a feature flag.
