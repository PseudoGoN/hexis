---
name: triage-sentry-report
description: Pulls every Linear issue currently in Triage, finds the Sentry issue each one is linked to, inspects what actually happened in Sentry, and returns a plain-text report listing each triage issue with a descriptive explanation of the underlying error.
metadata:
  version: "1.1.0"
  owner: ProductionSupport
  runs-against: cakewalk MCP gateway (the `cakewalk` server on this plugin)
  requires-tools:
    - linear__list_teams
    - linear__list_issues
    - linear__get_issue
    - linear__get_attachment
    - sentry__find_organizations
    - sentry__search_issues
    - sentry__get_sentry_resource
    - sentry__analyze_issue_with_seer
---

# Triage → Sentry report

Produce a text report of everything sitting in Linear **Triage**, with, for each
item, the linked Sentry issue and a short explanation of what actually went
wrong. Use it for daily/weekly triage sweeps.

Linear issues in triage are *usually* linked to a Sentry issue (Sentry's Linear
integration adds the Linear issue as a link/attachment, and people often paste a
Sentry URL into the description or a comment). Your job is to resolve that link,
read the Sentry side, and explain it in prose.

## Tools

This skill runs against the **Cakewalk MCP gateway** (the `cakewalk` server on
this plugin), which exposes Linear and Sentry tools prefixed by their upstream.
Use whichever registered form matches — the gateway advertises them as below; if
your client namespaces them under the server, the equivalent `cakewalk` form is
the same tool.

- **Linear**: `linear__list_teams`, `linear__list_issues`, `linear__get_issue`,
  `linear__get_attachment`
- **Sentry**: `sentry__find_organizations`, `sentry__search_issues`,
  `sentry__get_sentry_resource`, `sentry__analyze_issue_with_seer`

Every gateway tool requires two audit fields, `__cwPromptDescription` (a short
sentence describing what the user asked) and `__cwPromptId` (a UUIDv4).
Generate ONE pair at the start of the run and pass the SAME pair, unchanged, on
every single tool call.

## Steps

### 1. List the triage issues

Call `linear__list_issues` with `state: "triage"` to get every issue whose
status type is *triage* across the workspace. Request useful `fields` so you
don't need a follow-up call for the basics:

```
linear__list_issues(
  state: "triage",
  limit: 250,
  fields: ["identifier", "title", "description", "url", "status", "priority",
           "assignee", "team", "createdAt", "triageIntel"],
  __cwPromptDescription: "<run description>", __cwPromptId: "<uuid>"
)
```

- If you need to scope to a specific team, call `linear__list_teams` first and
  pass `team`.
- If the result is paginated (a `cursor` is returned), keep calling with
  `cursor` until you have them all.
- If there are zero triage issues, say so plainly and stop.

### 2. Find the linked Sentry issue for each

For each triage issue, locate its Sentry issue. Look in this order and stop at
the first hit:

1. **Attachments / links** — call `linear__get_issue(id, includeRelations: true)`
   and inspect its attachments. The Sentry integration attaches a link whose URL
   contains `sentry.io/...` (or a self-hosted Sentry host) and an issue path
   like `/issues/<SHORT-ID>`. Use `linear__get_attachment` if you need the full
   URL.
2. **Description / body** — scan the issue `description` for a Sentry URL or a
   Sentry short-id (e.g. `PROJECT-1Z43`).
3. **Title** — sometimes the Sentry short-id or error class is in the title.

Extract either the full **Sentry issue URL** or the **short-id**. If you find
neither, mark the issue as *no Sentry link found* and move on — do not guess.

### 3. Read what happened in Sentry

For each resolved Sentry reference:

- Prefer fetching by URL: `sentry__get_sentry_resource(url: "<sentry issue url>")`.
- Otherwise fetch by id: first `sentry__find_organizations()` to get the org
  slug (do this once and reuse it), then
  `sentry__get_sentry_resource(resourceType: "issue", organizationSlug: "<slug>", resourceId: "<SHORT-ID>")`.

From the Sentry issue, pull out: the error type/culprit, the message, how often
it's happening (event count / users affected), when it was first and last seen,
the status (unresolved/resolved/ignored), and the most relevant stack-trace
frame.

Only when the issue details alone don't make the cause clear, and you want a
root-cause read, call `sentry__analyze_issue_with_seer(issueUrl: "<url>")`. Do
NOT call Seer on every issue by default — it is slow (minutes) and meant for
genuine root-cause work. Keep it to the ambiguous ones unless the user asked for
deep analysis on all of them.

### 4. Write the report

Return a single **plain-text** response. Lead with a one-line summary
(`N issues in triage, M linked to Sentry`), then one entry per triage issue:

```
<LINEAR-ID> — <title>   [priority · assignee · team]
  Linear:  <linear url>
  Sentry:  <sentry url or short-id>   (status · X events · Y users · last seen <date>)
  What happened: <2–4 sentences in plain language: the error, where it fires,
                 the likely trigger, and how impactful it looks.>
```

For issues with no Sentry link, keep the entry but write
`Sentry: none found` and base "What happened" on the Linear description alone,
saying explicitly that it is unverified against Sentry.

Guidelines for the explanation:
- Explain in prose a non-expert on-call engineer can follow — avoid dumping raw
  stack traces; quote at most the single most telling frame.
- Name the source of each claim (which Sentry issue) and the date it was true,
  per the knowledge base convention.
- Do not invent causes. If Sentry doesn't make the cause clear and you didn't
  run Seer, say "cause unclear from Sentry details".

### 5. (Optional) Sort

If the list is long, order entries by severity: Urgent/High priority first,
then by Sentry event count descending, so the worst offenders are at the top.
