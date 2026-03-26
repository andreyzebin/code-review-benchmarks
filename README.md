# Code Review Agent Benchmark

Benchmark system for measuring and regression-testing a code review agent that works with Bitbucket Server and Jira.

## How it works

1. Loads a YAML scenario (diff, PR metadata, Jira ticket)
2. Starts fake Bitbucket and Jira servers locally
3. Calls the agent with substituted URLs — agent works normally, unaware it's a test
4. Captures everything the agent wrote (comments, review status)
5. LLM-as-judge scores the output against expected findings

## Quickstart

```bash
cd benchmark
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

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
agent:
  base_url: "http://localhost:8080"
  api_key: "${AGENT_API_KEY}"
  timeout_seconds: 120

judge:
  model: "claude-opus-4-6"
  temperature: 0
```

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

Each scenario is a self-contained YAML file with input data and expected findings.

## Adding a scenario

Create a YAML file in `benchmark/scenarios/<language>/SCEN-NNN-name.yaml`:

```yaml
id: SCEN-009
name: "Your scenario name"
tags: [java, bug]

input:
  bitbucket:
    base_provider: fixture
    data:
      pull_request: { id: 1, title: "...", ... }
      diff: [...]
      codebase_context: [...]
  jira:
    base_provider: fixture
    data:
      issue: { key: "PROJ-1", summary: "...", ... }

expected_output:
  required_comments:
    - id: EXP-1
      type: inline          # inline | general
      severity: critical    # critical | major | minor
      location: { file: "src/Foo.java", line: 42 }
      description_keywords:
        - ["keyword1", "keyword2"]  # any of these must appear
      rationale: "Why this comment is required"
  expected_status_change: "NEEDS_WORK"
  thresholds:
    min_score: 0.70
    min_required_found: 1
    max_false_positives: 3
```

## Running tests

```bash
cd benchmark
pytest
```

## Project structure

```
benchmark/
├── fake_servers/       # Fake Bitbucket + Jira (FastAPI)
│   └── providers/      # fixture / live / overlay data providers
├── runner/             # scenario loader, agent client, judge, scorer
├── scenarios/          # YAML test scenarios
├── prompts/            # LLM judge prompt
├── tests/              # unit tests (38 tests)
├── cli.py
└── config.yaml
```
