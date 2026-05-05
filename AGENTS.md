# Agent Guide

This document describes the codebase for AI agents and coding assistants.

## What this project does

Benchmarks a code review agent against a set of curated scenarios. Each scenario points at a real branch already published in the test repo (Bitbucket / GitHub). The benchmark creates a PR from that branch, triggers the agent via a configurable strategy, then uses an LLM-as-judge to score whether the agent found the expected issues and made a sound verdict.

**The bench never pushes code.** It only performs metadata operations — create PR, post seed comments, post a trigger comment, fetch back the agent's outputs, decline the PR. All test code (branches with planted bugs, paired step0/step1 branches for incremental review, etc.) must be pre-published in the test repo.

## Repository layout

```
benchmark/
├── bitbucket/
│   ├── base.py             # AgentPRView ABC, AgentPRViewFactory ABC, data classes
│   ├── real_factory.py     # RealBitbucketFactory + RealBitbucketPRProxy
│   └── __init__.py         # build_proxy(cfg) entry point
├── runner/
│   ├── scenario_loader.py  # load_scenarios(), Scenario dataclass, YAML schema
│   ├── agent_client.py     # AgentClient — HTTP POST /review to the agent under test
│   ├── trigger.py          # Trigger ABC, HttpTrigger, WebhookTrigger (Strategy pattern)
│   ├── judge.py            # LLMClient ABC, Judge ABC, LLMJudge, JudgeOutput
│   ├── scorer.py           # score_scenario() → ScenarioResult
│   ├── run.py              # run_scenario() — orchestrates one scenario end-to-end
│   ├── results_store.py    # SQLite + JSON persistence for run history
│   └── html_report.py      # generates self-contained HTML report from results
├── scenarios/
│   ├── java/               # YAML scenario definitions (review + incremental)
│   ├── interaction/        # /help, /ask, unknown-command, dispatcher scenarios
│   └── drafts/             # Loader-skipped specs for not-yet-runnable scenarios
├── prompts/
│   └── judge.txt           # LLM judge prompt template
├── tests/
│   ├── test_bitbucket_proxy.py
│   └── test_judge.py
├── cli.py                  # Typer CLI: run / report / history / ab
├── config.yaml             # Committed defaults (${VAR} placeholders only)
└── config.local.yaml       # Local overrides — gitignored, not committed
```

## Key abstractions

### `AgentPRView` (`bitbucket/base.py`)

Verification-only view of a single benchmark PR. Created by a factory, used as an async context manager.

```python
proxy.pr_id                       # int — Bitbucket PR id
await proxy.get_comments()        # list[CommentThread] — agent account only
await proxy.get_review_status()   # ReviewStatus | None — agent account only
await proxy.add_reviewer(username)# adds a reviewer (used by WebhookTrigger)
await proxy.close()               # declines the PR, called automatically by __aexit__
```

`get_comments()` uses the **activities** endpoint filtered by `action == "COMMENTED"`.
`get_review_status()` reads the participants endpoint and filters by agent slug.
Both methods return only activity from the configured `agent_account`.

### `AgentPRViewFactory` (`bitbucket/base.py`)

```python
proxy = await RealBitbucketFactory.build(cfg)
```

`cfg` must contain `provider`, `connection` (from config), and `pull_request` (from scenario).
`build_proxy(cfg)` in `__init__.py` dispatches on `cfg["provider"]`.

The factory also applies SSL config from `connection.ssl`:
- `ca_cert` — overrides session verify with a custom CA bundle
- `client_cert` / `client_key` — mutual TLS client certificate (PEM)

### `Trigger` (`runner/trigger.py`)

Strategy for activating the agent under test. Selected from `config.yaml` via `agent.trigger`.

```python
class HttpTrigger(Trigger):
    # POSTs to agent's /review endpoint with pr_id
    async def activate(proxy): await agent_client.run(pr_id=proxy.pr_id)

class WebhookTrigger(Trigger):
    # Adds agent_account as PR reviewer, then sleeps timeout_seconds
    async def activate(proxy): await proxy.add_reviewer(agent_account); sleep(timeout)
```

