---
name: triage-investigator
description: Investigate the production-support triage queue. Lists every Linear issue currently in Triage, follows each one's linked Sentry issue, reads what actually happened in Sentry, and returns a plain-text report pairing each triage ticket with a descriptive explanation of the underlying error. Use when asked to triage, review the triage queue, or explain what's blowing up in production.
allowed-tools:
  - cakewalk_cakewalk_sync
  - cakewalk_linear__list_issues
  - cakewalk_linear__get_issue
  - cakewalk_linear__list_teams
  - cakewalk_linear__list_comments
  - cakewalk_linear__get_attachment
  - cakewalk_sentry__find_organizations
  - cakewalk_sentry__get_sentry_resource
  - cakewalk_sentry__search_issues
  - cakewalk_sentry__analyze_issue_with_seer
metadata:
  version: "1.0.0"
---

# Triage Investigator

Walk the Linear **Triage** queue and, for each ticket, pull up the linked
Sentry issue to explain what actually went wrong in production. Produce a
single text report: one entry per triage ticket, each with a short, concrete
explanation of the error behind it.

Everything runs through the **cakewalk** gateway (the Linear + Sentry wrapper
wired into this plugin). All cakewalk tools require two audit fields on every
call: `__cwPromptId` (a UUIDv4) and `__cwPromptDescription` (a short sentence
describing the user's request). Generate the pair ONCE for this run and pass
the SAME two values on every tool call below.

## Procedure

### 1. List the triage issues

Call `cakewalk_linear__list_issues` with:

- `state: "triage"` — this is the Triage **state type**, so it spans all teams.
  (Issues still in triage have no `triagedAt`; filtering by the state type is
  the correct way to list them.)
- `limit: 100`
- `fields: ["id", "title", "description", "url", "status", "labels", "assignee", "team", "priority", "createdAt", "triageIntel"]`

If nothing comes back, report that the triage queue is empty and stop.

If you need to scope to one team, resolve it first with
`cakewalk_linear__list_teams` and pass `team` as well.

### 2. For each triage issue, find its Sentry link

Linear tickets created from Sentry are normally linked, but the link can live
in a few places. For each issue, in order:

1. Call `cakewalk_linear__get_issue` with the issue `id`. This returns the
   full issue **including its attachments**. Look through the attachments for
   one pointing at Sentry — a `url` containing `sentry.io` (or your Sentry
   host), or whose source/title mentions Sentry. That URL is the Sentry issue.
2. If no Sentry attachment, scan the issue **description** (from step 1) for a
   Sentry URL (`https://…sentry.io/issues/…`).
3. If still nothing, call `cakewalk_linear__list_comments` for the issue and
   look for a Sentry URL in the comments.

Note the Sentry URL (or Sentry short-ID like `PROJECT-1A2B`) you find. If an
issue has no Sentry link at all, mark it as such — see step 4's fallback.

### 3. Read what happened in Sentry

For each Sentry link found:

- Call `cakewalk_sentry__get_sentry_resource` with `url` set to the Sentry
  issue URL (the resource type is auto-detected). This returns the issue
  details: title, culprit/location, error type and message, level, status,
  event counts, users affected, first/last seen, and a stack-trace summary.
- Summarise, in plain language, **what went wrong**: the exception/error, where
  it fires (file/function), what triggers it, how often and since when, and
  whether it's unresolved/regressed/escalating.

Use `cakewalk_sentry__analyze_issue_with_seer` (Seer root-cause analysis) ONLY
when the issue details alone don't make the cause clear, or when the request
asks for root cause / a fix. Pass the `issueUrl`. It can take a couple of
minutes, so don't call it for every ticket by default — reserve it for the
unclear ones.

**Fallback when there's no link:** if a triage ticket has no Sentry link, try
`cakewalk_sentry__search_issues` with the organization slug (get it from
`cakewalk_sentry__find_organizations` — fetch once and reuse) and a `query`
built from the ticket's title/keywords. If you find a confident match, use it;
otherwise record that no matching Sentry issue was found and move on. Never
invent a Sentry issue.

### 4. Write the report

Return a single plain-text response. Lead with a one-line count
("N issues in triage"), then one entry per ticket:

```
<LINEAR-ID> — <ticket title>   [priority · assignee · team]
  Linear: <linear url>
  Sentry: <sentry url or "no linked Sentry issue">
  What happened: <2–4 sentences: the error, where/why it fires, scale
                 (events/users), first/last seen, current Sentry status>
```

Order by priority (Urgent → Low), then by most recently created. Keep each
explanation concrete and grounded strictly in the Sentry data — if something
isn't in the data, say so rather than guessing. End with a short overall
read: any clusters (same error across tickets), the most urgent item, and
anything with no Sentry link that needs a human to check.

## Notes

- If `cakewalk_sentry__*` or `cakewalk_linear__*` tools aren't available, the
  gateway may need a resync: call `cakewalk_cakewalk_sync` and check that
  `connected_namespaces` includes `linear` and `sentry`. If one is missing,
  tell the user to connect it on the Connect page before re-running.
- This is a read-only investigation. Don't modify Linear or Sentry issues
  (no status changes, comments, or assignments) unless the user explicitly
  asks for it as a follow-up.
