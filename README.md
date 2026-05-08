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

Three options depending on what you want to happen on Bitbucket:

```bash
git clone --bare https://github.com/andreyzebin/orderflow.git orderflow.git
cd orderflow.git
git remote add bitbucket https://bitbucket.example.com/scm/myproj/orderflow.git
```

| Recipe | Behaviour |
|---|---|
| `git push --all --force bitbucket` | Push EVERY branch from upstream, force-overwrite tips. Branches that exist only on Bitbucket are left alone. Safest "auto-update everything" option. |
| `git push --mirror bitbucket` | Push every ref AND delete Bitbucket-only refs that aren't on GitHub. Use when you want Bitbucket to be a 1:1 copy. Same recipe as the [initial mirror in Quickstart § 2](#2--mirror-the-example-repository). |
| `git push --force bitbucket origin/<BRANCH>:<BRANCH> …` | Surgical: refresh only the branches you list, leave everything else (and tags) untouched. Useful when you reworked one scenario branch and don't want to touch the others. |

Cleanup: `cd .. && rm -rf orderflow.git`.

`--force` is needed because scenario branches sometimes have history
rewritten (e.g. when hint comments are removed); without it Bitbucket
rejects the push as non-fast-forward.

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

# Run all scenarios against multiple LLM providers in one pass.
# `providers` come from agent.providers in config.local.yaml; each entry
# is a profile name from ~/repos/.llm_creds.toml.
.venv/bin/python benchmark/cli.py run --all-providers
.venv/bin/python benchmark/cli.py run -p deepseek -p qwen3-6
```

### Trace layout

Set `BENCHMARK_TRACE_DIR` to dump every LLM/tool API call to disk for
later inspection. One session per benchmark invocation:

```
<BENCHMARK_TRACE_DIR>/<YYYYMMDD-HHMMSS[-label]>/
  bench.json      providers, scenarios, agent git_sha, judge model
  summary.json    totals + per-provider rollup + flat rows
  <provider>/<scenario>/attempt-NN/
    agent/        diff-graph trace tree (LLM/tool request+response per step)
    judge/        judge request.json / response.json
    result.json   verdict + score for this attempt
```

`attempt-NN` auto-increments per `(provider, scenario)` — re-running the
same session measures variance without bookkeeping. Different agent
versions live under separate sessions; set `BENCH_LABEL` to tag them.

The runner sets `DIFFGRAPH_TRACE_PATH=<...>/agent/` for each agent
subprocess so its trace tree lands directly inside the attempt
directory — no separate `runs/<uuid>/` to reconcile later.

### LLM provider matrix

`-p <name>` (repeatable) and `--all-providers` invoke the same scenarios
against multiple LLM profiles defined in `~/repos/.llm_creds.toml`.
Each iteration is independent: a failure in one provider/scenario is
captured as `verdict="error"` in the result row and the matrix
continues. The agent CLI receives `--provider=<name>` via the trigger
command template (`{provider}` placeholder is substituted automatically).

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

### Corporate proxies (CheckPoint, Zscaler, etc.)

On startup, `cli.py` auto-loads the `truststore` library which injects OS-level CA
certificates into Python's SSL module (Windows Certificate Store, macOS Keychain,
Linux `/etc/ssl/certs`). Corporate proxy CAs added to the OS trust store are picked
up automatically — no `REQUESTS_CA_BUNDLE` or `--no-verify-ssl` needed.

If your Bitbucket Server uses a self-signed or corporate CA that isn't in the OS
trust store, either add it system-wide or pass `ca_cert` in `config.local.yaml`
(see below).

### Windows UTF-8 output

`cli.py` sets `PYTHONUTF8=1` automatically so output redirection to files works
with non-ASCII content on Windows (avoids `UnicodeEncodeError: 'charmap' codec…`).

### Fallback: disable SSL verification

If nothing else works, use `--no-verify-ssl`. Suppress the resulting warning:

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

Each scenario is a YAML file under `benchmark/scenarios/`. Three tiers:

**`tier:unit`** (`benchmark/scenarios/agents/`) — isolated tests of one
agent at a time, ~5–10 LLM calls per test. The unit suite drives the
pre-commit gate. Uses `--mocks` to short-circuit subagent calls and
`--user-message-from` to swap the agent's task framing without touching
its system prompt:

| ID | Agent under test | Channel | What it tests |
|---|---|---|---|
| `DISP-001` | `dispatcher` | PR ack reply (`pr_comments`) | `/review` spawns reviewer once, posts brief ack, doesn't mirror findings. Reviewer is mocked. |
| `INV-001` | `investigator` | `done(findings)` (`intended_findings`) | Standalone investigator on a single focus; identifies the BLOCKER cheapest-item bug, cites AGENTS.md. |
| `REV-001` | `reviewer` (concerns-only) | `reflect.questions_remaining` (`intended_concerns`) | LOOK phase only — surfaces concerns, no investigation/publishing. |
| `REV-002` | `reviewer` (consolidation-only) | PR comments (`pr_comments`) | JUDGE phase only — receives prefilled findings via custom user message, consolidates, publishes. |

`expected_output.assert_via` declares which channel the judge matches
against; `expected_output.concern_focuses` adds keyword groups for
reflect-based tests; `setup.mocks` and `trigger.user_message_from` are
the per-scenario knobs that drive isolation.

**`tier:integration`** (`benchmark/scenarios/{interaction,java}/`) —
full-stack tests with no mocks; the dispatcher / reviewer pipeline runs
end-to-end and findings are scored against real PR comments. Drives the
pre-merge gate. Includes all interaction scenarios plus four java
sentinels (009, 010, 011, 305).

**Review scenarios** (`benchmark/scenarios/java/`) — agent must produce inline
comments on the diff, judged against `required_comments`:

| ID | Branch | Issues the agent must catch |
|---|---|---|
| SCEN-009 | `feature/ORD-234-buy-3-get-1-free` | Free item picked by position not cheapest price · missing ownership check · missing `@Transactional` · N+1 promotion query · no Lombok on entity |
| SCEN-010 | `feature/ORD-301-store-credit` | Store credit IDOR (any user redeems any credit) · credit deducted from post-tax total instead of pre-tax subtotal |
| SCEN-011 | `hotfix/ORD-287-cancel-npe` | Null guard on `@OneToMany` hides a Hibernate mapping error and silently skips inventory release |

**Interaction scenarios** (`benchmark/scenarios/interaction/`) — agent receives a
`/command` comment and must reply in the thread; judged against
`expected_output.reply` (semantic match) and `side_effects` (no inline comments,
no status change for read-only commands):

The agent supports exactly three commands, each with a distinct role,
and all three are thread-aware when the trigger sits inside a comment
thread:

- **`/ask`** (or plain text without a slash) — *discussion*. Answer
  scoped to the active thread's topic; sibling threads are background.
- **`/help`** — *interface help*. Explains the commands, how to summon
  the agent, and which command best fits the user's current situation.
- **`/review`** — *deep analysis*. Full code review. When invoked from
  a thread, focuses on that thread but also reads sibling threads with
  author attribution to strengthen findings via prior debate.

| ID | Branch | What it tests |
|---|---|---|
| SCEN-200 | `hotfix/ORD-287-cancel-npe` | `/help` lists the three supported commands, no review |
| SCEN-201 | `hotfix/ORD-287-cancel-npe` | `/ask` reads multi-author thread context (single-account simulation via `[name]` text prefixes) |
| SCEN-202 | `hotfix/ORD-287-cancel-npe` | Unknown `/improve` answered with "not supported" + the three commands, NOT silently routed to `/ask` |
| SCEN-204 / 204b / 204c | `feature/ORD-234-buy-3-get-1-free` | Multi-thread comprehension. Three sibling threads on distinct topics; the same intentionally bland question (`/ask что об этом думаешь?`) is asked in each one in turn. The agent must follow `parent_id` and answer the triggered thread, not whichever sibling's text happens to match the question semantically. Crosstalk reproducer. |
| SCEN-205 | `feature/ORD-234-buy-3-get-1-free` | Author attribution. A five-speaker debate thread; the trigger asks specifically which comments belong to `alice`. The agent must enumerate alice's two non-adjacent messages and not mix in the bot's bug report, its own past reply, or bob's verdict. |
| SCEN-205b | `feature/ORD-234-buy-3-get-1-free` | SELF recognition. Same shape as 205 but the agent itself has two prior replies in the thread and the trigger asks "which comments are yours?". Catches the "I haven't commented yet" failure mode. Pairs with 205 to cover both directions of attribution. |

### Running tiers

```bash
# unit gate: 4 isolated tests × providers
.venv/bin/python benchmark/cli.py run -t tier:unit -p deepseek -p qwen3 -p qwen3-6 \
    --mode aggressive --max-per-provider 2

