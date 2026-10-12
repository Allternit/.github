# Agency API — research update (2026-10-12)

Reading list and notes for the next revision of the Allternit Agency API: the agent
runtime layer in [`allternit-platform`](https://github.com/Allternit/allternit-platform)
that routes decisions, runs workflows, keeps memory and evaluates agents.

**Source status.** Every linked post was read directly (via the fxtwitter API) on 2026-10-12, and the
Hugging Face IDs were checked against the HF API. Each item is tagged:
- **Verified**: the post or primary source was read.
- **Inferred**: some figures come from secondary coverage only.

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
| **TypeSafe Jev** (hosted) | n/a | ~100 ms for most calls; claims 193x faster and 445x cheaper than LLMs | Proprietary API | TypeSafe API, Cloudflare, OpenRouter | Inferred (secondary) |
| **Liquid d1-3B** | 3B (text + image) | 8 ms on RTX 3090 (Atomic Chat) or 4090; 16 ms on Jetson AGX Thor; 50 ms on Jetson Orin Nano | LFM Open v1.0 (reportedly free below $10M revenue) | HF `LiquidAI/d1-3B` (+ `-GGUF`, `-w8a8`), Liquid API ([console.liquid.ai](http://console.liquid.ai)) | Verified |
| **Liquid d1-omni-600M** | 600M (text + image, or text + audio) | n/a | LFM Open v1.0 | Experimental | Verified |
| **Atomic Chat d1-3B GGUF** | Q8_0 2.9 GB; AD-Q4_K_M 1.7 GB (~97% agreement with Q8) | n/a | as d1 | HF `AtomicChat/d1-3B-GGUF`, llama-server `/v1/systemone` | Inferred |
| **Cloudflare Clef** | 27B (Qwen3.8-27B base), ~85 GB VRAM | 209 ms ("4x faster than Jev") | Apache 2.0 | HF `Cloudflare/clef`, Workers AI | Verified |
| **Cloudflare Clef-flash** | 9B (Qwen3.5-9B base), ~41 GB VRAM | 39 ms | Apache 2.0 | Workers AI | Inferred |
| **vLLM-SR d3** | 27B, text/JSON + 4 images | n/a; self-reported Decision Index 0.3 public score 64.24 | Apache 2.0 | HF `vllm-sr/d3` | Verified |
| **vLLM-SR d3-flash** | 9B | 19.4 ms (text) | Apache 2.0 | HF `vllm-sr/d3-flash`; collection also lists `d3-mini`, `d3-nano`, `d3-lite`, `d3-edge` | Verified |
| **mrm8488 Typed Decision** | EmbeddingGemma 2 base (~740M) | n/a | Apache 2.0 | HF `mrm8488/embeddinggemma-2-ft-typed-decision`; 76.4% on typed decisions, about 6 points behind a 27B LLM with ~100x fewer parameters | Verified |

Notes:
- **Liquid d1** ([announcement](https://x.com/liquidai/status/2105003475017085167)) claims to be the first model to beat Jev on HF's Decision Index, with better multilingual results, prompt-injection robustness and long-input handling. Docs: [decision models](https://docs.liquid.ai/lfm/models/decision-models), [migration guide](https://docs.liquid.ai/guides/decision-model-guide), [road-decider demo](https://github.com/Liquid4All/cookbook/tree/main/examples/road-decider).
- **Atomic Chat demo** ([post](https://x.com/atomic_chat_hq/status/2107931155840299417)): playing Snake, local d1 on one RTX 3090 decided in 8 ms per move (82 moves/s, 86 apples). OpenAI's Decision API took 345 ms per call and its snake ate 3 apples. The demo shows that latency matters for control loops.
- **How Clef works** ([MiaAI_lab](https://x.com/MiaAI_lab/status/2107134733394325810)): a frozen Qwen does one prefill pass, then a tiny schema head scores every answer option in parallel, with no text generated. It reached #1 on HF trending.
- **Clef** is API-compatible with Jev. It takes a state plus up to 64 typed questions, has 64k context, and returns a probability for every option. Cloudflare also announced an RL fine-tuning service. Reported prices of $0.24 / $0.09 per M input tokens come from secondary sources only.
- **Jev question types** ([JeffreyBiles](https://x.com/JeffreyBiles/status/2107211222320021722): the post is a video captioned "Jev opens up entirely new question types". The breakdown below comes from secondary sources, and field names were not checked against the docs):
  - `choice`: up to 255 options; returns the choice, the distribution and a confidence.
  - `score`: an ordered rubric; returns the tier, the distribution and a confidence.
  - yes/no ("noul"): returns a single probability.
- **Jev + Opus support workflow** (Verified, [Brankotrcek](https://x.com/Brankotrcek/status/2109419442643845309), relaying TypeSafe's CEO; claims a 90% cut in support cost). The pattern:
  1. Check saved results first.
  2. Jev routes each ticket to one of three paths: approved response, Opus investigation, or human review.
  3. Keep a decision log of route, reason and outcome, so wrongly rejected tickets can be audited.
  4. On failure, send Opus only the failed check and its evidence, not a full re-run.

  This is the reference design for Agency API routing. TypeSafe raised $870M at a $7.5B valuation on Oct 9 (secondary).
- **Unsloth: train your own decision model** (Verified, [tweet](https://x.com/UnslothAI/status/2107868866361930236), [releases](https://github.com/unslothai/unsloth/releases)).
  - Turns any text or vision LLM (Qwen3.8, Gemma 4) into a decision model by fine-tuning with a Clef-style head.
  - Qwen3.5-0.8B went from 20.7% to 74.3% aggregate accuracy on 3 decision benchmarks, using 4 GB of VRAM.
  - The tweet says LoRA r=64 for one epoch lifted downstream accuracy from 30–37% to 78%. The docs recommend 2 epochs, r=16 and learning rate 2e-4. [Guide and notebooks](https://unsloth.ai/docs/basics/train-your-own-decision-model-with-unsloth).
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

**AgentRun, the Jev-powered workflow DSL** (Verified, [repo](https://github.com/Parcha-ai/agentrun), [_aj](https://x.com/_aj/status/2102061534956662818)).

- The tweet's key idea: **the harness learns the job as it runs, moving steps from LLM calls into code.**
- Its numbers: 100,000 compliance alerts cost more than $290K on Opus 5 and less than $26K with AgentRun (about 91% lower).
- Companion article: "AgentRun: a harness for repetitive knowledge work".

- A workflow language that runs inside an existing agent stack. The host app keeps its own tools, model access, permissions and budgets.
- Workflows are JSON or a TypeScript builder with Zod contracts. Also on PyPI as `parcha-agentrun`, with a Pi extension `@parcha/agentrun-pi`.
- Nodes:
  - `call` runs a tool.
  - `judge` asks Jev a typed question and gets back a decision with a confidence score.
  - Also: sub-workflows, parallel map, bounded loops, and an agent fallback when the fixed path fails.
- Example pattern: search, then judge (accept "yes" at ≥ 0.8), then one agent retry, then escalate to a human.
- Caveats: code nodes run with process privileges and need a sandbox for untrusted workflow authors. A typed decision does not prove the answer is correct. Current version is `0.1.0-beta.4`.

**Fit:** this is the pattern the Agency API needs. Cheap decision models gate each step, and expensive agents run only as the fallback. Back `judge` with the §1 `decide()` contract.

**flow-1** (Verified, [arafatkatze](https://x.com/arafatkatze/status/2107165909639160268), quoting [@skull8888888888](https://x.com/skull8888888888)).
- An RL-trained model that **finds errors in agent traces**.
- Claimed to match GPT-6-sol at trace analysis while being 23x cheaper, and 25% cheaper to run than GPT-6-luna.
- The pitch is monitoring every agent run instead of sampling.
- Ara's broader point: a small base model, post-trained on curated data from thousands of frontier-model runs, beats frontier models on a narrow task at a fraction of the cost.
- **Fit:** a trace-monitoring model for Agency API runs. It feeds the execution reports in §3 and the evals in §5.

## 3. Harness engineering and self-improvement

The main point across these papers is that **the harness is a learnable artifact**. Large gains come from changing orchestration code
without changing model weights. That makes the Agency API itself the lever. (Paper summaries come from DAIR.AI's Top AI Papers of the Week.)

- **Harness Learning (CMU).** A 4B proposer model edits harness code from execution reports and is trained with RL on the harness score. The solver model stays frozen. It beats its 35B teacher and transfers to task families it never saw. [paper](https://academy.dair.ai/papers/harness-learning-enables-generalizable-test-time-adaptation-2609.35738)
- **HERMES.** Each repository component gets a resident LLM ("Dev-Primitives"), plus dependency-aware activation and failure diagnosis. A repo migration task went from 6.5% to 31.0% with the same model. Qwen3-8B primitives stay within 4.5% of a frontier-only setup. [paper](https://academy.dair.ai/papers/harness-engineering-for-software-engineering-via-modular-executable-dev-primitiv-2610.07832) — relevant to Gizzi Code.
- **VeriHarness (Google).** A same-model verifier with a disagreement resolver and a consensus challenger. It beats majority voting and adds about 6 points over a single rollout. [paper](https://academy.dair.ai/papers/veriharness-scaling-agentic-verification-for-long-horizon-tasks-2610.00972)
- **VERA (NVIDIA).** More than 9,000 restartable, rubric-verified sandboxes. It co-evolves the model and the harness skills, and accepts a harness edit only if it gains at least 5 points on the dev set. [paper](https://academy.dair.ai/papers/vera-scaling-verifiable-environments-for-agentic-co-evolution-2610.05923)
- **MASS (Sakana).** One base model (Qwen3.6-27B) acts as agent, evaluator and optimiser. Evolutionary search finds multi-agent workflows, then the model is fine-tuned on its own traces. [paper](https://academy.dair.ai/papers/recursive-self-improvement-through-multi-agent-self-supervision-2610.12176)
  - **Verified:** code at [SakanaAI/mass](https://github.com/SakanaAI/mass). It includes the pipeline CLI, workflow search, LoRA training, ScienceAgentBench and MLR-Bench adapters, and 12 synthetic tasks. Model weights are not included and the README names no license.
  - Paper findings from the [tweets](https://x.com/SakanaAILabs/status/2109385006603006435):
    - Training only on task-solving data also improves the model as optimiser and as evaluator.
    - Multi-agent traces are better training data than single-agent traces.
    - The optimiser, not the evaluator, can be what limits the speed of improvement.
    - Sakana flags a safety risk: when a model is its own evaluator, shared blind spots can be reinforced.
  - Repo results across cycles L0 → L2: gains on ScienceAgentBench (28.8 → 31.7) and MLR-Bench (1.71 → 2.25), but small drops on Terminal-Bench 2.0 (0.375 → 0.352) and SWE-bench Verified (0.626 → 0.611).
  - Tweets: [SakanaAILabs](https://x.com/SakanaAILabs/status/2109385006603006435), [hardmaru](https://x.com/hardmaru/status/2109385496237678844).
- **RSI-Anything (PhAI Labs, Ling Yang et al.).** Verified via the [repo](https://github.com/Gen-Verse/RSI-Anything) (Apache-2.0, `rsi` CLI) and [arXiv 2610.13912](https://arxiv.org/abs/2610.13912). Uses predictive world models (built on JEPA-Anything) to choose actions and training data for recursive self-improvement.
  - Code repair on CodeContests-200: 26.3% → 31.2%, with the coder model fixed.
  - Lab planning: 21.80% → 46.01% over five policy versions.
  - MiniWoB++: 99% success with 39.4% fewer actions.
  - OSWorld: +18.4% relative, versus +12.3% for RLAnything.
  - Tweet: [LingYang_PU](https://x.com/LingYang_PU/status/2109468200866074817).
- **JEPA-Anything** (Verified, [Charles_Y_Wu](https://x.com/Charles_Y_Wu/status/2107134454762496397)). One world model that works across molecules, cells, fluids, patients, robots and weather. It is the base that RSI-Anything builds on. [arXiv 2609.20800](https://arxiv.org/abs/2609.20800), [code](https://github.com/Gen-Verse/JEPA-Anything).
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
  - About 567 MB quantised on a phone. Official HF id: `google/embeddinggemma-2` ([mylifcc](https://x.com/mylifcc/status/2107521708223758818)); community GGUF: `unsloth/embeddinggemma-2-GGUF`.
  - **Compression** ([KShivendu_](https://x.com/KShivendu_/status/2107558721568321784)): 10M vectors take about 30 GB as float32. That drops to about 1 GB with 1-bit TurboQuant (99% of nDCG@10 kept), and about 0.4 GB with MRL to 256 dims (94.5% kept with rescoring). About 77x less RAM.
  - **Local search app** ([xueyu1125](https://x.com/xueyu1125/status/2109301901783429283)): an open-source Mac app, "DigUp", uses it to search file contents, images and audio, and is reported as much better than Spotlight.
  - **As a decision model** ([mrm8488](https://x.com/mrm8488/status/2109406679045423394)): fine-tuned into a typed-decision model (see §1).
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

- **Workers IO** (Verified launch post, [workersio](https://x.com/workersio/status/2107923334134710653), [workers.io](https://workers.io/)). A YC F2026 startup.
  - Deterministic VMs check a whole system against millions of production scenarios (fault interleavings and timings).
  - Every failure can be replayed exactly, which gives coding agents a feedback loop.
  - **Fit:** a possible verification backend for Gizzi Code changes.
- **Evals and the simulation machine** (Verified, [Vtrivedy10](https://x.com/Vtrivedy10/status/2107300627617337801), LangChain Labs).
  - To close the Sim2Real gap you first close the Real2Sim gap. Evals must mirror the real world: data schemas, sizes, API shapes, user-request types, tools and tool responses, built from real trajectories.
  - Evals then serve three roles: (1) the measuring device, (2) the data you hill-climb on, and (3) the infrastructure for a "simulation machine" that answers what-if questions ("cheaper?", "vague users?").
  - Definitions from the quoted thread:
    - An eval is a grader applied to a trace.
    - Graders are deterministic, reference-answer, rubric, or trajectory graders.
    - Run online and offline evals, and "promote" bad production traces into offline cases.
    - Run each case k times, requiring all-of-k for high-stakes behaviour and n-of-k elsewhere.
- **15 eval projects** (Verified, [suraj_sharma14](https://x.com/suraj_sharma14/status/2106715961403273433)). A practical checklist; the most relevant items:
  - a trajectory grading engine
  - a shadow-routing comparator (mirror 5% of traffic to a new model)
  - a calibrated LLM judge
  - a CI regression gate (block on a >2% drop)
  - a counterfactual replay debugger
  - a red-team fuzzer
  - a production drift monitor
  - a cost/quality Pareto view of System-One versus System-Two routing
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
- **Qwen3.8-Flash-Next locally** (Verified, [yume_arasaki](https://x.com/yume_arasaki/status/2107293459556225446)).
  - A 180B MoE with about 6B active per token: 512 experts per layer, the router picks 10, plus one shared expert. 51B of the parameters are an n-gram lookup table that streams from SSD. 262K context.
  - The author places it near Opus 4.8 on agentic and coding work: it scores 40 versus 42 on the Artificial Analysis index and wins on Terminal-Bench 4.0. It is weaker on HLE and SWE-bench Pro.
  - **Why it runs locally:** a VRAM / RAM / SSD expert-cache hierarchy plus MTP speculative decoding.
  - **The Strata engine** (MIT) serves an **Anthropic-compatible API on localhost**, so Claude Code and Codex CLI can point at it directly.
  - Reported speeds:

    | Hardware | Speed |
    |----------|-------|
    | RTX 4060 laptop (8 GB VRAM) | about 30 tok/s |
    | RTX 5090 | 160 tok/s (vs 15 on llama.cpp) |
    | One DGX Spark | about 70–107 tok/s |

  - Macs need 64 GB+ unified memory (TensorFold or oMLX). Below that, use Qwen3.8-27B dense.
  - The 32 GB-RAM floor is claimed but unverified.
  - **Fit:** a strong local "System Two" model for desktop users with gaming-class hardware. Not for phones.

---

## Proposed next steps (priority order)

1. **Decision contract plus backends.** Define `decide()` on the Jev question schema. Benchmark d1-3B, d3-flash, Clef-flash and Jev on our routing and gating traces, and log every call for later Unsloth fine-tuning.
2. **Workflow DSL.** Adopt or replicate AgentRun's `call` / `judge` / agent-fallback / escalate pattern, backed by `decide()`. Code nodes must run in a sandbox.
3. **Execution reports and versioned harnesses.** These are the groundwork for harness learning, MASS/RSI-style loops and evals.
4. **Eval harness and monitoring.** Trajectory graders, sandboxes, rubrics and a simulated user, plus deterministic replay (Workers IO) and a flow-1-style trace monitor on every run.
5. **Memory.** EmbeddingGemma 2, a CorpusMap entity layer and the UniiChat summary tree. Living Weights stays a research spike.

**Still open:** the JeffreyBiles post is only a video, so its contents are unconfirmed. Clef's prices and TypeSafe's funding come from secondary coverage only.
