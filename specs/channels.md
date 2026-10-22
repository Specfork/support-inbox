# Channels

**Status:** Draft · **Prefix:** CHN

## Purpose

How messages enter and leave an inbox: one adapter contract for every channel, plus the generic API channel.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| CHN-1 | Each inbox has exactly one channel. Built-in types: `web_widget` ([WID](widget.md)), `email` ([EML](email.md)), `api` (CHN-9). Other types are channel adapters that implement CHN-3, CHN-5, CHN-6 and optionally CHN-7; an adapter declares `{type, identity_kind: uuid\|email\|phone\|external, reply_window_hours?}`. | register adapter `sms` (phone) → inbox of type `sms` creatable |
| CHN-2 | Contact identity: `{contact, inbox, source_id, verified (default false)}`, `(inbox, source_id)` unique. Default `source_id` by identity kind: uuid → random UUID; email → the contact's lower-cased email; phone → the contact's E.164 phone. Missing email or phone for those kinds → `identity_missing`. | email inbox, contact without email → `identity_missing` |
| CHN-3 | Inbound: the adapter passes `{source_id, contact hints {name?, email?, phone?, identifier?, avatar_url?}, message {source_id, content?, attachments[], in_reply_to_source_id?, sent_at}}`. The system finds the identity by `source_id`, else matches or creates the contact ([CON-5](contacts.md)) and creates the identity, selects a conversation (CHN-4) and stores an `incoming` message. An inbound message whose `source_id` already exists in the same inbox is ignored (at-least-once adapters). | new sender → contact, identity, conversation, message created |
| CHN-4 | Conversation selection for an identity: if `lock_to_single_conversation`, its latest conversation whatever the status; else its latest conversation that is not `resolved`; else a new one. A message into a `resolved` conversation is refused (`conversation_closed`) when `allow_messages_after_resolved` is false. | lock off, last one resolved → new conversation |
| CHN-5 | Outbound: after commit, each public `outgoing` or `template` message without a `source_id` is handed to the adapter once; success stores the provider id as `source_id`; failure sets `status` = `failed` and `external_error`. Private notes and activity messages are never handed over. | adapter error → message `failed` with error text |
| CHN-6 | Delivery updates from the adapter, by message `source_id`, follow [MSG-10](messages.md); an unknown `source_id` is ignored. | `read` for unknown id → no change |
| CHN-7 | Reply window: if the channel declares `reply_window_hours`, a public outgoing message more than that many hours after the conversation's last incoming message, or with no incoming message, is refused (`reply_window_closed`); private notes are always allowed. No window → no limit. | 24 h window, last incoming 25 h ago → refused |
| CHN-8 | An agent with access to the inbox may start a conversation for a contact that has the identity the channel needs (email, phone, or an existing identity for uuid kinds), with a first outgoing message; such a conversation starts with `waiting_since` empty. | email inbox, contact with email → conversation created |
| CHN-9 | API channel: `{identifier (public, ≥ 128 random bits), hmac_token (≥ 256 bits), hmac_required (default false), webhook_url? (https, ≤ 2048), secret (≥ 256 bits), reply_window_hours? (> 0)}`. | `reply_window_hours` 0 → refused |
| CHN-10 | API channel outbound: conversation, message and typing events of the inbox ([EVT-2](events.md), private notes and activity excluded) are POSTed to `webhook_url` signed with `secret` per `@specfork/webhooks:HOOK-3`, retried per HOOK-5; when all attempts for a `message.created` delivery fail the message becomes `failed`. | receiver down → message `failed` after last retry |
| CHN-11 | Public inbox API (API channel and widget), authenticated only by inbox `identifier` and the identity's `source_id`: create contact (returns `source_id`, default random UUID), update contact (identifier immutable), list and create conversations, list messages (no private or activity), create incoming messages with attachments, resolve a conversation (actor: contact), typing on/off, mark seen. | unknown `source_id` → 404 |
| CHN-12 | Identity check (widget and public API): when an `identifier` is sent with `identifier_hash`, or the channel has `hmac_required`, `identifier_hash` must equal hex HMAC-SHA256(key = `hmac_token`, message = identifier), compared in constant time; mismatch → 401; match sets the identity `verified`. A verified identity lists all conversations of the contact in that inbox; an unverified one only its own. Admin+ can rotate an inbox's `hmac_token` (widget, api) and `secret` (api); the old value stops working immediately. | wrong hash → 401 |

## Out of scope

- Specific third-party adapters (social networks, messaging apps, SMS and voice providers), provider message templates and provider sign-up flows.

## Decisions

- One adapter contract instead of per-provider code; reply windows are declared data, not hard-coded per provider.
- Conversation selection and `allow_messages_after_resolved` apply to every channel; the source applied them per channel and the widget ignored the lock setting.
- API channel deliveries are signed and retried like other webhooks; the source sent once and did not retry.
- A verified hash at contact creation sets `verified`; the source missed this on the public API path.
