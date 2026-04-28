# seeknal-bpom-rpo-1 Claude Guide

Use `AGENTS.md` as the canonical project guidance. This file exists for Claude
Code and tools that look for `CLAUDE.md`.

## Common commands

```bash
seeknal source list --project .
seeknal source inspect <source> --project .
seeknal source sync <source> --project .
seeknal source test <source> --project .
seeknal ask chat --project .
seeknal ask test --project . --sql-only
seeknal ask test --project .
```

## Mode setup quick reference

### Tap-in / read-only connected-source analyst

Use this when the project should ask questions against an existing database:

```bash
cp .env.example .env
# edit .env and set WAREHOUSE_URL
seeknal source connect warehouse --connector postgresql --dsn-env WAREHOUSE_URL --project .
seeknal source sync warehouse --project .
seeknal source test warehouse --project .
seeknal ask chat --project .
```

Then fill `SEEKNAL_ASK.md`, `context/*.md`, `seeknal/sql_pairs/*.yml`, and
`seeknal/tests/*.yml` with reusable business definitions and QA cases.

### Data pipeline builder

Use this when Seeknal should build managed data assets:

```bash
# edit profiles.yml for pipeline credentials
seeknal draft source <name>
seeknal dry-run draft_source_<name>.yml
seeknal apply draft_source_<name>.yml
seeknal plan
seeknal run
```

Keep source/transform/feature-group/model definitions under `seeknal/`, and
keep `target/` as generated output only.

### Hybrid

Use this when both are present. Keep `mode.default: auto` in
`seeknal_agent.yml`; register read-only connected sources with `seeknal source
connect`, and keep managed pipeline definitions under `seeknal/`. Use SQL
pairs and Ask tests to make source selection explicit for important business
questions.

## Ask project assets

- `seeknal_agent.yml` steers Ask mode and connected read-only sources.
- `.env` stores source DSNs and LLM keys locally; `.env.example` stores only
  safe placeholder variable names.
- `SEEKNAL_ASK.md` is loaded into Ask sessions as durable project instructions.
- `.seeknal/context/sources/` contains generated source context from
  `seeknal source sync`; do not hand-edit it.
- `context/` contains user-taught memory saved from chat; `context/sql_pairs/`
  is for reusable SQL examples the agent saves when instructed.
- `seeknal/sql_pairs/` contains human-curated examples the agent can read as context.
- `preferences.yml` contains short remembered rules saved from chat.
- `seeknal/tests/` contains executable prompt-to-SQL QA cases.

## Setup checklist for a connected-source project

1. Put secrets in `.env`; keep only placeholder names in committed files.
2. Run `seeknal source connect ... --dsn-env <ENV_VAR> --project .`.
3. Run `seeknal source sync <source> --project .` to generate table context.
4. Fill `SEEKNAL_ASK.md`/`context/*.md` with business vocabulary and caveats.
5. Add reusable SQL examples to `seeknal/sql_pairs/*.yml`.
6. Add executable QA cases to `seeknal/tests/*.yml`.
7. Run `seeknal ask test --project . --sql-only`.
8. Run `seeknal ask chat --project .` for a real multi-turn smoke test.

When answering data questions, prefer source context, project memory, and SQL
pairs before ad-hoc schema probing. When a stored SQL pair directly matches a
simple known business question, execute it as authoritative and answer from the
result. Continue with SQL/Python/modeling only when the user asks for deeper or
changed analysis. When the user says "remember" or "save this", persist only
non-secret reusable project knowledge. When validating behavior, run Ask tests
rather than inventing one-off checks.
