# Productivity

**Status:** Draft · **Prefix:** PRD

## Purpose

Tools that make agents faster and data consistent: labels, canned responses, macros, custom attributes and saved filters.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| PRD-1 | Label: `{id, title (2–255, lower-cased, starts with a letter or digit, then letters, digits, `_` or `-`; Unicode letters allowed), description?, color (#RRGGBB, default #1f93ff), show_on_sidebar (default true)}`; title unique per workspace. Create, update, delete: admin+; list and show: member+. | title `-vip` → refused; `VIP` → stored `vip` |
| PRD-2 | Conversations and contacts carry labels from the workspace's defined set only; undefined → 422 naming them. Calls: replace the set, add, remove. Each change on a conversation writes add/remove activity ([MSG-7](messages.md)). | add `ghost` → 422 |
| PRD-3 | Renaming a label updates it on every conversation and contact in the background, without activity messages; deleting a label removes it from all of them the same way. | rename `vip` → `gold` → conversations show `gold` |
| PRD-4 | Canned response: `{id, short_code (1–100, no spaces, unique per workspace case-insensitive), content (1–20,000, may hold variables [MSG-5](messages.md)), created_by}`. Any agent creates; author or admin+ edits and deletes. Variables are filled per conversation when the message is created. | duplicate `Refund` vs `refund` → refused |
| PRD-5 | Canned search: case-insensitive on short_code and content; ranked short_code prefix, then short_code substring, then content substring; no query → all, by short_code. | `ref` → `refund` before `pre-refund` |
| PRD-6 | Macro: `{id, name (1–255), visibility: personal\|global (default personal), actions[] (1–20, ordered), created_by}`. Members create only personal macros. Listing shows global macros plus the user's own. Run and show: global or own; edit and delete: own personal, or admin+ for global. | member creates global → stored personal |
| PRD-7 | Macro actions `{action, params}`: assign_agent (user id, `self`, or null), assign_group (id or null), add_label, remove_label, change_status (open\|pending\|resolved\|snoozed), resolve, snooze, change_priority (value or null), send_message (text with variables), add_private_note, send_attachment (stored file), mute. Unknown action or invalid params → refused on save. | action `teleport` → refused |
| PRD-8 | Running a macro on ≤ 100 conversations applies its actions in order to each, as the running user, with the same rules and permissions as the manual calls; a failing action is recorded and the rest continue; inaccessible conversations are skipped. | macro with invalid assignee → labels still added |
| PRD-9 | Custom attribute definition: `{id, model: conversation\|contact, key (PRD-10), display_name (1–255), display_type: text\|number\|currency\|percent\|link\|date\|list\|checkbox, description?, values[] (list only, ≥ 1), pattern? (regex, text only), pattern_hint?}`. Create, update, delete: admin+; list: member+. | list type without values → refused |
| PRD-10 | Definition key: letters, digits, `_`, `.`, `-`; unique per workspace and model; never equal to a standard filter key of that model ([CONV-11](conversations.md), [CON-8](contacts.md)), checked on create and on rename. | contact key `email` → refused; rename to `email` → refused |
| PRD-11 | Values written to `custom_attributes` with a definition are checked: number, currency, percent → number; date → ISO-8601 date; link → absolute http(s) URL; checkbox → boolean; list → one of `values`; text → string matching `pattern` if set. Keys without a definition are kept as given. | list value not in values → 422 |
| PRD-12 | Changing a definition updates widget pre-chat fields with the same key ([WID-5](widget.md)); deleting it removes that pre-chat field and the filter key; stored values stay. | delete definition → pre-chat field gone |
| PRD-13 | Saved filter: `{id, user, name (1–255), filter_type: conversation\|contact\|report, query (condition list [CONV-10](conversations.md))}`, private to its user; ≤ 1,000 per user per workspace; editing never counts against the limit. | 1,001st → refused; edit at 1,000 → allowed |

## Out of scope

- Automation rules (event-triggered actions) and their delays; help center and articles.

## Decisions

- Labels must be defined before use everywhere; the source checked this only in the widget and import.
- Canned responses get an author; the source let any agent change any shared response.
- Custom attribute values are validated by type on the server; the source checked them only in the client.
- The standard-key clash is checked on rename too; the source checked it on create only.
