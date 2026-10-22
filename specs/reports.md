# Reports

**Status:** Draft · **Prefix:** RPT

## Purpose

How fast and how well the team answers: response and resolution metrics, summaries and satisfaction surveys.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| RPT-1 | Reporting event, written when it happens and never changed: `{name: conversation_opened\|conversation_resolved\|first_response\|reply_time, value (seconds), value_in_business_hours (seconds), workspace, inbox, user?, conversation, started_at, ended_at}`. | resolve → one `conversation_resolved` event |
| RPT-2 | `conversation_opened`: value 0 on creation; on reopen, seconds since the last resolve. `conversation_resolved`: seconds from the latest open (creation or reopen) to the resolve, `user` = assignee at that moment. | created 10:00, resolved 10:30 → 1,800 |
| RPT-3 | `first_response`: seconds from the latest open to the first human reply ([MSG-6](messages.md)) after it, `user` = replier. `reply_time`: for every later human reply, seconds from `waiting_since` to the reply, `user` = replier. | wait 10:00, reply 10:05 → 300 |
| RPT-4 | Business-hours value counts only seconds inside the inbox's open schedule ([INB-8](inboxes.md)) in its time zone; with `working_hours_enabled` false it equals `value`. | asked Fri 16:50, replied Mon 09:10, 09–17 → 1,200 |
| RPT-5 | Metrics for a range: `conversations_count` (created), `incoming_messages_count`, `outgoing_messages_count` (public, by agents), `avg_first_response_time`, `avg_resolution_time`, `avg_reply_time`, `resolutions_count`; averages over the events in range, in seconds; `business_hours=true` uses business-hours values. | 2 replies 60 s and 120 s → 90 |
| RPT-6 | Time series: `since`, `until` (unix seconds, required, `until` exclusive, else 422), `group_by: hour\|day\|week\|month\|year` (default day), `timezone_offset` or the workspace `reporting_time_zone` (IANA, default the team's, `@specfork/teams:TEAM-13`), scope `workspace\|inbox\|agent\|group\|label` with `id`; empty buckets returned as 0; averages also return a per-bucket `count`. | 3-day range by day → 3 buckets |
| RPT-7 | Agent scope = conversations assigned to the agent, messages the agent sent and events with `user` = the agent. Group and label scopes use the conversation's group and labels at report time. | relabel later → counts move |
| RPT-8 | Summaries per agent, inbox, group and label for a range ≤ 6 months (longer → 422): `conversations_count, resolutions_count, avg_first_response_time, avg_resolution_time, avg_reply_time`; every agent, inbox and group is listed, with zeros when idle. CSV download of each summary with durations formatted as hours, minutes, seconds. | 7-month range → 422 |
| RPT-9 | Live counts: `open`, `unattended` (open, no first reply or waiting), `unassigned` (open), `pending`; also grouped by assignee or by group. | 2 open unassigned → `unassigned` 2 |
| RPT-10 | Satisfaction survey: inbox `csat_rules {message?, display_type: emoji\|star (default emoji), label_operator: contains\|does_not_contain, labels[]}`. When a conversation becomes `resolved` in an inbox with `csat_enabled`, and no survey was ever sent in it, and the label rule passes (no labels = pass), a template message with `content_type` `input_csat` is sent; outside the reply window ([CHN-7](channels.md)) an activity note records that it was not sent. | resolve twice → one survey |
| RPT-11 | Survey response: `{conversation, message (unique), contact, assignee (at response time), rating 1–5, feedback? (≤ 2,000), created_at}`; the contact may change it for 14 days after the survey was sent, then 403. | rating 6 → refused |
| RPT-12 | Survey report (admin+): responses filterable by range, agent, inbox, group, rating; 25 per page; metrics `total_responses`, `ratings_count {1..5}`, `total_sent`, response rate; CSV download. | 4 sent, 2 answered → rate 50% |

## Out of scope

- SLA reports, bot resolution and hand-off metrics, traffic heatmaps, pre-aggregated rollups.

## Decisions

- Resolution time counts from the latest open, so reopened conversations are not over-counted; the source counted from creation.
- Reply time is credited to the replier, not the assignee.
- Business-hours values equal raw values when the inbox has no schedule; the source stored 0.
- A missing `since` or `until` is refused; the source silently returned nothing.
- The 14-day edit window applies on every path; the source missed it in the widget.
