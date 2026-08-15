---
name: evorule
description: >-
  Execute local JSON compliance rules deterministically with EvoRule,
  producing tamper-evident JSONL fact logs with BLAKE3 hash-chain and
  structural invariant verification. Use when the user needs a rule engine
  to run deterministic workflows, generate auditable execution trails,
  replay or diff execution facts, or verify the integrity of a fact log
  (hash chain, monotonic FactId, valid cause references). Requires the
  evorule CLI (single static binary, zero network, zero telemetry).
---

# EvoRule — Deterministic Rule Execution & Audit

## Overview

EvoRule is a deterministic rule engine: the user writes business/compliance
rules as a JSON file, EvoRule executes them locally and writes **every step**
to a JSONL fact log (a WAL audit trail). "LLM thinks, EvoRule executes."
This skill teaches you how to drive the `evorule` CLI end-to-end.

**Key principles**
- Zero network, zero telemetry, zero AI decisions inside the engine.
- Deterministic: same rules + same payload => identical fact log.
- Every execution leaves a verifiable audit trail (BLAKE3 hash chain).

## When to Use

- User wants to run local compliance/business rules (JSON) with an audit trail.
- User needs deterministic, reproducible execution instead of a script.
- User needs to verify a fact log has not been tampered with.
- User needs to replay what happened, or diff two runs.

Do NOT use for: ad-hoc one-off transformations, general scripting, or anything
that must call external services (evorule-cli has no I/O handler in 0.2.x).

## Prerequisites

1. Install the CLI (single static binary, pick your arch):
   ```bash
   wget https://gitee.com/evorule/evorule/releases/download/v0.2.4/evorule-x86_64
   chmod +x evorule-x86_64 && sudo mv evorule-x86_64 /usr/local/bin/evorule
   evorule --version
   ```
2. Check it works: `evorule --help` must list the 5 subcommands.

## Core Workflow

1. **Write rules** as JSON files in a directory (see `examples/` in this repo).
2. **Validate**: `evorule validate ./rules/`
3. **Run**: `evorule run ./rules/ --payload-file payload.json -o fact.log`
4. **Verify**: `evorule verify-chain fact.log` (hash chain + invariants)
5. **Audit**: `evorule replay fact.log` / `evorule diff a.log b.log`

## Command Reference

| Command | Purpose | Exit 0 means |
| --- | --- | --- |
| `evorule validate <rules-dir>` | Check transform types are in core_eval whitelist | all transforms legal |
| `evorule run <rules-dir> [--payload S|--payload-file F] [-o OUT] [--max-steps N]` | Execute rules -> fact log (JSONL) | execution completed (may contain Error fact) |
| `evorule replay <fact-log>` | Pretty-print each fact | file read & parsed |
| `evorule diff <a.log> <b.log>` | Align facts by index and show differences | done (differences allowed) |
| `evorule verify-chain <fact-log>` | BLAKE3 hash chain + FactId monotonic + cause refs valid | all three checks pass |

Exit codes: `0` ok, `1` error, `2` rule dir missing / no .json files.

Fact types (JSONL, one per line): `Command`, `StateTransition`, `IoRequest`,
`IoResponse`, `Stable`, `Error`, `PayloadUpdate`.

## Rule File Format

Each file contains a `transform` array (same shape as the `core_eval.json`
constitution). Multi-file directories are loaded in filename sort order
(deterministic across platforms). Supported shapes: `{"transform":[...]}`,
`{"transforms":[...]}`, `[...]` (top-level array), `{...}` (single transform).

Valid `type` values (core_eval meta-instructions):
`branch` (conditional), `set` (payload mutation: set/add/sub), `push`
(queue instructions), `io_request`, `noop`, plus business instructions
(`increment`, `decrement`, ...) that the constitution maps.

`set` attr rules:
- Literal string `"x"` => payload field name.
- Dotted path `"a.b.c"` => auto-creates nested objects.
- Do NOT write `"__exec__.payload.x"` as an attr — it is parsed as a path.

Example — always set `x = 42`:

```json
{
  "transform": [
    {
      "type": "branch",
      "params": {
        "domain": {"type": "all", "inner": []},
        "on_true": [
          {"type": "set", "params": {"attr": "x", "operation": "set", "value": 42}}
        ]
      }
    }
  ]
}
```

Full working examples live in `examples/` of this repo: `hello-rules`
(always-set), `echo-rules` (copy `input` -> `result`), `fifo-rules`
(step1 then step2, proving FIFO queue order).

## Integration with DSH

Use EvoRule as the **deterministic execution/audit layer** for an agent:

1. Agent reads user intent, writes a rules JSON into the workspace.
2. Agent runs `evorule run` to execute (deterministic, no hallucination).
3. Agent feeds the `Stable.final_snapshot` back to the user.
4. Keep `fact.log` as the audit evidence; verify-chain before trusting it.

Because evorule-cli is a static binary and uses only local files, it is safe
to invoke from the agent's shell with a workspace-scoped rules directory.

## Common Mistakes

- Writing unknown `type` in transform -> `validate` fails (exit 1). Use the
  core_eval whitelist.
- Using `"__exec__.payload.x"` as `set.attr` -> interpreted as path, not field.
- Expecting external I/O to run: in 0.2.x `io_request` emits `IoRequest` +
  `Error` facts (no handler), it does not call out.
- Trusting a fact log without `verify-chain` — always verify before audit.
- Forgetting that `run` never fails on rule errors; check the `Error` fact.

## Reference

- Source & docs: https://gitee.com/evorule/evorule
- License: AGPL-3.0-or-later (engine). `core_eval.json` constitution is CC0-1.0.
