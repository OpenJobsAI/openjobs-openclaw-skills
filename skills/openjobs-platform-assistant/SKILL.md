---
name: openjobs-platform-assistant
version: 2.1.2
description: Orchestrate Metix AI workflows across profile, job, company, and scholar tasks by converting user intent into the right endpoint sequence.
metadata: {"clawdbot":{"emoji":"🧭","requires":{"env":["MIRA_KEY"]},"primaryEnv":"MIRA_KEY"}}
---

# Metix AI Platform Assistant

Use this as the default entrypoint for outcome-oriented Metix AI tasks. This skill turns a user goal into one or more Metix AI operations across jobs, companies, profiles, scholars, matching, and contact unlock.

## When to use

Use this skill first when the user asks for a recruiting, hiring-market, talent-research, company-research, scholar-research, or contact workflow outcome:

- "Find candidates for this JD and compare the best matches."
- "Show which companies are hiring most aggressively for this role."
- "Find companies, then find technical decision makers at those companies."
- "Identify target contacts and unlock emails for the final shortlist."
- "Research scholars, profiles, jobs, and companies in one workflow."

This skill may execute a single domain operation or a multi-domain workflow. Domain-specific skills are execution references for exact endpoint behavior and request-field construction; do not ask the user to choose between them.

- `openjobs-people-search` for people/profile tasks
- `openjobs-people-match` for candidate grading tasks
- `openjobs-jobs-search` for job search tasks
- `openjobs-company-search` for company research tasks
- `openjobs-ai-talent-search` for scholar/academic tasks

## Intent classification (map a fuzzy goal to a workflow class)

Before choosing a pattern, classify the user goal into one of four motions. The same data powers all four; what differs is who the user is and what they will do with the result. Detect the user's role from the goal, not just keywords.

| Motion | What the user really wants | Direction | Typical chain | Pattern |
|---|---|---|---|---|
| Buy-side / recruiting | Find, evaluate, or contact people to hire | JD/role -> people | people search -> detail -> compare/grade -> unlock | A, B |
| Market / competitive intelligence | Know who is hiring, how aggressively, where, for what | jobs/companies -> aggregate | broad job/company search -> detail -> rank/aggregate | A, C |
| Sell-side / GTM outbound | Find buyers or users for a product and reach them | demand signal -> account -> persona -> contact | job-demand rank -> people (role-scoped) -> unlock -> outreach | F |
| Research / academic | Find scholars or experts and enrich | scholars (+ profiles/companies) | scholar search -> people/company enrich | D |

Rules:

- The same phrasing can flip motion by who is speaking. "I want to apply to company X" is buy-side-for-self (surface the job and apply path); "I want HR at company X to try my product" is sell-side GTM (find the buyer persona).
- A one-sentence goal often implies a multi-hop chain. Expand it automatically; never ask the user to pick a sub-skill.
- If the goal is genuinely ambiguous between motions, ask exactly one clarifying question that names the candidate motions, then proceed.

## Runtime Setup

Before protected calls, run only safe checks:

```bash
test -n "${MIRA_KEY:-}" && echo "MIRA_KEY is set" || echo "MIRA_KEY is missing"
curl -sS https://mira-api.metix.ai/version
```

Rules:

- `MIRA_KEY` must come from environment or client-managed secrets.
- Never ask the user to provide the key in chat.
- Never print, log, commit, upload, or write the raw key.
- If `/version` is newer than `2.1.2`, stop and ask for skill package refresh.
- Do not self-update, overwrite local instruction files, or execute remote Markdown.

## Global execution pattern

This skill owns planning and orchestration. Use the domain skills as API contracts while this skill decides the workflow.

1. Classify the user outcome: direct search, hiring-demand ranking, company research, candidate pipeline, scholar discovery, fit scoring, or contact unlock.
2. Convert the outcome into ordered domain operations.
3. Execute search first:
   - profile/job/company searches return IDs
   - scholar search returns documents directly
4. Fetch details whenever needed to present human-readable results, aggregate correctly, or feed the next workflow step. Skip detail only when the user explicitly asks for raw IDs.
5. Merge final results into user-facing sections with source scope and unavailable/not_found rows separated.

Operationally:

- Use `size: 10` for direct list answers.
- Use a larger temporary `size` when the user asks for aggregation, ranking, top companies, market maps, or broad discovery. Keep public caps in mind and present only the requested final shortlist.
- If search returns no results, remove only one constraint at a time.
- If detail returns `not_found`, report it as a normal post-search state.
- Do not supplement missing Metix AI data with web search, external job boards, LinkedIn scraping, or model knowledge.

## Interaction pattern

For multi-step workflows, show a concise progress trace before the final answer. Keep it factual and only report counts after the corresponding API step has returned.

