# Proposal: Agency Descent — runs that get cheaper every time

**One line:** the Agency API runs a job with expensive models first, then **learns the job**. It moves each step
down the tier ladder, from S3/S2 model calls to S1 decisions and finally S0 code. Each move is proven safe by
replaying past runs, and only then promoted. Every run makes the next one cheaper, faster and more local.

This builds on what is already in `allternit-platform` and uses the research in
[`agency-api-update.md`](agency-api-update.md). All paths below are in `allternit-platform`.

---

## Why this, and why Allternit is already halfway there

The links have one idea in common: **AgentRun** ("the harness learns the job as it runs, moving steps from LLM
calls to code", $290K → $26K), **Jev, d1, Clef and d3** (decisions in milliseconds), **Unsloth** (train your
own decision model on 4 GB), **flow-1** (cheap trace monitors), and **Harness Learning, MASS, RSI-Anything and VERA**
(improve the harness, verify, then promote). They all describe the same loop:
*do the work expensively → record it → distil it → verify it → promote it.*

The Agency API already has most of the parts for that loop:

| Piece the loop needs | Already in the Agency API | Missing |
|---|---|---|
| A tier ladder | S0 Commands / S1 Decisions / S2 Generate / S3 Solve (`kernel_ui/routing_policy.rs`) | Nothing moves work *down* the ladder |
| A decision runtime | `POST /v1/decisions` (alias `/v1/systemone`), escalation chain `head → local → vendor → typesafe → oracle` (`decisions/backends.rs:201`) | Only a generic local scorer; no d1, Clef or d3 backends |
| Shadow → live promotion | Flywheel heads promoted on accuracy (`decisions/flywheel.rs`) | **"Training (datasets, adapters, the decision model) is not here"** (`flywheel.rs:24`) |
| Ground-truth labels | Oracle answers, verifier results, signed receipts | Not exported as training data |
| A recorded history | Receipt chains, effect journal, journaled model outputs (`safety.rs`) | n/a |
| A simulator | `POST /v1/replays` with cassettes (`mod.rs:1044-1074`) | **"replay overrides are not available in the alpha"**, so a new head or graph can't be tested against old runs |
| Harness as data | ComputeGraph JSON per task type (`task_types/graphs/*.json`) | Nothing proposes or scores graph edits |
| Cost accounting | `usage_ledger.rs` by run, node and tier | No cost-per-run trend per task type |
| Retrieval | BM25 funnel in BUG_FIX only | Routing `retrieval` is **trace-only**; there are no embeddings in agency |

The gaps marked in bold are exactly what the researched projects fill.

---

## The loop

```
          ┌──────────────── run (live) ─────────────────┐
 goal ──► │ S3/S2 model calls ─► S1 decisions ─► S0 code │ ──► verified Run + receipts
          └──────────────────────────┬───────────────────┘
                                     │ receipts, oracle answers, verifier results
                                     ▼
   1. HARVEST    turn every decision/oracle call into a labelled example
   2. DISTIL     Unsloth-train a small decision head (d1-3B / EmbeddingGemma-2 base)
                 or compile a repeated decision into S0 code (AgentRun move)
   3. SIMULATE   replay N recorded runs with the candidate swapped in (replay overrides)
   4. JUDGE      replay equivalence + verifier + flow-1-style trace monitor
   5. PROMOTE    shadow → live via the existing flywheel; roll back on regression
                                     │
                                     └──► next run uses the cheaper tier
```

### 1. Harvest: decision logs become a dataset
- Every `/v1/decisions` call already escalates to an oracle when a head is unsure. Write each case as
  `(state, question schema, oracle answer, verifier outcome)` to the Factory ledger.
- Add the decision log from the Jev support workflow: route, reason and outcome, so wrong rejections can be found.
- Export with `GET /v1/decisions/datasets/:kind` (new).

### 2. Distil: two moves down the ladder
- **S2/S3 → S1 (train a head).** An Unsloth decision fine-tune with a Clef-style schema head on `d1-3B` or Qwen3.5-0.8B
  for the edge, or on EmbeddingGemma-2 (the `mrm8488` typed-decision recipe) for phones. One adapter per decision kind.
  This fills the `flywheel.rs:24` gap.
- **S1 → S0 (compile to code).** When a head's answers are near-deterministic over its inputs, a planner model writes
  a pure function for it (an AgentRun-style move) and saves it as an S0 primitive node.

### 3. Simulate: replay with overrides
- Add `overrides` to `POST /v1/replays`: swap a decision head, a model class, or a graph version.
- Effects stay `recorded_only`, so this is safe by construction: the cassette supplies every tool response.
- This is the "simulation machine" from Vtrivedy10: evals built from real trajectories.

### 4. Judge: three signals, all already half-built
- **Replay result:** `equivalent` / `diverged_expected` / `diverged_unexpected`, which already exists.
- **Verifier:** the completion contract still passes on the replayed run.
- **Trace monitor:** a flow-1-style small model, served as one more `/v1/decisions` kind, that scores the replayed trace for errors.
- **Acceptance rule (VERA-style):** promote only if quality ≥ baseline − ε on the replay set *and* cost or latency improves.
  Use pass@k, not just pass@1, so the Sharpening Tax shows up.

