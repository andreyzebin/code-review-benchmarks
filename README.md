# Code Review Agent Benchmark

Benchmark system for measuring and regression-testing a code review agent that works with Bitbucket Server and Jira.

## How it works

1. Loads a YAML scenario (PR branches, expected findings)
2. Creates a real pull request in Bitbucket from the scenario's branches
3. Calls the agent with the PR — agent works against real Bitbucket, finds the Jira ticket on its own
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
export ANTHROPIC_API_KEY=...

# Check scenarios load correctly
python cli.py run --dry-run

# Run all scenarios against your agent
AGENT_API_KEY=... python cli.py run --agent-url http://localhost:8080

# Run a single scenario
python cli.py run --scenario SCEN-001

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

## Scenarios

8 built-in scenarios in `benchmark/scenarios/`:

| ID | Type | Language | What's tested |
|---|---|---|---|
| SCEN-001 | bug | Java | NPE from unguarded Optional |
| SCEN-002 | security | Python | SQL injection via f-string |
| SCEN-003 | design | Java | God class / SRP violation |
| SCEN-004 | performance | Python | N+1 queries in loop |
| SCEN-005 | test_coverage | TypeScript | New function without tests |
| SCEN-006 | style | Java | Clean code — agent should approve |
| SCEN-007 | bug | Python | Off-by-one in pagination |
| SCEN-008 | security | Java | Hardcoded AWS credentials |

Each scenario requires a matching feature branch to already exist in the target repository.

## Adding a scenario

Create a YAML file in `benchmark/scenarios/<language>/SCEN-NNN-name.yaml`:

```yaml
id: SCEN-009
name: "Your scenario name"
tags: [java, bug]

input:
  bitbucket:
    provider: real
    pull_request:
      from_branch: "feature/PROJ-900-your-feature"
      to_branch: "main"
      title: "[BENCHMARK] SCEN-009: Your scenario"

expected_output:
  required_comments:
    - id: EXP-1
      type: inline          # inline | general
      severity: critical    # critical | major | minor
      location: { file: "src/Foo.java", line: 42 }
      description_keywords:
        - ["keyword1", "keyword2"]  # any of these must appear
      rationale: "Why this comment is required"
  forbidden_comments:
    - description: "Comment topic the agent must not raise"
  expected_status_change: "NEEDS_WORK"
  thresholds:
    min_score: 0.70
    min_required_found: 1
    max_false_positives: 3

metadata:
  difficulty: medium
  language: java
  pr_size: small
  scenario_type: bug
```

## Running tests

```bash
cd benchmark
pytest
```

## Project structure

```
benchmark/
├── bitbucket/          # BitbucketPRProxy ABC + RealBitbucketFactory
├── runner/             # scenario loader, agent client, LLM judge, scorer
├── scenarios/          # YAML test scenarios
│   ├── java/
│   ├── python/
│   └── typescript/
├── prompts/            # LLM judge prompt template
├── tests/              # integration tests for judge
├── cli.py
└── config.yaml
```