Use `http` when the agent has a direct HTTP endpoint.
Use `webhook` when the agent is already wired to Bitbucket via `PR_REVIEWER_UPDATED` webhook.

### `LLMJudge` (`runner/judge.py`)

`AgentPRView` is injected at construction time. `evaluate(scenario)` fetches comments and
review status from the view internally — callers do not pass them.

```python
judge = LLMJudge(llm_client, proxy)
output: JudgeOutput = await judge.evaluate(scenario)
# output.comments, output.review_status, output.score, output.summary
```

Two LLM clients available:
- `AnthropicLLMClient` — uses `ANTHROPIC_API_KEY`
- `OpenAILLMClient(model, api_url, api_key)` — any OpenAI-compatible endpoint

Selected by config: if `judge.api_url` is set → `OpenAILLMClient`; otherwise `AnthropicLLMClient`.

### `run_scenario` (`runner/run.py`)

```
build_proxy(cfg)            # create PR in Bitbucket
  └─ trigger.activate(proxy)  # HttpTrigger: POST /review  |  WebhookTrigger: add reviewer + sleep
  └─ judge.evaluate(scenario) # fetch comments, score with LLM
  └─ score_scenario(...)      # → ScenarioResult
  └─ proxy.close()            # decline the PR (via __aexit__)
```

## Configuration

`config.yaml` contains committed defaults with `${VAR}` placeholders.
`config.local.yaml` (gitignored) is deep-merged on top at runtime — write only the keys you want to override.

```yaml
# config.local.yaml example
bitbucket:
  connection:
    base_url: "https://bitbucket.mycompany.com"
    project: "MYPROJ"
    repo: "orderflow"
    agent_account: "review-bot"
    ssl:
      ca_cert: "/path/to/ca.crt"
      client_cert: "/path/to/client.pem"

agent:
  trigger: "webhook"       # http | webhook
  timeout_seconds: 120

judge:
  model: "deepseek-chat"
  api_url: "https://api.deepseek.com/v1"
  api_key: "${DEEPSEEK_API_KEY}"
```

Secrets go in `.env` (gitignored), sourced before running:

```bash
source .env
python cli.py run --scenario SCEN-009
```

## Scenario YAML format

```yaml
id: SCEN-NNN
name: "Human-readable name"
tags: [java, security, severity:critical]

input:
  bitbucket:
    provider: real
    pull_request:
      from_branch: "feature/PROJ-NNN-short-name"
      to_branch: "master"
      title: "[BENCHMARK] SCEN-NNN: Short description"
      description: |
        ## Jira ticket content — the agent reads this

expected_output:
  required_comments:
    - id: EXP-1
      type: inline          # inline | general
      severity: critical    # critical | major | minor
      location:
        file: "src/Foo.java"
        line: 42
      description_keywords:
        - ["keyword1", "alt1"]   # row = OR, rows = AND
      rationale: "Why the agent must raise this"
  forbidden_comments:
    - description: "Topic the agent must not raise"
  expected_status_change: "NEEDS_WORK"  # NEEDS_WORK | APPROVED
  thresholds:
    min_score: 0.70
    min_required_found: 1
    max_false_positives: 3

metadata:
  difficulty: medium        # easy | medium | hard
  language: java
  pr_size: small            # small | medium | large
  scenario_type: bug        # bug | security | design | performance | style | test_coverage
```

`description_keywords` logic: each row is a list of alternatives (OR). All rows must match (AND).
Matching is semantic, not literal substring.

## CLI reference

```bash
python cli.py run --agent-url http://localhost:8080
python cli.py run --scenario SCEN-009
python cli.py run --tag security
python cli.py run --dry-run
python cli.py run --no-verify-ssl          # skip TLS cert verification
python cli.py run --compare-with last
python cli.py ab --agent-a http://v1:8080 --agent-b http://v2:8080
python cli.py report last                  # terminal table
python cli.py report last --html           # HTML file, opens in browser
python cli.py history
```

## Testing

```bash
cd benchmark
pytest
```

Tests use `CapturingLLMClient` (records prompt) and `MockProxy(AgentPRView)` (returns fixed
comments/status). No real Bitbucket or LLM calls are made.

## Adding a new scenario