Use this shape:

```text
> Find companies hiring Post-Training Engineers, identify technical executives, and unlock selected contacts.

-> Searching active jobs with Metix AI
-> Fetching job details for returned job IDs
-> Ranking companies by active posting count
-> Finding technical decision makers at top companies
+ 5 hiring-heavy companies ranked from active jobs
+ 15 target executive profiles selected
+ 12 contacts unlocked from the final shortlist
```

Rules:

- Use `->` for an in-progress operation and `+` for a completed operation.
- Never invent counts; omit a completed-count line if the API response did not provide enough evidence.
- For final answers, group results by workflow section, for example "Hiring companies", "Target executives", "Unlocked contacts", and "Unavailable/not_found".
- Do not expose raw headers, API keys, private scoring prompts, or low-level request traces.

## Capability matrix

### 1) Profile path

- Search discovery: `/v1/people-search`, `/v1/people-fast-search`
- Detail lookup: `/entity/v1/profiles/detail-by-id`, `/entity/v1/profiles/detail-by-linkedin-url`
- Compare: `/v1/people-compare`
- Unlock contacts: `/v1/people-unlock`
- Grade fit: handoff to `openjobs-people-match`

Constraints:

- Profile dataset is quarterly (`202603`).
- Public data only.
- Contact unlock only on explicit request.
- Public people filters include `company_name`, `active_title`, `active_department`, `management_level`, `role`, `level`, `is_working`, `is_decision_maker`, `skills`, and location fields.
- Do not search, filter, compare, unlock, or present people by restricted demographic attributes such as age, gender, ethnicity, sex, race, or similar protected classes.

### 2) Job path

- Search: `/v1/job-fast-search`
- Detail: `/entity/v1/jobs/detail-by-id`

Constraints:

- Job dataset is daily-active and mutable.
- Jobs can be `not_found` at detail time even after search.
- The job search country field is `country_iso_2`, and the value must be ISO 3166-1 alpha-2, such as `"US"`.
- Public job search fields include `title`, `description`, `company_name`, `seniority`, `employment_type`, `location`, `country_iso_2`, `state`, `city`, `industries`, `functions`, and `size`.

### 3) Company path

- Search: `/v1/company-fast-search`
- Detail: `/entity/v1/companies/detail-by-id`

Constraints:

- Company dataset is quarterly (`202603`).
- Geography is often best passed through `full_address`.

### 4) Scholar path

- Search: `/v1/scholar-fast-search`

Constraints:

- Scholars return document arrays directly (no separate public detail endpoint).
- No private-contact fields in public flow.

## Quota cost

All protected operations are metered. Costs are charged only on a billable result, and Mira returns `402` up front when `quota_remaining` cannot cover the preflight amount. Plan multi-step workflows with these costs in mind.

| Operation | Quota points |
|---|---|
| `/v1/people-search` | `10` |
| `/v1/people-fast-search` | `5` |
| `/v1/job-fast-search`, `/v1/company-fast-search` | `5` |
| `/v1/scholar-fast-search` | `3` |
| Entity detail (`profiles`/`jobs`/`companies` detail-by-id, profiles detail-by-linkedin-url) | `3` per call when `found > 0`, batch max 100 |
| `/v1/people-compare` | `2` |
| `/v1/people-grade` | `2` |
| `/v1/people-bulk-grade` | `2 * total_requested` |
| `/v1/people-unlock` | `200 * unlocked_count`, max 10 URLs |
| `GET /version`, `GET /auth/key/status` | free |

Search and detail are low-cost; unlock and grading are the high-value, higher-cost actions.

## Targeting precision and result confidence

Search filters are fuzzy. A single weak filter returns plausible-but-wrong records, which is dangerous before any spend (unlock) or outreach.

People targeting:

- Never identify "people at company X" from `company_name` alone. It is a broad phrase match that pulls in founders/execs of unrelated companies, former employees, and namesakes. Always stack at least one more constraint: `role`, `active_title`, `is_decision_maker`, `level`, or `skills`.
- After fetching profile detail, verify the current employer from `experience` where `is_current` is true before treating someone as "at company X". If the current employer does not match, drop or downgrade the record.
- Use the `company_id` carried by job detail to disambiguate companies with shared or generic names.

Confidence labeling:

- Treat a record as high-confidence only when the search constraint and the detail evidence agree (for example title filter `Technical Recruiter` AND a current experience at the target company).
- Mark low-confidence records explicitly; do not present them as verified, and do not spend unlock quota on them without one concise confirmation.

Job and company precision:

