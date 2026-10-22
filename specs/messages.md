# Messages

**Status:** Draft · **Prefix:** MSG

## Purpose

What a conversation is made of: customer messages, replies, private notes, system activity and attachments.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| MSG-1 | Message: `{id, conversation, message_type: incoming\|outgoing\|activity\|template, content? (≤ 150,000), content_type (default text), private (default false), status: sent\|delivered\|read\|failed (default sent), external_error?, source_id?, sender? (user\|contact; none = system), in_reply_to?, content_attributes, attachments[], created_at}`. Messages are ordered by `created_at` asc, then id. A message needs content or ≥ 1 attachment. `in_reply_to` (message id) or `in_reply_to_source_id` is resolved within the same conversation; no match → null, not an error. | empty message, no file → refused |
| MSG-2 | `content_type`: text, input_text, input_textarea, input_email, input_select, cards, form, article, incoming_email, input_csat. Interactive types need ≥ 1 item in `content_attributes.items`, each a map: select `{title, value}`; cards `{title, description?, media_url?, actions[{type: link\|postback, text, uri?, payload?}]}`; form `{name, type, label, placeholder?, required?, options?}`; article `{title, description?, link}`; unknown keys → refused. | cards item without actions → refused |
| MSG-3 | Agent create: `message_type` defaults to `outgoing`, sender = the user; `incoming` only through a channel adapter or the public API ([CHN-11](channels.md)), else `incoming_not_allowed`. Each new message sets the conversation's `last_activity_at`. An optional client `echo_id` is returned in the live event, not stored. | agent posts incoming in email inbox → refused |
| MSG-4 | Private note = `outgoing` with `private` true: visible only to agents with access, never sent to a channel, the widget or the public API, never reopens, notifies or counts as a reply. | note → absent from widget list |
| MSG-5 | Variables in outgoing and template content are replaced at creation: `{{contact.name}}`, `{{contact.first_name}}`, `{{contact.last_name}}`, `{{contact.email}}`, `{{contact.phone}}`, `{{contact.custom_attribute.<key>}}`, `{{agent.name}}`, `{{agent.first_name}}`, `{{agent.last_name}}`, `{{agent.email}}`, `{{conversation.id}}` (display id), `{{conversation.custom_attribute.<key>}}`, `{{inbox.name}}`, `{{account.name}}`; unknown → empty; text inside backticks and malformed templates are left as is. | `Hi {{contact.first_name}}` to Ann Lee → `Hi Ann` |
| MSG-6 | Human reply = public `outgoing` message sent by a user (or echoed by the channel as sent from its native app), not forwarded, not a template and not sent by the system (auto-resolve). Only human replies clear `waiting_since` and set `first_reply_at` ([CONV-6](conversations.md)). | template greeting → still waiting |
| MSG-7 | Activity messages (`activity`, sender none) are added for status change, assignee change, group change, label add/remove, priority change, mute/unmute, with `content_attributes.activity {type, actor (user name or system), from?, to?}`. A group change that also changes the assignee writes only the group entry. | resolve by Ann → activity "resolved", actor Ann |
| MSG-8 | Flood limit: a conversation accepts at most 200 messages per minute; more → 429 `rate_limited`. | 201st in a minute → 429 |
| MSG-9 | Attachments: ≤ 15 per message; size ≤ the instance limit (default 40 MB); type image (jpeg, png, gif, bmp, webp), video (mp4, webm, ogg, quicktime), audio (any audio), file (pdf, txt, csv, rtf, json, xml, zip, 7z, rar, tar, office and open-document formats) or location `{lat, lng}`; `file_type` set from the MIME type; images get a 250 px wide thumbnail. | 16 files → refused |
| MSG-10 | Status moves only forward: sent → delivered → read; `failed` can be set from and left to any status; `external_error` is kept only while `failed`. Retry: an agent may resend a `failed` message once per request: status → `sent`, error cleared, handed to the channel again ([CHN-5](channels.md)); other statuses → 422. | `delivered` after `read` → stays read |
| MSG-11 | Delete (soft): content replaced by a fixed deleted marker, `content_type` → text, `content_attributes` → `{deleted: true}`, attachments destroyed. Allowed for own outgoing messages and notes (member+) and any non-activity message (admin+); activity messages cannot be deleted. No content editing. | member deletes colleague's reply → 403 |
| MSG-12 | Paging: newest 20 by default; `before=<id>` the 20 before; `after=<id>` the next 100; both → up to 1,000 between them; `hide_internal=true` drops notes and activity. | `before` oldest id → empty |
| MSG-13 | Mentions: a note may mention users or groups (`mention://user/<id>` or `mention://group/<id>` links); a group expands to its members; only users with access to the conversation count. Each mention records `(user, conversation, mentioned_at)` (unique, time refreshed), adds a participant and notifies ([EVT-9](events.md)). | mention a group of 3 → 3 participants |

## Out of scope

- Message translation, voice and video calls, stickers and provider-specific content types, rich-text editor details.

## Decisions

- Delete is limited to own messages for members and admin+ for others; the source let any agent delete any message, including customer messages.
- The 15-attachment cap and size limit apply on every channel; the source checked size and type for the widget only and its count check could be bypassed.
- Activity entries are written even without a user (actor `system`); the source skipped some.
