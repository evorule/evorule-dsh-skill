# EvoRule DSH Skill

[![AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0--or--later-green)](LICENSE)

A [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) skill
that teaches agents how to use **EvoRule** — the deterministic JSON rule engine
with tamper-evident audit trails.

**"LLM thinks, EvoRule executes."**

## What is EvoRule?

EvoRule is a deterministic rule engine:
- Write business/compliance rules as **JSON files**.
- Run them locally — **zero network, zero telemetry, zero AI decisions**.
- Every execution step is recorded in a **JSONL fact log** (WAL audit trail).
- Verify the log is untampered with **BLAKE3 hash chain** verification.

## Install

### 1. Install the CLI

Download the single static binary (1.8 MB, no dependencies):

```bash
wget https://gitee.com/evorule/evorule/releases/download/v0.2.4/evorule-x86_64
chmod +x evorule-x86_64
sudo mv evorule-x86_64 /usr/local/bin/evorule
evorule --version
```

Available for `x86_64` and `aarch64` (Linux, musl, static linked).

### 2. Install the Skill

```bash
mkdir -p ~/.dsh/skills
git clone https://github.com/evorule/evorule-dsh-skill ~/.dsh/skills/evorule
```

Or copy manually:

```bash
cp -r evorule-dsh-skill ~/.dsh/skills/evorule
```

Restart your DSH session. The agent will now automatically activate the
EvoRule skill when you ask about rule execution, audit trails, or fact log
verification.

## Quick Start (30 seconds)

```bash
# 1. Validate your rules
evorule validate examples/hello-rules/

# 2. Run with a payload
echo '{}' | evorule run examples/hello-rules/ --payload-file /dev/stdin

# 3. Run and save the fact log
evorule run examples/hello-rules/ --payload '{}' -o /tmp/facts.jsonl

# 4. Verify the hash chain
evorule verify-chain /tmp/facts.jsonl

# 5. Replay the execution
evorule replay /tmp/facts.jsonl
```

## License

AGPL-3.0-or-later. See [LICENSE](LICENSE).

The EvoRule engine (`core_eval.json` constitution) is CC0-1.0 Public Domain.