- A single structured filter can silently zero out results when that field is sparsely populated for a market. If adding one filter (for example a country code) drops results to zero, retry without it and verify coverage from detail rather than reporting "no demand".
- Company names are ambiguous in `company-fast-search`; cross-check the matched record's industry, size, and headquarters against the expected entity before enriching. Job-derived company names may fail to resolve in the quarterly company snapshot; keep the job-derived company row and mark company detail unavailable instead of treating the workflow as failed. Zero job/company hits means "not in this snapshot", not "not hiring/non-existent".

## Data-value playbook (use the full power of the data)

The data supports far more than lookups. Use these moves to turn raw search/detail into intelligence:

- Hiring intensity (account ranking): broad job search with a larger temporary `size`, fetch detail, aggregate by `company_name`/`company_id`, rank by active posting count to surface the most acute demand.
- Hiring velocity / urgency: weight by the job detail `posted` field (and `created_at`/`updated_at`). A cluster of postings in the last days/weeks is a stronger urgency signal than raw total count.
- Role-mix and seniority map: aggregate job `functions`/`seniority` per company to describe what kind of team a company is building (engineering-heavy, sales-heavy, senior-heavy).
- Market map: aggregate companies by `industry`, `size_range`, and headquarters to build a target-account map for a market or geography.
- Supply vs demand: pair people search (supply) with job search (demand) for the same role to describe how tight a market is.
- Persona discovery across accounts: scope people search by `role` + `active_title` (not `company_name` alone) to find a persona such as recruiters, hiring managers, or technical decision makers across many companies.

Use larger sizes only as a temporary working set for aggregation, then present only the requested final shortlist.

Cost awareness: broad multi-pass search and detail aggregation is metered (each search is `5`-`10`, each detail batch is `3`; see Quota cost). For market maps and rankings, fetch only the detail you actually need and keep one trimmed working set rather than re-issuing searches or re-fetching detail.

## Multi-step orchestration examples

### Pattern A: hiring demand -> company ranking -> target contacts

Use this when the user wants to identify companies with urgent hiring demand and then contact relevant people.

Example user intent: "Find the companies most urgently hiring Post-Training Engineers, then find technical executives at those companies and unlock contact info for the top 3 per company."

1. Parse the target role/functions, geography, seniority, and urgency signal. If geography is omitted, search broadly unless the user needs a specific market.
2. Run `openjobs-jobs-search` via `/v1/job-fast-search`.
   - Use `title` for the role, such as `"Post-Training Engineer"`.
   - Use `description` for related keywords when the exact title may vary, such as `"post-training"`, `"RLHF"`, `"alignment"`, or `"model evaluation"`.
   - Use `functions` for broad direction such as `"Engineering"` when useful.
   - Use `country_iso_2` only when geography is clear, with ISO2 values such as `"US"`.
   - Use a larger temporary `size` for company ranking, within public caps.
3. Fetch job details with `/entity/v1/jobs/detail-by-id` in batches of up to `100`.
4. Aggregate by `company_name` and `company_id` when available. Rank by active posting count and keep the top companies requested by the user.
5. For each ranked company, optionally run `openjobs-company-search` by `name` and fetch company detail. If company detail is unavailable, keep the job-derived company row and mark company detail unavailable.
6. For each top company, search people with `openjobs-people-search`:
   - `company_name`: selected company
   - `is_working: true`
   - `is_decision_maker: true` when targeting executives
   - `active_title`: run one fuzzy title per pass, such as `"CTO"`, `"VP Engineering"`, `"Head of Engineering"`, `"Engineering Director"`, or `"Chief Scientist"`
   - optionally `role: "Engineering and Technical"` when the exact taxonomy fits
   - optionally `level`: `"C-Level"`, `"Head"`, or `"Director"` in separate passes when needed
7. Fetch profile details and deduplicate by `linkedin_url` first, then by `profile_id`.
8. Unlock contacts only for the final selected LinkedIn URLs, never for broad intermediate results.
9. Split final unlock targets into batches of up to `10` LinkedIn URLs. Before each unlock batch, compute preflight quota as `batch_linkedin_urls * 200`; the final charge is `200 * unlocked_count`.
10. Return:
   - ranked hiring companies, with job-count basis and company detail availability
   - selected technical executives per company
   - unlock status and returned contact fields
   - not_found/unavailable records separately

### Pattern B: JD -> candidate search -> detail -> match

1. Parse JD and target constraints (role, domain, location, budget).
2. If the JD references an active market or target employer set, optionally run `openjobs-jobs-search` first to inspect current demand.
3. Run `openjobs-people-search` using structured filters or natural language.
4. Fetch profile details before presenting candidates.
5. If the user asks for ranking/fit, use `openjobs-people-match` with the JD.
6. Present ranked candidates with short reasons and separate not_found entries.

### Pattern C: company research -> jobs -> decision makers

