# Code Review Agent Benchmark

Measures and regression-tests a code review agent that works against Bitbucket Server.
Each scenario is a real pull request — the benchmark opens it, triggers the agent, reads
back what the agent wrote, and scores it with an LLM judge.

## How it works

```
Scenario YAML
    │  from_branch / to_branch
    ▼
Bitbucket Server  ◄──────────────────────────────────────────────┐
    │  open PR                                                    │
    ▼                                                             │
Agent under test  ──── reviews PR, posts comments ───────────────┘
    │
    ▼
Benchmark reads comments + review status (agent account only)
    │
    ▼
LLM judge scores against expected_output in scenario YAML
    │
    ▼
Pass / Fail  +  detailed report
```

---

## Quickstart

### 1 — Clone and run the interactive setup wizard

```bash
git clone https://github.com/andreyzebin/code-review-benchmarks
cd code-review-benchmarks
./setup.sh
```

The wizard will ask you for:

| Prompt | Example |
|---|---|
| Bitbucket Server URL | `https://bitbucket.example.com` |
| Project key | `MYPROJ` |
| Repository slug | `orderflow` |
| Personal access token | `ATBBxxxxxxxx` |
| Agent bot account username | `review-bot` |
| Agent base URL | `http://localhost:8080` |
| Judge LLM (Anthropic or OpenAI-compatible) | `1` |
| Anthropic API key | `sk-ant-…` |

It will also print the exact `git push` commands to mirror the example repository
into your Bitbucket instance (see step 2 below).

At the end it writes a `.env` file, installs Python dependencies, and runs
`cli.py run --dry-run` to confirm scenarios load correctly.

---

### 2 — Mirror the example repository

