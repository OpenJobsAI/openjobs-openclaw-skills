# OpenJobs AI — OpenClaw Skills

A collection of [OpenClaw](https://github.com/clawdbot/openclaw) skills for recruiting and talent sourcing, powered by the [OpenJobs AI](https://www.openjobs-ai.com/) API.

## What are OpenClaw Skills?

OpenClaw skills are Markdown-based instruction files that extend AI assistants (like Claude) with domain-specific capabilities. Each skill teaches the AI how to interact with a specific API or workflow.

## Skills

### 🔍 [openjobs-people-search](./openjobs-people-search/SKILL.md)

Search, discover, and retrieve professional candidate profiles from the OpenJobs AI database.

**Capabilities:**
- Search candidates using structured filters (skills, location, experience, industry, etc.)
- Look up full profiles by LinkedIn URL (up to 50 at once)
- Compare multiple candidates side by side
- Analyze talent pool statistics and distributions
- Unlock candidate contact information (email addresses)

---

### 🎯 [openjobs-people-match](./openjobs-people-match/SKILL.md)

Evaluate candidate-job fit using the OpenJobs AI grading model.

**Capabilities:**
- Score a single candidate CV against a job description (0–100 rating)
- Bulk-grade multiple LinkedIn profiles against one JD and rank them by fit score

---

## Requirements

- A **Mira API key** (`mira_...`) from [platform.openjobs-ai.com](https://platform.openjobs-ai.com/)

## License

Apache 2.0