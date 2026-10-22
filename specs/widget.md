# Widget

**Status:** Draft · **Prefix:** WID

## Purpose

A chat bubble embedded on a website: anonymous visitors chat at once, and signed-in users of that site can be identified.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| WID-1 | Widget channel: `{website_url, color (#RRGGBB, default #1f93ff), website_token (public, ≥ 128 random bits), hmac_token, hmac_required (default false), reply_time: in_a_few_minutes\|in_a_few_hours\|in_a_day (default in_a_few_minutes), welcome_title?, welcome_tagline?, features {attachments, emoji_picker, end_conversation, allow_mobile_webview} (first three default true), allowed_domains[], pre_chat {enabled (default false), message?, fields[]}, email_continuity (default true)}`. Identity kind: uuid. | new widget → `end_conversation` on |
| WID-2 | Visitor token: loading the widget with a missing, invalid or expired token creates an anonymous `visitor` contact with a generated name and an identity, and returns a signed token `{source_id, inbox_id, issued_at, expires_at}` valid 180 days. | expired token → new visitor and token |
| WID-3 | Every widget call resolves the inbox by `website_token` and the identity by the token's `source_id` within that inbox; unknown → 404. | token from inbox A on inbox B → 404 |
| WID-4 | Identify: `set_user {identifier, identifier_hash?, name?, email?, phone?, avatar_url?, custom_attributes?}` follows [CHN-12](channels.md) and matches or merges contacts per [CON-5](contacts.md). If the current contact already has a different `identifier`, a new contact identity and token are issued instead of overwriting. Invalid email or phone values are dropped, not fatal. | logged-in user B on A's browser → new token for B |
| WID-5 | Pre-chat field: `{name, label, type: text\|email\|number\|phone\|list\|checkbox\|date, enabled, required, placeholder?, values[]? (list), pattern?, pattern_hint?}`. Built-in names `email`, `full_name`, `phone` set the contact; other names set a contact or conversation custom attribute with the same key ([PRD-10](productivity.md)). Enabled required fields are checked server-side. | required email missing → refused |
| WID-6 | Start conversation: updates the contact from the pre-chat values (name only when empty), creates a conversation with `additional_attributes {browser_language, browser, platform, referer, initiated_at}` and the first incoming message. | start → conversation with referer URL |
| WID-7 | Send message: content and/or attachments (allowed only when `features.attachments`) into the conversation chosen by [CHN-4](channels.md); optional `labels` on a new conversation keep only labels defined in the workspace. | label `vip` not defined → ignored |
| WID-8 | The visitor lists only messages that are not private and not activity, paged as [MSG-12](messages.md); live updates follow [EVT-8](events.md). | private note → not in widget |
| WID-9 | End conversation: resolves it with the contact as actor when `features.end_conversation`; else 403. Already resolved → no change. | feature off → 403 |
| WID-10 | Embedding: with `allowed_domains` empty the widget may load on any site; else only in pages from those domains. With `allow_mobile_webview`, requests without a web origin (blank or file origin) are also allowed. | page on other.com, allowed `shop.com` → not loaded |
| WID-11 | Email continuity: when on and the contact has an email, public replies the contact has not seen ([CONV-13](conversations.md)) are emailed: one email per conversation per 2-minute batch, holding the new replies plus up to 10 earlier messages; replies to that email thread back per [EML-3](email.md). | 3 replies in 1 min, unseen → one email |
| WID-12 | Email collect: in a widget inbox with `email_collect`, the first incoming message from a contact without email adds, once per conversation, a template message asking for an email (`content_type` `input_email`); submitting it sets the contact's email ([CON-5](contacts.md) rules). | visitor answers → contact email set |
| WID-13 | Custom events: the page can send `{name (1–255), info?}`; each emits `widget.triggered` ([EVT-2](events.md)) with the identity, name, info and browser data. | event `opened_pricing` → webhook delivered |

## Out of scope

- Widget look and layout, help-center article search, campaigns shown in the widget, the widget's client SDK.

## Decisions

- The widget follows the inbox's conversation selection ([CHN-4](channels.md)); the source always appended to the visitor's latest conversation.
- `end_conversation` is a state-changing POST; the source routed it as GET.
