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

> **Already mirrored and need to sync updates?** Skip to [Re-syncing after updates](#re-syncing-after-updates) below.



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

### Re-syncing after updates

If the scenario branches were updated (e.g. after a code change) push only the changed branches with `--force`:

```bash
git clone --no-single-branch https://github.com/andreyzebin/orderflow.git orderflow-tmp
cd orderflow-tmp
git remote add bitbucket https://bitbucket.example.com/scm/myproj/orderflow.git
git push bitbucket --force \
  origin/feature/ORD-234-buy-3-get-1-free:feature/ORD-234-buy-3-get-1-free \
  origin/feature/ORD-301-store-credit:feature/ORD-301-store-credit \
  origin/hotfix/ORD-287-cancel-npe:hotfix/ORD-287-cancel-npe
cd .. && rm -rf orderflow-tmp
```

`--no-single-branch` fetches all remote branches so they are available as `origin/*` refs.
`--force` is needed when branch history is rewritten (e.g. hint comments removed).

---

### 3 — Run the benchmark

```bash
source .env
.venv/bin/python benchmark/cli.py run --agent-url http://localhost:8080
```

Useful flags:

```bash
# Single scenario
.venv/bin/python benchmark/cli.py run --scenario SCEN-009 --agent-url http://localhost:8080

# Filter by tag
.venv/bin/python benchmark/cli.py run --tag security --agent-url http://localhost:8080

# Skip TLS verification (corporate self-signed certificates)
.venv/bin/python benchmark/cli.py run --no-verify-ssl --agent-url http://localhost:8080

# Dry-run (no real PR, no agent call — just validates YAML)
.venv/bin/python benchmark/cli.py run --dry-run

# Benchmark specific prompt version (for evolution A/B testing)
.venv/bin/python benchmark/cli.py run --prompts=/path/to/prompts/v2
.venv/bin/python benchmark/cli.py run --prompts=bitbucket://server/PROJECT/prompts-repo/refs/mut-042/prompts

# Compare with the previous run
.venv/bin/python benchmark/cli.py run --compare-with last --agent-url http://localhost:8080

# A/B test two agent versions
.venv/bin/python benchmark/cli.py ab --agent-a http://agent-v1:8080 --agent-b http://agent-v2:8080

# Show last run report (terminal table)
.venv/bin/python benchmark/cli.py report last

# Generate HTML report and open in browser
.venv/bin/python benchmark/cli.py report last --html
.venv/bin/python benchmark/cli.py report run-20260331-091228 --html

# Run history
.venv/bin/python benchmark/cli.py history
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
  output: "log"   # stream — show LLM response as it generates / log — silent (default)
  # For any OpenAI-compatible endpoint (DeepSeek, Ollama, vLLM, OpenAI, …):
  # api_url: "https://api.deepseek.com/v1"
  # api_key: "${DEEPSEEK_API_KEY}"
```

### Trigger modes

| `trigger` | How it works | When to use |
|---|---|---|
| `http` (default) | Sends `POST /review` with the PR id to `base_url` and waits for a response | Agent exposes a direct HTTP endpoint |
| `webhook` | Adds `agent_account` as a reviewer on the PR, then waits `timeout_seconds` | Agent is wired to Bitbucket via `PR_REVIEWER_UPDATED` webhook — no HTTP call needed |
| `cli` | Runs a local shell command and waits for it to exit | Agent is invoked via CLI (e.g. pr-agent, custom scripts) |

**Webhook setup example:**

```yaml
agent:
  trigger: "webhook"
  timeout_seconds: 120   # how long to wait after adding the reviewer
```

The `agent_account` is taken from `bitbucket.connection.agent_account` — no extra config needed.
`base_url` and `api_key` are ignored in webhook mode.

**CLI trigger setup example:**

```yaml
agent:
  trigger: "cli"
  command: 'source .env && .venv/bin/python pr_agent/cli.py --pr_url="{pr_url}" improve --extended'
  cwd: "~/repos/pr-agent"   # working directory (~ is expanded); defaults to current dir
  timeout_seconds: 300
  output: "stream"          # stream — single updating line / log — one line per output (default)
```

Available placeholders in `command`:

| Placeholder | Value |
|---|---|
| `{pr_url}` | Full Bitbucket PR URL |
| `{pr_id}` | Integer PR ID |

The command runs via `bash`, so `source`, pipes, and shell variables work.

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
.venv/bin/python benchmark/cli.py run --agent-url http://localhost:8080
```

Neither file is committed. Claude Code does not read them unless explicitly asked.

---

## Enterprise / corporate infrastructure

### Self-signed TLS certificates

If your Bitbucket Server uses a self-signed or corporate CA certificate, use `--no-verify-ssl`.
To suppress the resulting `InsecureRequestWarning`, add to your `.env`:

```bash
export PYTHONWARNINGS="ignore:Unverified HTTPS"
```



```bash
.venv/bin/python benchmark/cli.py run --no-verify-ssl --agent-url http://localhost:8080
```

For a proper fix (recommended), supply the CA bundle and optionally a client certificate
in `config.local.yaml`:

```yaml
bitbucket:
  connection:
    ssl:
      ca_cert: "/path/to/corporate-ca.crt"
```

Then run without `--no-verify-ssl` — the CA bundle is used for verification.

### Mutual TLS (client certificate)

Some corporate Bitbucket instances require a client certificate in addition to the Bearer
token (e.g. when the server enforces PKI authentication).

Convert your P12 to PEM once:

```bash
openssl pkcs12 -in ~/certs/client.p12 -out ~/certs/client.pem -nodes -passin pass:<password>
chmod 600 ~/certs/client.pem
```

Then configure in `config.local.yaml`:

```yaml
bitbucket:
  connection:
    ssl:
      ca_cert: "/path/to/corporate-ca.crt"   # omit if using --no-verify-ssl
      client_cert: "/home/user/certs/client.pem"
      # client_key: "/home/user/certs/client.key"  # if cert and key are separate files
```

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
  capabilities: [business_logic, null_safety]  # for evolution capability breakdown
```

**Step 3 — verify:**

```bash
.venv/bin/python benchmark/cli.py run --dry-run --scenario SCEN-NNN
```

---

## Running tests

```bash
cd benchmark && pytest
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