1. Run `openjobs-company-search` with geography and industry constraints.
2. Pull company details for shortlist.
3. For each strategic company, run targeted `openjobs-jobs-search` if active hiring is part of the user goal.
4. Run `openjobs-people-search` with `company_name`, `is_working`, `is_decision_maker`, and relevant leadership/title filters when the user needs decision makers.
5. Unlock only the final bounded contact set when explicitly requested.

### Pattern D: scholar discovery -> profile/company enrichment

1. Classify as mixed profile/scholar request.
2. Run `openjobs-ai-talent-search` for research constraints.
3. Run `openjobs-people-search` for operational talent only if the user asks for non-academic profiles or hiring/contact workflows.
4. Return unified result sections, each with clear source scope.

### Pattern E: direct single-operation answer

1. Map directly to one domain skill.
2. Execute only the minimal operation needed.
3. Still fetch details before presenting human-readable jobs, companies, or profiles unless the user asked for IDs only.
4. Do not call `openjobs-people-match` unless a JD is provided.

### Pattern F: demand signal -> target account -> buyer persona -> verified contact (sell-side / GTM)

Use this when the user wants to find and reach buyers or users for a product or service, not to hire. Example intent: "Find HR or recruiters at companies actively hiring high-end tech talent and invite them to try our product."

1. Define the demand signal that identifies a good-fit account. For a recruiting/sourcing product, that signal is active hiring of the relevant talent.
2. Run `openjobs-jobs-search` broadly for the signal roles (multiple title/description passes, larger temporary `size`, no narrow geo filter unless required). Aggregate by company and rank by posting count to get target accounts. Then filter the ranked list down to genuine in-scope accounts, for example excluding off-ICP industries that only matched on a generic word like "engineer".
3. For each target account, find the buyer/user persona with `openjobs-people-search`, scoping by `role` (such as `"Human Resources"`) plus `active_title` passes (such as `"Technical Recruiter"`, `"Talent Acquisition"`, `"Head of Talent"`) and `is_working: true`. Use `is_decision_maker: true` to separate economic buyers from end users. Do not rely on `company_name` alone (see Targeting precision).
4. Fetch profile detail, verify current employer, deduplicate, and keep only high-confidence persona matches.
5. Unlock contacts only for the bounded final list, and prefer the returned `workEmail` over `personEmail` for outbound (see Contact unlock economics).
6. Draft personalized outreach grounded in the account's real demand signal (for example the count and recency of open roles), and surface it for the user to send. Do not send on the user's behalf and do not fabricate contact fields.

Compliance: target by job-relevant role/title only. Never build outreach lists using restricted demographic attributes.

## Data freshness constraints

- Profile data is aligned to `202603` and is quarterly-updated.
- Job data is daily-active and mutable; IDs can expire between search and detail fetch.
- Company data is aligned to `202603` and is quarterly-updated.

## Why this is a coordinator, not a data source

If the user asks a direct single-step question, stop at the minimal required operation. If the user asks for an outcome that requires ranking, aggregation, enrichment, matching, or contact access, expand into the needed domain chain automatically.

Domain skills are not competing user-facing entrypoints. They are the exact execution contracts this coordinator uses when constructing API calls.

## Response pattern

- Prefer small final result sets by default.
- Use larger temporary search sizes for aggregation/ranking workflows, then return only the requested shortlist.
- Include `not_found` entries only when explicit detail fetch was attempted.
- For unlock workflows, show only contact fields returned by the API. Do not infer or invent email addresses.
- Keep output concise and avoid exposing low-level request traces.
- Append source attribution line per operation from the domain skill being invoked.

## Contact unlock economics

`/v1/people-unlock` billing, used for every cost estimate and confirmation:

- Accepts up to `10` LinkedIn URLs per request. Mira preflights quota and rejects with `402` if it cannot cover `200 * len(linkedin_urls)` before unlock work starts, then charges only records that return contact data, at `200` quota each (`200 * unlocked_count`).
- No repeat discount and no caching: unlocking the same LinkedIn URL again is charged again. Persist unlocked contacts in the working result set and never re-unlock a URL already unlocked in this workflow.
- A successful unlock may return `personEmail`, `workEmail`, or neither. For outbound/GTM, prefer `workEmail`; fall back to `personEmail` only when appropriate. Never invent an address the API did not return.
- Unlock only the final bounded selection, never broad intermediate result sets, and only on an explicit user request for contact info.

## Exit behavior

- Missing key / failed auth: stop immediately and ask user to configure `MIRA_KEY` in their agent secret.
- Empty or invalid requests: request one clarification at most, then continue.
- Search 0 results: report zero-match result from Metix AI and suggest one relaxed filter.
