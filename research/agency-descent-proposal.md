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

## The unified system: wire in what already exists

Most of the loop is already built somewhere in `allternit-platform`, just not connected to the Agency API.
The repo's own plan, the eight-plane table in `REPO_STRUCTURE.md:134-147`, already says these pieces belong together.
In code, `agency_api` only reaches the decision runtime, System One/Laya and computer use. Everything below is wiring
plus a few upgrades, not new systems.

| Plane (REPO_STRUCTURE) | Already built | Connected to Agency today? | Descent role | Upgrade from the research |
|---|---|---|---|---|
| 5 Context | **`memory_index.rs`**: hybrid BM25 (FTS5) + cosine + RRF + HNSW over facts, entities, observations and doc chunks. Embedder: `nomic-ai/modernbert-embed-base` at :7719 (`tools/system-one-local/laya/serve-embed.py`), spawned by desktop | ❌ (same binary, not called) | Makes routing `retrieval` real: the context compiler pulls memory into every run | Test **EmbeddingGemma 2** as `ALLTERNIT_EMBED_MODEL`. Same OpenAI-style endpoint; it adds images, audio and code, and MRL lets the index shrink |
| 5 Context | **`context-pack-builder` + `ontology-runtime`** (typed object graph) | ❌ (not a dependency of allternit-api) | Deterministic context bundles per node; the ontology becomes the **CorpusMap-style entity layer** | Build entity pages offline from memory_index entities |
| 5 Context | `semantic_cache.rs` (shadow only) | ❌ | Step 0 of every decision: "check saved results first" (the Jev support pattern) | Promote out of shadow via the same replay gate |
| 4 Cognition | **`domains/decision-runtime`** scorer (`/v1/score`; Qwen3.5-4B on MLX or llama.cpp) | ✅ `local` backend | The S1 heads | Add **d1-3B** (GGUF, llama.cpp ≥ Oct 8) and **Clef-flash / d3-flash** as presets. Clef and d1 use schema heads, so they need a new engine next to `engine_llamacpp.py` |
| 4 Cognition | `flywheel.rs` (only head is `head.lookup`, an exact-match read) | ✅ | Promotion gate | Accept trained adapter heads, not just lookups |
| Training | **`tools/system-one-local/laya/finetune/`** (Laya typed-decision fine-tune) + **`services/model-training`** "Model Lab" (:9020, GGUF/MLX export) | ❌ | The missing Distil step | Use the **Unsloth** decision recipe; feed it opt-in Harvest data |
| 6 Execution | **`services/local-engine`** (:3015, llama.cpp runtimes, `/recommend`) | ❌ (only `local_engine_routes.rs`) | Serves `mc.local` S2 models and trained heads | Add **Qwen3.8-Flash-Next / Strata** to its catalog |
| 7 Governance | **`domains/kernel/.../evaluation-harness`**, `domains/governance` **`allternit-evals`**, receipts, replay | partly (receipts and replay yes; evals no) | The Judge step | Replay overrides + **flow-1-style** trace monitor |
| Self-improvement | **`evolution-layer`** ("ALMA, SkillRL, AdaptEvolve"), `services/self-improve` (scaffold), `allternit-rlm` | ❌ | Phase 4 harness learning | Patterns from **MASS / Harness Learning / VERA** |

**Consolidate embedders.** There are four today: modernbert (API), `mxbai-embed-large` via Ollama
(`services/memory`), MiniLM (`open-notebook`), and OpenAI `text-embedding-3` (gizzi brain). Moving everything onto the one
`ALLTERNIT_EMBED_URL` endpoint is what makes memory **one** system. Whatever runs there (modernbert now, possibly
EmbeddingGemma 2 later) then serves agents, memory, notebook and decisions alike.

**Why the unified system is fast:**
- Every step tries the cheapest layer first: semantic cache → trained S1 head (ms, local) → retrieval-backed S2 → S3.
- Each layer's misses become training data for the layer below it.
- Desktop already spawns the embedder and the decision scorer locally (`system-one-manager.ts`,
  `decision-runtime-manager.ts`). For the cloud test (decision 1), run the same two services next to the cloud API.

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

**Phase 1: Wire + Harvest, on the cloud test deployment.**
- Deploy the decision-runtime scorer and the embed server next to the cloud API, with execution enabled for internal test orgs only.
- Call `memory_index` from the Agency context step, so `retrieval` stops being trace-only.
- Add d1-3B and Clef-flash presets to decision-runtime.
- Log every decision (oracle label + verifier outcome) for opted-in orgs.
- Add the `/v1/descent` metrics endpoint.
- *Proves:* memory and decisions run as one loop, and we can see how much work could move to S1.

  **Status (2026-10-12):** code done on `allternit-platform` branch `claude/stoic-ritchie-6g7zpd`, not yet merged.
  - Every embedder now runs **EmbeddingGemma 2**: the API memory index, the memory agent, open-notebook and gizzi brain.
  - New **`open` decision backend** speaking `/v1/systemone`, for d1, Clef and d3. The sidecar's new `systemone` engine runs **d1-3B**, tested live.
  - **Opt-in harvest** with DLP redaction and JSONL export (`/v1/decisions/training`, `/v1/decisions/datasets/:kind`).
  - **`/v1/descent`** metrics endpoint.
  - **Memory recall** in the Agency context step.
  - Still to do: the cloud test deployment itself (run the two sidecars next to the API and enable execution for internal orgs).

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

