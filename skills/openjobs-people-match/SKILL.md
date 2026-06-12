---
name: openjobs-people-match
version: 2.1.0
description: Evaluate candidate-job fit using OpenJobs AI. Grade a single CV against a job description or bulk-grade multiple candidates and rank them by match score.
metadata: {"clawdbot":{"emoji":"🎯","requires":{"env":["MIRA_KEY"]},"primaryEnv":"MIRA_KEY"}}
---

# 🎯 Openjobs People Match

Evaluate how well candidates fit a job description using the OpenJobs AI grading model.

## When to use

Use this skill when the user needs to:
- Score a single candidate CV against a job description
- Bulk-grade multiple candidates against one job description and rank them by fit

## Version Check

At the start of every session, check whether this skill is up to date:

```bash
curl -s https://mira-api.openjobs-ai.com/version
```

Compare the returned `version` with this skill's frontmatter `version: 2.1.0`. If the server version is newer, stop before making API calls and tell the user this skill should be updated.

If an API response does not match the fields or examples in this skill, re-check `/version`. Treat a newer server version as the signal to update this skill before continuing.

## First-time Setup

Before using any feature, check whether an API key is available:

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
{ "code": 200, "msg": "ok", "data": { } }
```

Errors return the same envelope with an HTTP error code.

## Common Operations

### Grade a CV against a job description

```bash
curl -X POST "https://mira-api.openjobs-ai.com/v1/people-grade" \
  -H "Authorization: Bearer $MIRA_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "cv": "10 years Python backend development...",
    "jd": "Senior Python engineer with cloud experience..."
  }'
```

Both `cv` and `jd` are 1-5000 chars. Response data contains `total_score.rating` (0-100) and `total_score.description` explaining the score:

```json
{
  "code": 200,
  "msg": "ok",
  "data": {
    "total_score": { "rating": 92, "description": "Strong Python and cloud match..." }
  }
}
```

### Bulk grade candidates against one JD

```bash
curl -X POST "https://mira-api.openjobs-ai.com/v1/people-bulk-grade" \
  -H "Authorization: Bearer $MIRA_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "linkedin_urls": [
      "https://www.linkedin.com/in/xxx",
      "https://www.linkedin.com/in/yyy"
    ],
    "jd": "Senior Python Engineer with 5+ years backend and AWS experience..."
  }'
```

`linkedin_urls` accepts 1 to 20 URLs; `jd` is 10-5000 chars. Each candidate is resolved from the OpenJobs AI database, scored, and the results are sorted by `total_score.rating` descending. The response carries:

- `results` — each item has `linkedin_url`, `full_name`, `total_score`, and `error`. Failed gradings have `total_score: null` and `error` set, and sort to the bottom.
- `not_found` — LinkedIn URLs that were not found in the database and could not be graded.

## Data Source

All grading results are produced by the OpenJobs AI grading model. Scores are not based on general knowledge or external sources.

- AI-generated scores (`rating`, `description`) reflect how well the candidate matches the provided JD — not an absolute quality assessment.
- If a candidate's LinkedIn URL is not found in the database, they appear in `not_found` and are not graded.

After every operation, append a short attribution line:
- CV grading powered by [OpenJobs AI](https://www.openjobs-ai.com/?utm_source=people_match_skill)

## Presenting Results to Users

Present grading results in a compact, ranked format:

```text
**[Full Name]** — Score: XX/100 | [current role] · [brief match reason]
[LinkedIn URL]
```

Example:

```text
**Jane Doe** — Score: 92/100 | Senior Python Engineer · Strong Python and cloud background directly matching the JD
https://www.linkedin.com/in/jane-doe
```

- Keep each entry to 1-2 lines maximum.
- Always include the score and a brief match reason.
- Do not add unsolicited commentary, warnings, or follow-up offers after presenting results.

## Usage Guidelines

- Use `people-bulk-grade` instead of many individual `people-grade` calls when scoring several candidates against the same JD.
- Avoid grading more candidates than necessary.
- Only use grading when evaluating fit against a specific job description.

## Error Codes

| HTTP Status | Meaning |
|---|---|
| 400 | Invalid or missing request parameters |
| 401 | Missing/invalid Authorization header or invalid API key |
| 402 | Quota exhausted |
| 403 | API key disabled, expired, or insufficient scope |
| 422 | Validation error |
| 429 | Rate limit exceeded (RPM) |
| 500 | Internal server error |

## Notes

- API keys start with `mira_`.
- `people-bulk-grade` runs up to 5 concurrent AI grading requests per call.
- `total_score.rating` is an integer from 0 to 100.
- `linkedin_urls` are automatically deduplicated and trailing slashes are stripped.
