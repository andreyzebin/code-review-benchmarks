# Agent Guide

This document describes the codebase for AI agents and coding assistants.

## What this project does

Benchmarks a code review agent against a set of curated scenarios. Each scenario has a real feature branch in Bitbucket. The benchmark creates a PR from that branch, triggers the agent, then uses an LLM-as-judge to score whether the agent found the expected issues.

## Repository layout

```
benchmark/
├── bitbucket/              # Bitbucket integration
│   ├── base.py             # BitbucketPRProxy ABC, BitbucketFactory ABC, data classes
│   ├── real_factory.py     # RealBitbucketFactory + RealBitbucketPRProxy
│   └── __init__.py         # build_proxy(cfg) entry point
├── runner/
│   ├── scenario_loader.py  # load_scenarios(), Scenario dataclass, YAML schema
│   ├── agent_client.py     # AgentClient — HTTP call to the agent under test
│   ├── judge.py            # LLMClient ABC, Judge ABC, LLMJudge, JudgeOutput
│   ├── scorer.py           # score_scenario() → ScenarioResult
│   ├── run.py              # run_scenario() — orchestrates one scenario end-to-end
│   └── results_store.py    # SQLite persistence for run history
├── scenarios/              # YAML scenario definitions
│   ├── java/
│   ├── python/
│   └── typescript/
├── prompts/
│   └── judge.txt           # LLM judge prompt template
├── tests/
│   └── test_judge.py       # Judge integration tests (pytest-asyncio)
├── cli.py                  # Typer CLI (run / report / history / ab)
└── config.yaml             # Runtime configuration
```

## Key abstractions

### `BitbucketPRProxy` (`bitbucket/base.py`)

Verification-only view of a single PR. Created by a factory, used as an async context manager.

```python
proxy.pr_id                    # int — Bitbucket PR id
await proxy.get_comments()     # list[CommentThread]
await proxy.get_review_status()  # ReviewStatus | None
await proxy.close()            # called automatically by __aexit__
```

`close()` declines/cleans up the PR after the benchmark run.

### `BitbucketFactory` (`bitbucket/base.py`)

```python
proxy = await RealBitbucketFactory.build(cfg)
```

`cfg` must contain `provider`, `connection` (from `config.yaml`), and `pull_request` (from the scenario). `build_proxy(cfg)` in `__init__.py` dispatches on `cfg["provider"]`.

### `Judge` / `LLMJudge` (`runner/judge.py`)

```python
output: JudgeOutput = await judge.evaluate(scenario, comments, review_status)
```

`LLMJudge` delegates to an injected `LLMClient`:

```python
class LLMClient(ABC):
    def complete_json(self, prompt: str) -> dict: ...
```

`_build_prompt` and `_interpret` are pure functions — easy to test without mocking.

### `run_scenario` (`runner/run.py`)

```
build_proxy(cfg)
  └─ agent_client.run(pr_id)        # trigger the agent under test
  └─ proxy.get_comments()           # read what the agent wrote
  └─ proxy.get_review_status()
  └─ judge.evaluate(...)            # score with LLM-as-judge
  └─ score_scenario(...)            # → ScenarioResult
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
      to_branch: "main"
      title: "[BENCHMARK] SCEN-NNN: Short description"

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

The `connection` block (base_url, project, repo, auth) is **not** in scenario files — it lives in `config.yaml` under `bitbucket.connection` and is injected at runtime.

## `config.yaml` structure

```yaml
bitbucket:
  connection:
    base_url: "${BITBUCKET_URL}"
    project: "${BITBUCKET_PROJECT}"
    repo: "${BITBUCKET_REPO}"
    auth:
      env: BITBUCKET_TOKEN

agent:
  base_url: "http://localhost:8080"
  api_key: "${AGENT_API_KEY}"
  timeout_seconds: 120

judge:
  model: "claude-opus-4-6"
  temperature: 0
  prompt_template: "prompts/judge.txt"

results:
  store_path: "results"
  db_path: "results/benchmark.db"
```

`${VAR}` placeholders are expanded from environment variables.

## Data flow for `description_keywords`

Each entry in `description_keywords` is a list of alternatives (OR). All entries must be satisfied (AND). The judge checks these semantically, not as literal string matches.

Example: `[["null", "NPE"], ["Optional", "check"]]` means the agent comment must mention null/NPE **and** Optional/check.

## Testing

```bash
cd benchmark
source .venv/bin/activate
pytest
```

Tests use `CapturingLLMClient` (records the prompt) and `MockProxy` (returns fixed comments). No real Bitbucket or LLM calls are made in tests. The prompt is printed to stdout so you can inspect it manually.

## Adding a new scenario

1. Push a feature branch to the target repo that contains the bug/issue to detect
2. Create `benchmark/scenarios/<language>/SCEN-NNN-short-name.yaml` following the format above
3. Run `python cli.py run --dry-run` to verify the YAML loads correctly
4. Run the scenario: `python cli.py run --scenario SCEN-NNN`
