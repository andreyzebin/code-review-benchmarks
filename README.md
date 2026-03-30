# Code Review Agent Benchmark

Benchmark system for measuring and regression-testing a code review agent that works with Bitbucket Server.

## How it works

1. Loads a YAML scenario (PR branches, expected findings)
2. Creates a real pull request in Bitbucket from the scenario's branches
3. Calls the agent with the PR — agent works against real Bitbucket
4. Reads back what the agent wrote (comments, review status)
5. LLM-as-judge scores the output against expected findings

## Quickstart

```bash
cd benchmark
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Set required env vars
export BITBUCKET_URL=https://bitbucket.example.com
export BITBUCKET_PROJECT=MYPROJ
export BITBUCKET_REPO=my-repo
export BITBUCKET_TOKEN=...
export BITBUCKET_AGENT_ACCOUNT=agent-bot   # Bitbucket username of the agent under test
export ANTHROPIC_API_KEY=...

# Check scenarios load correctly
python cli.py run --dry-run

# Run all scenarios against your agent
AGENT_API_KEY=... python cli.py run --agent-url http://localhost:8080

# Run a single scenario
python cli.py run --scenario SCEN-009

# Filter by tag
python cli.py run --tag java --tag security

# Compare with previous run
python cli.py run --compare-with last

# A/B test two versions
python cli.py ab --agent-a http://agent-v1:8080 --agent-b http://agent-v2:8080

# Show last run results
python cli.py report last

# Run history
python cli.py history
```

## Configuration

Edit `benchmark/config.yaml`:

```yaml
bitbucket:
  connection:
    base_url: "${BITBUCKET_URL}"
    project: "${BITBUCKET_PROJECT}"
    repo: "${BITBUCKET_REPO}"
    agent_account: "${BITBUCKET_AGENT_ACCOUNT}"
    auth:
      env: BITBUCKET_TOKEN

agent:
  base_url: "http://localhost:8080"
  api_key: "${AGENT_API_KEY}"
  timeout_seconds: 120

judge:
  model: "claude-opus-4-6"
  temperature: 0
```

`${VAR}` placeholders are expanded from environment variables at runtime.

## Example codebase

Scenarios target the **FlowMart order service** — a realistic Spring Boot / Gradle
Java project that exists as a standalone repository.
Mirror or fork it into your Bitbucket instance before running the benchmark:

```
github.com/your-org/flowmart-order-service
```

The repository has one branch per scenario in addition to `main`.
All scenario branches are permanent fixtures — never merge them.

## Scenarios

Each scenario is a YAML file under `benchmark/scenarios/`. It declares a `from_branch`
and a `to_branch` that must already exist in the target Bitbucket repository.
The benchmark opens a PR from `from_branch → to_branch`, triggers the agent,
reads back the comments, and closes the PR.

| ID | Branch | Type | What the agent must catch |
|---|---|---|---|
| SCEN-009 | `feature/ORD-234-buy-3-get-1-free` | bug · security · integrity · N+1 · style | Free item selected by position not cheapest price; missing ownership check; missing `@Transactional`; N+1 promotion query; no Lombok on entity |
| SCEN-010 | `feature/ORD-301-store-credit` | security · correctness | Store credit IDOR (any user redeems any credit); credit deducted from post-tax total instead of pre-tax subtotal |
| SCEN-011 | `hotfix/ORD-287-cancel-npe` | correctness · failure-handling | Null guard on `@OneToMany` hides Hibernate mapping error; silently skips inventory release |

## Adding a scenario

**Step 1 — push the branch.** Add a feature branch to the target repository that
contains the code problem to detect. The branch is a permanent fixture — never merge it.

**Step 2 — write the YAML.** Create `benchmark/scenarios/java/SCEN-NNN-name.yaml`:

```yaml
id: SCEN-NNN
name: "Short description"
tags: [java, security]

input:
  bitbucket:
    provider: real
    pull_request:
      from_branch: "feature/ORD-NNN-short-name"
      to_branch: "main"
      title: "[BENCHMARK] SCEN-NNN: Short description"
      description: |
        ## Jira ticket content / PR description
        Include acceptance criteria here — the agent can read this.

expected_output:
  required_comments:
    - id: EXP-1
      type: inline          # inline | general
      severity: critical    # critical | major | minor
      location: { file: "src/main/java/com/flowmart/orders/service/Foo.java", line: 42 }
      description_keywords:
        - ["keyword1", "alternative"]   # row = OR, rows = AND
      rationale: "Why this comment is required"
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

**Step 3 — verify.**

```bash
python cli.py run --dry-run --scenario SCEN-NNN
```

## Running tests

```bash
cd benchmark
pytest
```

## Project structure

```
benchmark/
├── bitbucket/          # AgentPRView ABC + RealBitbucketFactory (atlassian-python-api)
├── runner/             # scenario loader, agent client, LLM judge, scorer
├── scenarios/
│   └── java/           # YAML scenario definitions (one per PR branch)
├── prompts/            # LLM judge prompt template
├── tests/              # unit tests for judge and Bitbucket proxy
├── cli.py
└── config.yaml
```
