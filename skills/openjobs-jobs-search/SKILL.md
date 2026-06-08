---
name: openjobs-jobs-search
version: 2.0.1
description: Search and discover job positions using OpenJobs AI. Job search returns job IDs first; full job documents are fetched through entity detail APIs.
metadata: {"clawdbot":{"emoji":"💼","requires":{"env":["MIRA_KEY"]},"primaryEnv":"MIRA_KEY"}}
---

# 💼 Openjobs Jobs Search

Search and discover job positions from the OpenJobs AI job database.

## When to use

Use this skill when the user needs to:
- Search for job positions using structured filters
- Find open positions by title, company, location, seniority, employment type, or industry
- Fetch full job documents after a job ID search

## Version Check

At the start of every session, check whether this skill is up to date:

```bash
curl -s https://mira-api.openjobs-ai.com/version
```

Compare the returned `version` with this skill's frontmatter `version: 2.0.1`. If the server version is newer, notify the user that they should update the skill.

## First-time Setup

Check whether an API key is available:

```bash
echo $MIRA_KEY
```

If no key is found, ask the user whether they have a Mira API key. If yes, ask them to provide it and set:

```bash
export MIRA_KEY="mira_your_key_here"
```

If no, direct them to sign up at https://platform.openjobs-ai.com/.

Do not make API calls until a valid key is available.

## API Basics

All protected requests need:

```bash
curl -X POST "https://mira-api.openjobs-ai.com/v1/..." \
  -H "Authorization: Bearer $MIRA_KEY" \
  -H "Content-Type: application/json"
```

Unified response format:

```json
{ "code": 200, "message": "ok", "msg": "ok", "data": { } }
```

## Core Workflow

Mira API 2.0.1 uses ID-first job search:

1. Search jobs with `/v1/job-fast-search`.
2. Take returned `job_ids`.
3. Fetch full job docs with `/entity/v1/jobs/detail-by-id`, max 50 IDs per request.

Do not tell users that `/v1/job-fast-search` returns full job documents. It returns job IDs only.

## Common Operations

### Search jobs

```bash
curl -X POST "https://mira-api.openjobs-ai.com/v1/job-fast-search" \
  -H "Authorization: Bearer $MIRA_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Python Engineer",
    "country": "United States",
    "employment_type": "Full-time",
    "seniority": "Mid-Senior level",
    "size": 10000
  }'
```

At least one search condition is required. Only active, non-deleted jobs are returned.

Response:

```json
{
  "code": 200,
  "message": "ok",
  "msg": "ok",
  "data": {
    "job_ids": [1001, 1002]
  }
}
```

### Search by date range

```bash
curl -X POST "https://mira-api.openjobs-ai.com/v1/job-fast-search" \
  -H "Authorization: Bearer $MIRA_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Data Scientist",
    "time_posted_from": "2025-01-01",
    "time_posted_to": "2025-06-30"
  }'
```

### Fetch job details

```bash
curl -X POST "https://mira-api.openjobs-ai.com/entity/v1/jobs/detail-by-id" \
  -H "Authorization: Bearer $MIRA_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "job_ids": [1001, 1002],
    "_source": ["id", "title", "company_name", "location", "employment_type", "seniority", "industry"]
  }'
```

Maximum 50 job IDs per request. If `_source` is omitted, Mira returns the full job document. The response carries `total`, `found`, `not_found`, and `results`.

## Search Filter Fields (job-fast-search)

**Fuzzy match fields** (full-text search, affects relevance scoring):
- `title` — Job title (max 200 chars)
- `description` — Job description keywords (max 500 chars)
- `company_name` — Company name, phrase match (max 200 chars)
- `function` — Job function / direction (max 200 chars)

**Exact match fields** (precise filtering):
- `seniority` — Seniority level (max 100 chars). Valid values:
  `Entry level`, `Mid-Senior level`, `Associate`, `Director`, `Executive`, `Internship`, `Not Applicable`
- `employment_type` — Employment type (max 100 chars). Valid values:
  `Full-time`, `Part-time`, `Contract`, `Temporary`, `Internship`, `Volunteer`, `Other`
- `location` — Job location (max 200 chars)
- `country` — Country (max 100 chars)
- `industry` — Industry (max 200 chars)

**Date range fields** (ISO 8601 format):
- `time_posted_from` — Posted after (e.g. `"2025-01-01"`)
- `time_posted_to` — Posted before (e.g. `"2025-12-31"`)

**Search control:**
- `size` — Optional max job IDs, 1-10000, defaults to `10000`

## Data Source

All job data returned by this API comes exclusively from the OpenJobs AI database. Do not mix, substitute, or supplement it with external job boards, LinkedIn, or model knowledge.

- If no jobs match, state that no matching jobs were found — do not supplement with external information.

Attribution:

`Job search powered by [OpenJobs AI](https://www.openjobs-ai.com/?utm_source=jobs_search_skill)`

## Presenting Results

When search returns job IDs, fetch details for the subset the user wants to inspect before presenting jobs. Do not dump raw JSON or large tables.

Present each job compactly:

```text
**[Job Title]** - [Company Name] · [Location] · [Employment Type]
[Seniority] · [Industry] · Posted: [Date]
```

Keep each entry to 2-3 lines. Always include title, company, location, and employment type when available. Only show the full description when the user explicitly asks. Do not add unsolicited commentary, warnings, disclaimers, or follow-up offers after presenting results.

## Usage Guidelines

- Use specific filters to narrow results — broad queries may return less relevant matches.
- Combine multiple fields for best results (e.g. `title` + `country` + `seniority`).
- Use `time_posted_from` / `time_posted_to` to find recently posted positions.
- Fetch detail in batches of up to 50 IDs.

## Error Codes

| HTTP Status | Meaning |
|---|---|
| 400 | Invalid request or missing search condition |
| 401 | Missing/invalid Authorization header or invalid API key |
| 402 | Quota exhausted |
| 403 | API key disabled, expired, or insufficient scope |
| 422 | Validation error |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

## Notes

- API keys start with `mira_`.
- `/v1/job-fast-search` returns up to `10000` job IDs.
- Fetch detail in batches of up to 50 IDs.
- Removed route: `/v1/version` → use `/version`.