1. Push a feature branch to the target repo containing the bug/issue to detect
2. Create `benchmark/scenarios/java/SCEN-NNN-short-name.yaml` following the format above
3. Verify: `python cli.py run --dry-run --scenario SCEN-NNN`
4. Run: `python cli.py run --scenario SCEN-NNN`

### Test-code authoring rules

The agent reads the diff, comments, AGENTS.md of the test repo, and
arbitrary source files at will. Anything visible to the agent is fair
game for it to use as evidence. So:

- **Never** mention the scenario or its expected output in code,
  commit message, comments, javadoc, or PR description. A line like
  `// BUG: missing null guard` or `// intentional N+1 for SCEN-304`
  hands the answer to the agent and invalidates the test.
- The PR description should read like a real ticket: ID, motivation,
  acceptance criteria. No mentions of "we are testing X" or "this PR
  is part of a benchmark scenario".
- Planted bugs must be visible only through code reading (a missing
  annotation, a wrong-index `get(0)`, a per-iteration repository call).
  No marker comments, no hint variable names like `buggyHelper`.
- "Clean" PRs (false-positive resistance scenarios) must be genuinely
  clean. If the agent finds a real inaccuracy in the diff that you
  intended to be no-bug, the bug is yours — fix the test code, don't
  forbid the finding in `forbidden_comments`.

### Drafts

Scenarios that need bench-side machinery that doesn't exist yet
(BranchUpdater, multi-repo connection_override, etc.) live under
`scenarios/drafts/`. The loader skips that directory entirely so a
draft never accidentally runs and fails on missing branches. Each
draft carries a `metadata.status` field and a "Bench-side TODO" block
listing what needs to land before promotion.

### Cost tags

Scenarios tagged `cost:expensive` use spawn_many with several
investigators, two-round flows, large diffs, or extra repos — they
dominate wall-clock and token cost on a matrix run. Useful filters:

```bash
python cli.py run                              # all 11 scenarios
python cli.py run -t cost:expensive            # only the heavy ones
# (no native exclude flag; for "everything except expensive" use a
#  compound tag like 'java' that the heavy ones don't carry, or
#  --scenario for explicit lists)
```

## Aggregation across attempts

LLM judges and qwen-class agents both have non-trivial variance.
A single run can land 0.55 vs 0.75 on the same scenario back-to-back.

```bash
python cli.py run --repeat 3 -p qwen3-6
```

Each scenario runs N times, each attempt gets its own
`attempt-NN/result.json`, and the bench summary line shows the median
plus the `[min..max]` window. Verdict aggregation: pass when at least
half the attempts passed; error only when every attempt errored.
Warnings (scenario_warnings + agent_warnings) are unioned across
attempts, deduped by `(kind, detail)` so a flaky judge that emits a
warning on 2 of 5 attempts still surfaces it.

## Judge output streams

Three independent signals from the judge per scenario:

- **score / required_comments / false_positives** — the binary scorecard.
  Score and verdict feed the pass/fail call.
- **scenario_warnings** — flags about the *test design itself*: leaky
  description, unfulfillable expectation, contradiction with seed
  comments, trigger mismatch, other. Critique the scenario, not the
  agent.
- **agent_warnings** — flags about the *agent's reasoning quality*,
  independent of scoring: wrong-location, wrong-reasoning,
  surface-acceptance, contradicts-codebase, methodology-gap,
  interface-violation (no `[bot_user]` prefix or no dg footer), other.

`agent_warnings` does not affect overall_score — it surfaces the kinds
of concerns a binary scorecard misses. Two prompts that both score 0.7
can produce very different `agent_warnings` shapes; that's the
discriminator for prompt-comparison work.

The judge is fed the actual PR diff and AGENTS.md (when available) so
its `wrong-location` / `contradicts-codebase` / `methodology-gap`
calls are grounded in code, not asserted from world knowledge alone.
Both blobs are size-capped (~30k diff, ~10k AGENTS.md) before being
substituted into the prompt.

Reasonable findings the agent posts that aren't in `expected_output`
are NOT a quality regression — they're just out-of-scope-for-the-test.
The default agent strictness policy ("APPROVED unless a BLOCKER/MAJOR
stands") absorbs this gracefully.
