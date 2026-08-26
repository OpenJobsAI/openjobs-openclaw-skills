# This skill set has moved to Metix AI

The OpenJobs skills are retired. Metix AI is the same team and the same data,
under the name the product ships as now, and the current skills live at
[MetixAI-Official/metix-skills](https://github.com/MetixAI-Official/metix-skills).

Move over in two commands.

## 1. Remove the old skills

```bash
npx skills remove openjobs-people-search openjobs-company-search \
  openjobs-jobs-search openjobs-people-match \
  openjobs-ai-talent-search openjobs-platform-assistant
```

Run `npx skills list` first if you want to see what is installed, and add `-g`
to either command to work on globally installed skills instead of the ones in
the current project. `npx skills remove` on its own opens an interactive picker,
which is the easier route if you are not sure which are present.

You can also just ask your agent:

> Remove every installed skill whose name starts with `openjobs-`, then list
> what skills are still installed so I can check.

## 2. Install the current skills

```bash
npx skills add MetixAI-Official/metix-skills
```

Keep your API key in the `METIX_KEY` environment variable. If you had `MIRA_KEY`
set for the old skills, set `METIX_KEY` to the same value; the key itself does
not change.

## Where the documentation is

<https://platform.metix.ai/docs> covers the search endpoints, the field
reference for people, companies and jobs, worked query examples, and what each
call costs.
