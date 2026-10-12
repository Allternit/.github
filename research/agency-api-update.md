# Agency API — research update (2026-10-12)

Reading list and notes for the next revision of the Allternit Agency API: the agent
runtime layer in [`allternit-platform`](https://github.com/Allternit/allternit-platform)
that routes decisions, runs workflows, keeps memory and evaluates agents.

**Source status.** The build environment cannot reach x.com, huggingface.co or most vendor
sites. Each item is tagged:
- **Verified**: we found the post text or a primary source (repo, model card, official blog or changelog).
- **Inferred**: pieced together from secondary coverage; the original post was not read.
- **Not found**: we could not identify what the post was about.

Benchmark and price figures are vendor-reported unless stated otherwise.

---

## 0. The headline: a wave of "decision models"

Most of this list belongs to one trend from the last few weeks. **Decision models** (also called
"System One" models) do not generate text. They answer typed questions (choice, score, yes/no)
with calibrated probabilities over the options, in a single forward pass, within milliseconds.
TypeSafe's **Jev** started the category, and open-weight alternatives followed within weeks
(Liquid d1, Cloudflare Clef, vLLM-SR d3, plus Unsloth training recipes).

For the Agency API this is the most actionable change on the list. Routing, gating, guardrail
checks, tool selection and "is this answer good enough?" checks can all move from generative LLM
calls to decision calls costing 10–200 ms. The API should define its own decision contract and
treat these models as interchangeable backends.

## 1. Decision models: options

| Model | Size | Latency (median, vendor) | License | Get it | Status |
|-------|------|--------------------------|---------|--------|--------|
| **TypeSafe Jev** (hosted) | n/a | ~100 ms for most calls; claims 193x faster and 445x cheaper than LLMs | Proprietary API | TypeSafe API, Cloudflare, OpenRouter | Inferred |
| **Liquid d1-3B** | 3B (text + image) | 8 ms on RTX 4090; 16 ms on Jetson AGX Thor; 50 ms on Jetson Orin Nano | LFM Open v1.0 (reportedly free below $10M revenue) | HF `LiquidAI/d1-3B` (+ `-GGUF`, `-w8a8`); llama.cpp built Oct 8 or later | Inferred |
| **Liquid d1-omni-600M** | 600M (text, image, audio) | n/a | LFM Open v1.0 | Experimental | Inferred |
| **Atomic Chat d1-3B GGUF** | Q8_0 2.9 GB; AD-Q4_K_M 1.7 GB (~97% agreement with Q8) | n/a | as d1 | HF `AtomicChat/d1-3B-GGUF`, llama-server `/v1/systemone` | Inferred |
| **Cloudflare Clef** | 27B (Qwen3.8-27B base), ~85 GB VRAM | 209 ms | Apache 2.0 | HF `Cloudflare/clef`, Workers AI | Inferred |
| **Cloudflare Clef-flash** | 9B (Qwen3.5-9B base), ~41 GB VRAM | 39 ms | Apache 2.0 | Workers AI | Inferred |
| **vLLM-SR d3** | 27B, text/JSON + 4 images | n/a; self-reported Decision Index 0.3 public score 64.24 | Apache 2.0 | HF `vllm-sr/d3` | Verified |
| **vLLM-SR d3-flash** | 9B | 19.4 ms (text) | Apache 2.0 | HF `vllm-sr/d3-flash`; edge/nano/mini variants listed | Verified |

Notes:
- **Clef** is API-compatible with Jev. It takes a state plus up to 64 typed questions, has 64k context, and returns a probability for every option. Cloudflare also announced an RL fine-tuning service. Reported prices of $0.24 / $0.09 per M input tokens come from secondary sources only.
- **Jev question types** (Inferred, from [JeffreyBiles](https://x.com/JeffreyBiles/status/2107211222320021722) context; field names not checked against the docs):
  - `choice`: up to 255 options; returns the choice, the distribution and a confidence.
  - `score`: an ordered rubric; returns the tier, the distribution and a confidence.
  - yes/no ("noul"): returns a single probability.
- **Typesafe Jev** ([Brankotrcek](https://x.com/Brankotrcek/status/2109419442643845309)): the tweet itself was not found. TypeSafe raised an $870M round at a $7.5B valuation on Oct 9.
- **Unsloth: train your own decision model** (Verified, [tweet](https://x.com/UnslothAI/status/2107868866361930236), [releases](https://github.com/unslothai/unsloth/releases)).
  - Turns any text or vision LLM (Qwen3.8, Gemma 4) into a decision model using QLoRA.
  - Qwen3.5-0.8B went from 20.7% to 74.3% aggregate accuracy on 3 decision benchmarks, using 4 GB of VRAM.
  - Recommended settings: 2 epochs, LoRA rank 16, learning rate 2e-4. Free Colab and Kaggle notebooks; shipped in v0.1.904/905-beta.
  - This makes it possible to train our own small routers on Agency API traces.

Sources: [liquid.ai/blog/d1-open](https://www.liquid.ai/blog/d1-open), [alphasignal](https://alphasignal.ai/news/liquid-ai-s-d1-3b-makes-structured-ai-decisions-in-8-milliseconds-without),
[MarkTechPost on Clef](https://www.marktechpost.com/2026/10/01/cloudflare-releases-clef-and-clef-flash/), [The Register](https://www.theregister.com/ai-and-ml/2026/10/01/cloudflare-tries-to-outplay-jev-with-open-weight-clef-models/5300649), [InfoQ](https://www.infoq.com/news/2026/10/clef-decision-models/),
[vllm-sr/d3](https://huggingface.co/vllm-sr/d3), [Tom's Hardware on Jev](https://www.tomshardware.com/tech-industry/artificial-intelligence/typesafe-ais-jev-offers-an-alternative-to-llms-that-claims-to-be-193x-faster-and-445x-cheaper-system-one-type-model-is-bespoke-for-probabilistic-decision-making),
[SiliconANGLE on TypeSafe round](https://siliconangle.com/2026/10/09/jev-creator-typesafe-closes-870m-round-at-7-5b-valuation). Original posts: [liquidai](https://x.com/liquidai/status/2105003475017085167), [atomic_chat_hq](https://x.com/atomic_chat_hq/status/2107931155840299417), [MiaAI_lab](https://x.com/MiaAI_lab/status/2107134733394325810).

**Actions**
1. Define `decide(state, questions[]) -> [{type, answer, distribution, confidence}]` using the Jev schema (choice / score / yes-no), so Jev, Clef and other compatible backends plug in directly.
2. Benchmark on our own routing and gating traces: d1-3B (local, desktop and edge), d3-flash or Clef-flash (server), and Jev (hosted baseline).
3. Log every decision call. Those logs become Unsloth training data for an in-house router later.

## 2. Workflow DSL and orchestration

**AgentRun, the Jev-powered workflow DSL** (Verified, [repo](https://github.com/Parcha-ai/agentrun); the [_aj](https://x.com/_aj/status/2102061534956662818) tweet itself was not found, but a nearby [launch tweet](https://x.com/_aj/status/2102844881370763696) was).

- A workflow language that runs inside an existing agent stack. The host app keeps its own tools, model access, permissions and budgets.
- Workflows are JSON or a TypeScript builder with Zod contracts. Also on PyPI as `parcha-agentrun`, with a Pi extension `@parcha/agentrun-pi`.
- Nodes:
  - `call` runs a tool.
  - `judge` asks Jev a typed question and gets back a decision with a confidence score.
  - Also: sub-workflows, parallel map, bounded loops, and an agent fallback when the fixed path fails.
- Example pattern: search, then judge (accept "yes" at ≥ 0.8), then one agent retry, then escalate to a human.
- Cost claim: reviewing one compliance alert dropped from $2.89 with Opus 5 to $0.25 with AgentRun. This is the company's own internal test ([Runtimewire](https://runtimewire.com/article/grep-agentrun-compliance-agent-costs-jev)).
- Caveats: code nodes run with process privileges and need a sandbox for untrusted workflow authors. A typed decision does not prove the answer is correct. Current version is `0.1.0-beta.4`.

**Fit:** this is the pattern the Agency API needs. Cheap decision models gate each step, and expensive agents run only as the fallback. Back `judge` with the §1 `decide()` contract.

**Flow 1** ([arafatkatze](https://x.com/arafatkatze/status/2107165909639160268)): not found. The author works at Cline.

## 3. Harness engineering and self-improvement

The main point across these papers is that **the harness is a learnable artifact**. Large gains come from changing orchestration code
without changing model weights. That makes the Agency API itself the lever. (Paper summaries come from DAIR.AI's Top AI Papers of the Week.)

- **Harness Learning (CMU).** A 4B proposer model edits harness code from execution reports and is trained with RL on the harness score. The solver model stays frozen. It beats its 35B teacher and transfers to task families it never saw. [paper](https://academy.dair.ai/papers/harness-learning-enables-generalizable-test-time-adaptation-2609.35738)
- **HERMES.** Each repository component gets a resident LLM ("Dev-Primitives"), plus dependency-aware activation and failure diagnosis. A repo migration task went from 6.5% to 31.0% with the same model. Qwen3-8B primitives stay within 4.5% of a frontier-only setup. [paper](https://academy.dair.ai/papers/harness-engineering-for-software-engineering-via-modular-executable-dev-primitiv-2610.07832) — relevant to Gizzi Code.
- **VeriHarness (Google).** A same-model verifier with a disagreement resolver and a consensus challenger. It beats majority voting and adds about 6 points over a single rollout. [paper](https://academy.dair.ai/papers/veriharness-scaling-agentic-verification-for-long-horizon-tasks-2610.00972)
- **VERA (NVIDIA).** More than 9,000 restartable, rubric-verified sandboxes. It co-evolves the model and the harness skills, and accepts a harness edit only if it gains at least 5 points on the dev set. [paper](https://academy.dair.ai/papers/vera-scaling-verifiable-environments-for-agentic-co-evolution-2610.05923)
- **MASS (Sakana).** One base model (Qwen3.6-27B) acts as agent, evaluator and optimiser. Evolutionary search finds multi-agent workflows, then the model is fine-tuned on its own traces. [paper](https://academy.dair.ai/papers/recursive-self-improvement-through-multi-agent-self-supervision-2610.12176)
  - **Verified:** code at [SakanaAI/mass](https://github.com/SakanaAI/mass). It includes the pipeline CLI, workflow search, LoRA training, ScienceAgentBench and MLR-Bench adapters, and 12 synthetic tasks. Model weights are not included and the README names no license.
  - Repo results across cycles L0 → L2: gains on ScienceAgentBench (28.8 → 31.7) and MLR-Bench (1.71 → 2.25), but small drops on Terminal-Bench 2.0 (0.375 → 0.352) and SWE-bench Verified (0.626 → 0.611).
  - Tweets: [SakanaAILabs](https://x.com/SakanaAILabs/status/2109385006603006435), [hardmaru](https://x.com/hardmaru/status/2109385496237678844).
- **RSI-Anything (PhAI Labs, Ling Yang et al.).** Verified via the [repo](https://github.com/Gen-Verse/RSI-Anything) (Apache-2.0, `rsi` CLI) and [arXiv 2610.13912](https://arxiv.org/abs/2610.13912). Uses predictive world models (built on JEPA-Anything) to choose actions and training data for recursive self-improvement.
  - Code repair on CodeContests-200: 26.3% → 31.2%.
  - MiniWoB++: 99% success with 39.4% fewer actions.
  - OSWorld: +18.4% relative, versus +12.3% for RLAnything.
  - Tweet: [LingYang_PU](https://x.com/LingYang_PU/status/2109468200866074817).
- **World model** ([Charles_Y_Wu](https://x.com/Charles_Y_Wu/status/2107134454762496397)): not found. The author is CEO of PhAI Labs, the same lab as RSI-Anything. Candidates are Immune World Model, MedGenesis and JEPA-Anything.
- **Agent Plasticity (Meta).** Measures how well an agent turns experience into held-out gains per dollar, with weights frozen and agent-written artifacts carried between runs. Slow learners ignore their own artifacts. [paper](https://academy.dair.ai/papers/agent-plasticity-measuring-self-improvement-through-experience-2610.08902)
- **Sharpening Tax (Meta).** RL post-training raises pass@1 but lowers pass@K coverage. [paper](https://academy.dair.ai/papers/sharpening-tax-in-post-training-2610.01509)

**Actions**
1. Version harnesses (prompts, tool graph, workflow DSL) as data, so a proposer can diff, score and roll them back.
2. Give every run a structured execution report (inputs, tool calls, decisions, failures, score).
3. Add a verify step that targets agreement as well as disagreement, following VeriHarness, before results are accepted.
4. Persist agent-written artifacts across runs, and measure whether later runs actually reuse them.

## 4. Retrieval, memory and context

- **EmbeddingGemma 2** (Verified, [googlegemma](https://x.com/googlegemma/status/2107502533992464482), [Google dev blog](https://developers.googleblog.com/google-ai-edge-with-embeddinggemma-2/)). Released Oct 6, built on Gemma 4, Apache 2.0.
  - 740M parameters with modular encoders; the text-only configuration is about 270M.
  - Text, code, images, video and audio map into one 768-dimension space. Matryoshka dimensions down to 128, and 8K context.
  - MTEB Code 78.68 (versus 68.76 for v1).
  - About 567 MB quantised on a phone. Community GGUF: `unsloth/embeddinggemma-2-GGUF`.
  - The other EmbeddingGemma posts in the list ([mylifcc](https://x.com/mylifcc/status/2107521708223758818), [KShivendu_](https://x.com/KShivendu_/status/2107558721568321784), [xueyu1125](https://x.com/xueyu1125/status/2109301901783429283), [mrm8488](https://x.com/mrm8488/status/2109406679045423394)) were not read.
  - **Fit:** a strong default embedder that can run on device for desktop and phone workspace search.
- **CorpusMap (Microsoft).** Builds an offline entity graph over the corpus and gives each recurring entity a page. Reported +6.4 to +11.7 answer quality with 34–57% fewer input tokens than raw agentic search, and the map can be built without LLM calls. [paper](https://academy.dair.ai/papers/follow-the-entities-a-corpus-map-for-agentic-search-2609.37226)
- **UniiChat / "Opt chat" (Taelin gist).** Verified. One chat that never ends and acts as its own memory.
  - A cheap model folds messages into a binary tree of one-line summaries (≤ 512 B each).
  - Each turn sees a 64–128 KB view that is detailed for recent messages and coarse for older ones.
  - The agent can "zoom" a summary back down to the original messages. The view only grows at its end, so the prompt cache stays warm.
  - [gist](https://gist.github.com/VictorTaelin/91837951a5ce5b38f341ec1ba1df6449)
- **Continuous Memory Machines (Sakana).** A recurrent model with separate short-term and long-term matrix memories. Research-stage. [paper](https://academy.dair.ai/papers/continuous-memory-machines-2610.07907)
- **Living Weights / TensorFold v1.0.7.** Teaches facts as a parameter edit.
  - Adds about 10 neurons from a 64-neuron "pocket" beside each MoE shared expert in Nemotron 3.5 Lightning.
  - Edits are checked for recall and for leakage onto nearby questions, and are saved into the quantised safetensors, so the facts survive a restart.
  - Single personal facts worked on M5/M3 Ultra and GB10.
  - Document learning is still weak (6/16 on a 21-fact handbook), and Mac decoding was about 16% slower with 21 facts.
  - [release](https://github.com/ashhart/TensorFold/releases/tag/v1.0.7)

**Actions**
1. Prototype a CorpusMap-style entity layer over workspace files, using EmbeddingGemma 2 for entity resolution and search.
2. Try the UniiChat summary tree as the long-running session memory for the phone and desktop agent.
3. Treat Living Weights as an experimental, opt-in "teach" operation for local models only. Run it on a copy of the checkpoint, and keep retrieval as the default memory.

## 5. Evals, simulation and testing

- **Workers IO** (Inferred, [workersio](https://x.com/workersio/status/2107923334134710653), [workers.io](https://workers.io/)). A YC F2026 startup.
  - Deterministic VMs check a whole system against millions of production scenarios (fault interleavings and timings).
  - Every failure can be replayed exactly, which gives coding agents a feedback loop.
  - **Fit:** a possible verification backend for Gizzi Code changes.
- **Evals and the simulation machine** (Inferred, [Vtrivedy10](https://x.com/Vtrivedy10/status/2107300627617337801), LangChain Labs). The argument: to close the Sim2Real gap you first close the Real2Sim gap. Once you have a model of the world, agents can practise in it, and evals become training data and environments. Only a snippet was found. The [suraj_sharma14](https://x.com/suraj_sharma14/status/2106715961403273433) post was not found; that author writes AI-evals curricula.
- **MIMESIS (Meta).** A 9B user simulator trained on real human conversations. Agents trained against it beat agents trained against LLM-played users on 9 unseen user simulators. Use a realistic simulator, not an over-cooperative assistant model, as the "user" in agent evals.

**Actions**
1. Build an Agency API eval harness with restartable sandboxes and rubric checks (VERA-style) and a simulated user (MIMESIS-style).
2. Report pass@1 and pass@K for each model, so any Sharpening Tax regression shows up when choosing models.
3. Evaluate Workers IO for deterministic replay of failures in agent-written code.

## 6. Generative models for coding and local use

- **MAI-Code 1.1 Flash (Microsoft)** (Verified, [microsoft.ai](https://microsoft.ai/news/mai-code-1-1-flash-br-better-faster-at-a-quarter-of-the-cost/), [GitHub changelog](https://github.blog/changelog/2026-08-11-mai-code-1-1-flash-available-in-github-copilot/)).
  - Size: 137B total / 6.8B active per Microsoft (one model-card summary says 138B / 5B).
  - Price: $0.20 input / $0.02 cached / $1.20 output per 1M tokens, 73% lower than the previous model.
  - Copilot CLI: +22% on Terminal-Bench 2.1. Independent SWE-Bench Verified: 72.6%.
  - Runs locally as a 3-bit quant on 120 GB+ of RAM.
  - **Fit:** a cheap sub-agent model for Gizzi Code.
- **Qwen3.8-Flash-Next locally** (Inferred, [yume_arasaki](https://x.com/yume_arasaki/status/2107293459556225446), [recipe repo](https://github.com/yume-arasaki/Qwen3.8-Flash-Next-EXL3-DGX-Spark-recipe)).
  - A 125B-total / 6B-active MoE with 262K context.
  - About 50 tok/s on one DGX Spark (ExLlamaV3 + vLLM, about 80 GB). GGUFs need about 75 GB of RAM.
  - Workstation class only; it is **not** a phone model.

---

## Proposed next steps (priority order)

1. **Decision contract plus backends.** Define `decide()` on the Jev question schema. Benchmark d1-3B, d3-flash, Clef-flash and Jev on our routing and gating traces, and log every call for later Unsloth fine-tuning.
2. **Workflow DSL.** Adopt or replicate AgentRun's `call` / `judge` / agent-fallback / escalate pattern, backed by `decide()`. Code nodes must run in a sandbox.
3. **Execution reports and versioned harnesses.** These are the groundwork for harness learning, MASS/RSI-style loops and evals.
4. **Eval harness.** Sandboxes, rubrics and a simulated user, plus deterministic replay (Workers IO).
5. **Memory.** EmbeddingGemma 2, a CorpusMap entity layer and the UniiChat summary tree. Living Weights stays a research spike.

**Still open:** Flow 1 and the Charles_Y_Wu world-model post are unidentified. The tweet text was not read for the Liquid, Clef, Workers, Qwen, Jev and question-type items. Allowing x.com in the environment's network policy would make it possible to confirm them.