### 5. Promote
- Reuse the flywheel's shadow → live promotion.
- Changing `s1_backend` already resets everything to shadow, which is the right safety behaviour.
- Add `GET /v1/descent/:task_type`: cost per run, tier mix (% of S0/S1/S2/S3) and promotions over time. This is the
  number customers care about.

---

## Where each researched project fits

| Project | Role in Descent | Where it plugs in |
|---|---|---|
| Liquid **d1-3B** / d1-omni-600M | Default local S1 head (8 ms) | `system_one_local` backend; its llama-server already exposes `/v1/systemone` |
| Cloudflare **Clef / Clef-flash**, vLLM-SR **d3** family | Server-side S1 heads, baseline and teacher (open weights, Apache 2.0) | Self-hosted, as new open-weight backends in `decisions/backends.rs` |
| TypeSafe **Jev** | Not used (decision 3); Clef speaks the same API | Existing `typesafe` backend stays off |
| **Unsloth** decision training | The missing training step | New `domains/decision-trainer` job fed by Harvest |
| **EmbeddingGemma 2** | Phone-tier decision heads, and the retrieval embedder | Decision base model + context compiler |
| **AgentRun** | The S1 → S0 move, and the `call` / `judge` / fallback / escalate pattern | Graph templates + the compile-to-code step |
| **Jev + Opus support workflow** | Reference routing: cache → decide → Opus or human; targeted retries | Template for GENERAL_TASK / THREAD_WORK graphs |
| **flow-1** | Trace monitor in Judge | Decision kind `trace.error` |
| **Vtrivedy10 / suraj_sharma14** evals | Replay-as-simulator; promote prod traces to eval cases; CI gate; shadow-routing comparator | `task_types/evals/*.v1.json` + replay overrides |
| **Workers IO** | Deterministic fault injection for S0 code that Descent writes | Optional verifier for compiled nodes |
| **Harness Learning, MASS, RSI-Anything, VERA** | Phase 3: let a proposer edit the ComputeGraph JSON itself, scored by the same replay loop | Graph versions as override targets |
| **CorpusMap + EmbeddingGemma 2** | Make `retrieval` real (it is trace-only today) | Context compiler |
| **UniiChat summary tree** | Long-running Thread memory with "zoom" | `THREAD_WORK` context |
| **Qwen3.8-Flash-Next (Strata)** | Local S2 model behind `mc.local` (Anthropic-compatible localhost) | gizzi `GET /model-pool` |
| **TensorFold Living Weights** | Research spike only: per-user facts in local weights | Not in the loop |

---

## Phased plan

**Phase 1: Harvest + local S1 (smallest useful slice).**
- Add d1-3B as the `system_one_local` backend, using GGUF through the existing llama.cpp path.
- Log every decision with its oracle label and verifier outcome.
- Add the `/v1/descent` metrics endpoint.
- *Proves:* decisions run locally, and we can see what fraction could be.

**Phase 2: Replay overrides + distillation.**
- Add `overrides` to `/v1/replays`.
- Run an Unsloth trainer job and register the new head in shadow.
- Use the replay-based acceptance rule.
- *Proves:* a head trained on our own runs can replace oracle calls at equal verified quality.

**Phase 3: S1 → S0 compilation + trace monitor.**
- Compile stable decisions into code nodes.
- Add a flow-1-style monitor to Judge.
- Make CorpusMap/EmbeddingGemma retrieval real.

**Phase 4: Harness learning.**
- A proposer edits task-type graphs, MASS/VERA-style, scored by the same loop.
- Promote only with a human approval on the attention queue.

## What it means for customers

- "Your agents get cheaper the more you use them": cost per run and tier mix per task type are shown on `/v1/descent`.
- With `local_only`, the tier ladder ends on-device: d1 or EmbeddingGemma heads, and Qwen Flash for S2. The privacy
  story then holds up, not just the cost story.
- Every promotion is backed by receipts and replay evidence, which fits the existing "verified Run" contract.

## Founder decisions (2026-10-12)

1. **Where it runs: in the cloud first.** Enable `ALLTERNIT_AGENCY_EXECUTE=1` on a cloud test deployment only, with
   `ALLTERNIT_AGENCY_EXECUTE_ORGS` limited to internal test orgs. Turn it on for customers once the Phase 1 metrics look right.
2. **Training data: opt-in.** Harvest records examples only for orgs that opt in, with a new org setting next to
   `/v1/agency-safety/policy`. Runs from orgs that have not opted in still work; they just aren't recorded.
3. **No paid decision vendors.** Baselines and teachers are open-weight models:
   - **Cloudflare Clef / Clef-flash** (Apache 2.0, Jev-API-compatible) are the "vendor-grade" baseline and the teacher.
   - **vLLM-SR d3 family** (Apache 2.0) cover server-side heads.
   - **Liquid d1** handles local and edge. Its LFM Open license is reportedly free under $10M revenue; check the terms before scaling.

   The `typesafe` and OpenAI vendor backends stay off. The planner model stays the last-resort oracle.
4. **Trained heads are shared across orgs.** They are trained on pooled opt-in data, so every opted-in customer improves
   the heads for everyone. Guardrails:
   - Strip identifiers, secrets and raw tool payloads before pooling.
   - Hold out a slice per org for evals, so a shared head can't regress one customer without it showing up.
   - `local_only` orgs still get the shared heads, which run on-device; their data is pooled only if they opt in.