---

## Gap analysis (2026-10-12, after PR Allternit/allternit-platform#1515)

**Bottom line:** PR #1515 builds the *plumbing* of the loop: one shared embedder, open decision models, harvest, metrics, and memory in runs. The loop itself does not close yet. Nothing trains a head, nothing tests a new head against past runs, and Agency runs barely use S1 decisions. Those three gaps are what to do next.

### A. Phase 1 leftovers (finish before turning anything on)

| # | Gap | Why it matters | Who |
|---|---|---|---|
| A1 | **Cloud test deployment.** Run serve-embed (EmbeddingGemma 2) and the decision sidecar next to the cloud API, and set `ALLTERNIT_AGENCY_EXECUTE=1` for internal orgs only | Founder decision 1. Nothing runs for real until this exists | Infra / founders |
| A2 | **GPU for decision models.** d1-3B took about 6.5 s on CPU against about 8–50 ms on GPU; Clef-flash needs about 41 GB of VRAM | The `open` tier is only "fast" on a GPU | Infra |
| A3 | **Desktop impact.** The embed download grows from about 600 MB to about 1.5 GB, plus torchvision and pillow; `release-preflight.mjs` has not been run | Desktop release lock: the preflight must be 26/0 | Eng |
| A4 | **Opt-in UI.** `PUT /v1/decisions/training` has no screen in the console or workspace | Orgs cannot opt in without curl | Eng (UI) |
| A5 | **Descent dashboard.** `/v1/descent` returns JSON only | The customer pitch needs to be visible | Eng (UI) |
| A6 | **Legal.** Cross-org pooled training needs ToS / DPA wording. The Liquid d1 license (LFM Open 1.0) is reportedly limited to companies under $10M revenue | Decisions 2 and 4 need it on paper | Founders / legal |
| A7 | **Re-ingest open-notebook sources.** Old MiniLM chunks are skipped until re-ingested | Old notebooks look empty to search | Ops script |

### B. The loop does not close yet (Phase 2: the core of Descent)

| # | Gap | Today | Needed |
|---|---|---|---|
| B1 | **Agency runs barely use S1 decisions** | `executor.rs`: S1 is shadow classification only. Real `/v1/decisions` traffic comes from computer use | Graph nodes with `cognitive_role: S1` that call `decide()` live. Without this there is little to harvest or measure |
| B2 | **No trainer** | `flywheel.rs:24` says "Training is not here". Unsloth recipe and Model Lab (:9020) exist but are not wired | A job: dataset export → Unsloth decision fine-tune (d1-3B / EmbeddingGemma-2 base) → adapter → artifact store |
| B3 | **No model-backed head** | The only head is `head.lookup` (exact match) | A `Head` that serves a trained adapter through the sidecar, registered in shadow, promoted by the existing flywheel |
| B4 | **Replay cannot swap anything** | "replay overrides are not available in the alpha" | `overrides` on `POST /v1/replays` (head / model class / graph version), so a candidate is tested on past runs before going live |
| B5 | **No automated acceptance rule** | Flywheel promotes on live accuracy only | Promote only if replay quality ≥ baseline − ε and cost or latency improves; report pass@k (Sharpening Tax) |
| B6 | **Shared heads do not reach devices** | Heads live server-side | Ship promoted adapters to Desktop/phone for `local_only` orgs |

### C. Later phases (designed, not started)

| # | Gap | Phase |
|---|---|---|
| C1 | **S1 → S0 compile:** turn near-deterministic decisions into code nodes (AgentRun move) | 3 |
| C2 | **Trace monitor:** flow-1-style error finder on every run, as a decision kind | 3 |
| C3 | **Entity layer:** CorpusMap-style pages from `ontology-runtime` + `context-pack-builder`. Neither is a dependency of allternit-api today | 3 |
| C4 | **One memory store:** the vector space is shared now, but memory agent (:3201), open-notebook (SurrealDB/Qdrant) and the API index are still separate stores | 3 |
| C5 | **Eval harness:** trajectory graders, a CI regression gate, a simulated user (MIMESIS-style), deterministic fault replay (Workers IO) | 3 |
| C6 | **Harness learning:** a proposer edits task-type graphs (`evolution-layer`, MASS/VERA patterns), human-approved | 4 |
| C7 | **Local S2:** Qwen3.8-Flash / Strata in `services/local-engine` catalog as `mc.local` | Any time |

### D. Risks to watch
- **Privacy:** harvest redaction uses the DLP regex set (keys, cards, emails, SSNs and similar). Names and free-text PII are not caught. Pooled export strips scope and owner ids, but the context text itself may still identify someone.
- **Re-embedding load:** every existing memory row re-embeds on first start with the new model.
- **Memory in prompts:** recalled memory is added to S2/S3 prompts. Watch token cost and prompt injection from stored memories (the `injection_score` DLP check is not applied yet).

### Suggested order
1. A1 + A2 + A3 (get it running in the cloud test org)
2. B1 (real S1 traffic)
3. B2 + B3 (first trained head)
4. B4 + B5 (prove it safely)
5. A4 + A5 (show customers)
6. Then Phase 3