The scenarios target the **FlowMart order service** — a Spring Boot / Gradle Java
project at [`andreyzebin/orderflow`](https://github.com/andreyzebin/orderflow).
Mirror it into the Bitbucket project you configured above:

```bash
git clone --mirror https://github.com/andreyzebin/orderflow.git orderflow-mirror
cd orderflow-mirror
git remote add bitbucket https://bitbucket.example.com/scm/myproj/orderflow.git
git push bitbucket --mirror
cd .. && rm -rf orderflow-mirror
```

> The repository has one branch per scenario plus `main`. Never merge scenario
> branches — they are permanent fixtures.

---

### 3 — Run the benchmark

```bash
source .env
cd benchmark
.venv/bin/python cli.py run --agent-url http://localhost:8080
```

Useful flags:

```bash
# Single scenario
python cli.py run --scenario SCEN-009 --agent-url http://localhost:8080

# Filter by tag
python cli.py run --tag security --agent-url http://localhost:8080

# Skip TLS verification (corporate self-signed certificates)
python cli.py run --no-verify-ssl --agent-url http://localhost:8080

# Dry-run (no real PR, no agent call — just validates YAML)
python cli.py run --dry-run

# Compare with the previous run
python cli.py run --compare-with last --agent-url http://localhost:8080

# A/B test two agent versions
python cli.py ab --agent-a http://agent-v1:8080 --agent-b http://agent-v2:8080

# Show last run report
python cli.py report last

# Run history
python cli.py history
```

---

## Manual configuration (without the wizard)

Edit `benchmark/config.yaml` and set env vars yourself:

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
  trigger: "http"          # http | webhook  — see below
  base_url: "http://localhost:8080"
  api_key: "${AGENT_API_KEY}"
  timeout_seconds: 120

judge:
  model: "claude-opus-4-6"
  temperature: 0
  # For any OpenAI-compatible endpoint (DeepSeek, Ollama, vLLM, OpenAI, …):
  # api_url: "https://api.deepseek.com/v1"
  # api_key: "${DEEPSEEK_API_KEY}"
```

### Trigger modes

| `trigger` | How it works | When to use |
|---|---|---|
| `http` (default) | Sends `POST /review` with the PR id to `base_url` and waits for a response | Agent exposes a direct HTTP endpoint |
| `webhook` | Adds `agent_account` as a reviewer on the PR, then waits `timeout_seconds` | Agent is wired to Bitbucket via `PR_REVIEWER_UPDATED` webhook — no HTTP call needed |

**Webhook setup example:**

```yaml
agent:
  trigger: "webhook"
  timeout_seconds: 120   # how long to wait after adding the reviewer
```

The `agent_account` is taken from `bitbucket.connection.agent_account` — no extra config needed.
`base_url` and `api_key` are ignored in webhook mode.

Required environment variables:

| Variable | Description |
|---|---|
| `BITBUCKET_URL` | Bitbucket Server base URL |
| `BITBUCKET_PROJECT` | Project key |
| `BITBUCKET_REPO` | Repository slug |
| `BITBUCKET_TOKEN` | Personal access token (needs PR read/write) |
| `BITBUCKET_AGENT_ACCOUNT` | Username of the agent's Bitbucket account |
| `ANTHROPIC_API_KEY` | Required when using Anthropic judge |
| `AGENT_API_KEY` | Optional — passed as Bearer token to the agent |

---

## Local configuration

To override config without touching `config.yaml`, create `benchmark/config.local.yaml`
(gitignored). Only write the keys you want to change — the rest is taken from `config.yaml`.

```yaml
# benchmark/config.local.yaml
bitbucket:
  connection:
    base_url: "https://bitbucket.mycompany.com"
    project: "MYPROJ"
    repo: "orderflow"
    agent_account: "review-bot"

agent:
  trigger: "webhook"
  timeout_seconds: 180
```

Secrets (tokens, API keys) go in `.env` (also gitignored):

```bash
# .env  — copy from .env.example and fill in
export BITBUCKET_TOKEN="ATBBxxxxxxxx"
export ANTHROPIC_API_KEY="sk-ant-..."
```

```bash
source .env
cd benchmark
.venv/bin/python cli.py run --agent-url http://localhost:8080
```

Neither file is committed. Claude Code does not read them unless explicitly asked.

---

## Scenarios

Each scenario is a YAML file under `benchmark/scenarios/java/`.

| ID | Branch | Issues the agent must catch |
|---|---|---|
| SCEN-009 | `feature/ORD-234-buy-3-get-1-free` | Free item picked by position not cheapest price · missing ownership check · missing `@Transactional` · N+1 promotion query · no Lombok on entity |
| SCEN-010 | `feature/ORD-301-store-credit` | Store credit IDOR (any user redeems any credit) · credit deducted from post-tax total instead of pre-tax subtotal |
| SCEN-011 | `hotfix/ORD-287-cancel-npe` | Null guard on `@OneToMany` hides a Hibernate mapping error and silently skips inventory release |

---

## Adding a scenario

**Step 1 — push the branch** to the target repository with the code problem to detect.
The branch is a permanent fixture — never merge it.

**Step 2 — write the YAML** at `benchmark/scenarios/java/SCEN-NNN-name.yaml`:

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
        ## Jira ticket / PR description
        Include acceptance criteria — the agent reads this.

expected_output:
  required_comments:
    - id: EXP-1
      type: inline          # inline | general
      severity: critical    # critical | major | minor
      location: { file: "src/main/java/com/flowmart/orders/service/Foo.java", line: 42 }
      description_keywords:
        - ["keyword1", "alternative"]   # columns = OR, rows = AND
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

**Step 3 — verify:**

```bash
python cli.py run --dry-run --scenario SCEN-NNN
```

---

## Running tests

```bash
cd benchmark
pytest
```

---

## Project structure

```
benchmark/
├── bitbucket/          # AgentPRView ABC + RealBitbucketFactory (atlassian-python-api)
├── runner/             # scenario loader, agent client, LLM judge, scorer
├── scenarios/
│   └── java/           # YAML scenario definitions
├── prompts/            # LLM judge prompt template
├── tests/              # unit tests
├── cli.py
└── config.yaml
setup.sh                # interactive setup wizard
```
