# BHUSA-DAC — Detection as Code

Elastic Security detection rules managed as code. The repository is the source of
truth; rules are validated on pull request and deployed to Elastic on merge to `main`.

```
 submit ──▶ n8n DAC Agent ──▶ branch + commit + PR ──▶ CI validates ──▶ human approves
                  ▲                                        │                    │
                  └────────── fix loop (max 3) ◀────────────┘                 merge
                                                                                 │
                                                             deploy.yml ──▶ Elastic
                                                                                 │
                                            alert fires ──▶ webhook ──▶ SOC Triage Agent
```

## Layout

| Path | What it is |
|---|---|
| `rules/*.yml` | One detection rule per file. The deployable artifact. |
| `docs/rule-standard-v1.yaml` | The template. Copy it to author a rule by hand. |
| `schemas/rule.schema.json` | JSON Schema the CI validates every rule against. |
| `.github/workflows/validate.yml` | Runs on PR. The merge gate. |
| `.github/workflows/deploy.yml` | Runs on merge to `main`. Pushes to Elastic. |

## Submitting a rule

**Via the DAC Agent (normal path).** Upload a filled copy of
`docs/rule-standard-v1.yaml` to the n8n DAC Agent form, or let the Detection
Engineering Agent hand a draft to it. The agent validates the rule, mints the
`rule_id`, opens a branch and a pull request, and posts the PR back into the
originating Slack thread.

An uploaded file is used **verbatim** — it is never rewritten by the model. If it
fails validation you get the errors back and nothing is pushed.

**By hand.** Copy the template into `rules/`, generate a `rule_id`
(`python3 -c "import uuid; print(uuid.uuid4())"`), and open a PR.

## What CI checks

`validate.yml` runs four checks, all of which must pass before merge:

1. **`rule_id`** — present and a valid UUID4.
2. **Schema** — validated against `schemas/rule.schema.json`.
3. **ES|QL conventions** — non-aggregating queries (no `STATS`) must carry
   `METADATA _id, _index, _version` in the `FROM` clause and `KEEP` those three
   fields. Without them Elastic cannot de-duplicate alerts or link them back to
   the source document.
4. **ES|QL runtime execution** — the query is actually run against Elastic with
   `| LIMIT 0` appended. This catches syntax errors, unknown fields (Elastic
   returns "did you mean" suggestions), and index patterns that resolve to
   nothing.

Check 4 deliberately runs with `ELASTIC_API_KEY` — the same identity that will
own the deployed rule. A rule created by a key that cannot read its own index
deploys "successfully" and then sits in `partial failure` forever. Validating
with a more privileged key would pass and tell you nothing.

If validation fails on a PR the agent opened, the failure is posted back to n8n,
which pushes a fix commit to the same branch. This is capped at 3 attempts.

## Approval

`main` is protected by the `main-protection` ruleset: pull request required,
`Validate Rules` must pass, and one approving review is required.

Note that GitHub's "an author cannot approve their own PR" protection keys on the
**PR author**, which for agent-submitted rules is the n8n bot rather than the
person who requested the rule. The DAC PR Notifier flags in Slack when a PR is
approved by whoever requested it.

## Deployment

Merging to `main` triggers `deploy.yml`, which creates or updates each rule via
the Elastic detection engine API and attaches the `n8n-alert-triage` webhook
action to every one of them. That action is what makes an Elastic alert reach the
SOC Triage Agent directly, in seconds.

The action is defined once in `deploy.yml` rather than in each rule's YAML, so
there is nothing to drift and retargeting the webhook is a one-line change.
`actions:` in a rule file must stay empty — the schema enforces it.

## Conventions

| Field | Convention |
|---|---|
| `name` | `SOC-UC-{NNN}-{OS/Datasource}-{Short Description}` |
| `risk_score` | low 21 · medium 47 · high 73 · critical 99 |
| `interval` / `from` | `from` = interval + 5m (`5m` → `now-10m`) |
| `type` / `language` | `esql` only |

Valid index targets:

```
logs-endpoint.events.process-default
logs-endpoint.events.file-default
logs-endpoint.events.network-default
logs-endpoint.events.security-default
```

## Required secrets

| Secret | Purpose |
|---|---|
| `ELASTIC_URL` | Kibana base URL, including the space path |
| `ELASTIC_API_KEY` | Deploys rules and runs the ES\|QL check. Needs read on the target indices — see check 4 above. |
| `ELASTICSEARCH_URL` | Elasticsearch host for `_query`. Different host from `ELASTIC_URL`. |
| `N8N_DAC_WEBHOOK_URL` | Where CI reports validation failures for the fix loop |