# integration gate: 13 scenarios (interaction + 4 java sentinels)
.venv/bin/python benchmark/cli.py run -t tier:integration -p qwen3-6 \
    --mode aggressive --max-per-provider 2

# all (unit + integration on one provider)
.venv/bin/python benchmark/cli.py run -p qwen3-6 \
    --mode aggressive --max-per-provider 2
```

`--mode aggressive` runs (provider × scenario × attempt) tasks in
parallel via temp-branch PRs; `--max-per-provider N` caps concurrent
LLM calls per model. `gentle` (default) is sequential.

For agent-isolation tests the per-attempt artefacts include:
- `agent/` — full diff-graph trace tree (request/response per step)
- `judge/{request,response}.json` — judge prompt + verdict JSON
- `invocations.json` — every tool call (tool, args, mocked, mock_when)
- `result.json` — verdict + score for the attempt

`BENCHMARK_TRACE_DIR=<path>` enables the per-attempt artefact tree.
Without it, only the SQLite `~/.diffgraph/traces.db` gets the agent
events; the bench's invocations.json + judge artefacts aren't
written.

The fixture branches live in
[`andreyzebin/orderflow`](https://github.com/andreyzebin/orderflow) on GitHub.
First-time mirroring and ongoing re-syncs are covered in Quickstart §2:
[Mirror the example repository](#2--mirror-the-example-repository) and
[Re-syncing after updates](#re-syncing-after-updates).

---

## Adding a new fixture branch

1. Branch off `master` in `orderflow` with a descriptive name
   (`feature/<TICKET>-...` or `hotfix/<TICKET>-...`).
2. Commit the buggy / interesting state. Push to GitHub.
3. Re-sync the Bitbucket mirror — see
   [Re-syncing after updates](#re-syncing-after-updates).
4. Add `benchmark/scenarios/<dir>/SCEN-NNN-*.yaml` referencing the branch
   under `input.bitbucket.pull_request.from_branch`.
5. Verify: `python benchmark/cli.py run --scenario SCEN-NNN --dry-run`.

### Multi-step iteration scenarios (planned)

Some upcoming scenarios test the agent on PR re-reviews after a fix is pushed.
They use **paired branches**: `feature/X-step0` (initial bug) and
`feature/X-step1` (same content + the fix commit). The runner force-pushes
the step-1 tip onto the source branch between phases, so both branches must
exist in the mirrored Bitbucket repo before the scenario runs.

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
