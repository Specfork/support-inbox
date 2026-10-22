# Email

**Status:** Draft · **Prefix:** EML

## Purpose

An inbox fed by a mailbox, by forwarding or polling, whose replies go out as threaded email.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| EML-1 | Email channel: `{email (lower-cased, unique across the instance), forward_address (generated on create: ≥ 64 random bits @ the instance's inbound domain, unique), imap {enabled (default false), host, port, login, password, ssl (default true)}, smtp {enabled (default false), host, port, login, password, auth: plain\|login\|cram_md5, starttls (default true), ssl (default false)}, reauth_required (default false)}`. Passwords encrypted at rest. Saving with IMAP or SMTP enabled first tests the connection (10 s timeout); failure → `mail_connection_failed`, nothing saved. Identity kind: email. | wrong IMAP password → refused, inbox unchanged |
| EML-2 | Inbound routing: a received mail belongs to the email inbox whose `email` or `forward_address` is among its To, Cc, Bcc or original-recipient addresses (case-insensitive, `+tag` ignored), or to the conversation named by a reply address `reply+<conversation uuid>@<inbound domain>`. Other mail is dropped. | mail to `Support+x@Acme.com` → acme inbox |
| EML-3 | Threading, first match wins, always within the receiving workspace: (1) reply address uuid; (2) `In-Reply-To` equals a Message-ID this system generated (EML-7) or a message `source_id` in the same inbox; (3) any `References` id, same rules; (4) else a new conversation. A mail matching a forwarded message's id does not thread into it. | reply to our email → same conversation |
| EML-4 | New conversation: contact matched by sender email in the workspace, else created with the sender's display name; `additional_attributes.mail_subject` = Subject. | first mail from new address → new contact |
| EML-5 | Ignored mail: sender not a valid address (bounces), mail sent by this system's own notifications, duplicate Message-ID in the same inbox. Auto-replies (`Auto-Submitted` not `no`, or vacation headers) are stored with `auto_reply` = true. | same Message-ID twice → one message |
| EML-6 | Stored as `incoming`, `content_type` `incoming_email`: content = reply text without quoted history (≤ 150,000), `content_attributes.email {from, to, cc, bcc, subject, message_id, in_reply_to, references, html, text}`; NUL bytes stripped; at most the last 15 attachments kept. Message-ID becomes `source_id`. | mail with 20 files → 15 attachments |
| EML-7 | Outbound headers: Message-ID `<conversation/<conversation uuid>/messages/<message id>@<domain>>`; In-Reply-To = Message-ID of the last incoming mail, else `<account/<workspace id>/conversation/<conversation uuid>@<domain>>`; References = that mail's References plus In-Reply-To; Subject = `Re: <mail_subject>` once the thread has a prior message, else `[#<display_id>] <inbox name>` when no subject. | agent reply → In-Reply-To = customer's Message-ID |
| EML-8 | Outbound: To = contact email plus per-message `cc`, `bcc` (comma lists, each a valid address). With SMTP enabled, sent through it From the channel `email`; else From the workspace support address with Reply-To the reply address (EML-2). Display name: friendly = `<agent name> from <business_name or inbox name>`, professional = business or inbox name only. | professional → From shows only business name |
| EML-9 | IMAP polling: every minute per enabled inbox, mail since the previous day, deduplicated by Message-ID; one poll per inbox at a time; a mail failing 3 times within 6 hours is skipped. 10 consecutive authentication failures set `reauth_required`, stop polling and notify admins ([EVT-9](events.md)); saving new credentials clears it. | 10 auth errors → polling stops |
| EML-10 | Forward a message: an agent may forward a public message of an email conversation to ≥ 1 address; the copy is an outgoing message marked `forwarded`, with its own Message-ID and no In-Reply-To. | forward without recipient → refused |

## Out of scope

- Provider OAuth mailbox connection (Google, Microsoft), email templates and branded layouts, mailbox bulk import of history.

## Decisions

- Threading never looks outside the receiving workspace; the source matched reply addresses and In-Reply-To across all accounts.
- Replies into resolved email conversations follow [CHN-4](channels.md) like every channel; the source ignored inbox settings for polled mail.
