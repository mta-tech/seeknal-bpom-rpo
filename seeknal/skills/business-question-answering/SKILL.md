---
name: business-question-answering
description: "Translate business questions into metrics, SQL evidence, and actionable recommendations"
tags: [business, metrics, recommendation, analysis]
version: "1.0.0"
---

# Business Question Answering

Use this workflow for questions like "where is revenue strongest?", "what are
the opportunities?", "which segment should we prioritize?", or "what changed?".

## Workflow

1. **Clarify silently when possible**
   - Infer likely metric, dimension, grain, and time window from available schema and context.
   - Ask the user only when multiple materially different business definitions would change the answer.

2. **Get evidence**
   - If the data lives in a connected database, load `database-analyst` and use `list_tables`/`describe_table`/`execute_sql`.
   - Before writing ad-hoc SQL, always call `list_sql_pairs` to check if an authoritative SQL pair already exists for the question.
   - When a SQL pair matches the user's question (by intent or semantic similarity), you MUST call `execute_sql_pair` to run the pair's SQL as-is. Do NOT rewrite or substitute columns from the pair.
   - Only fall back to `execute_sql` when no matching SQL pair exists.
   - Do not answer quantitative questions without at least one SQL query when SQL tools are available.

3. **Validate**
   - Run a total/check query when recommendations depend on shares or rankings.
   - Compare at least two cuts when the user asks for opportunities, e.g. segment and region, or current and prior period.

4. **Recommend**
   - Separate facts from interpretation.
   - Tie every recommendation to a number from the query result.
   - Include caveats about sample size, filters, and grain.

## Output shape

Use this compact structure unless the user asks otherwise:

1. **Answer** — one sentence with the main finding.
2. **Evidence** — small table or bullets with actual numbers.
3. **Recommendation** — 1–3 actions.
4. **Caveat / next check** — one line.
