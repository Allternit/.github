# Agency API — research update (2026-10-12)

Reading list and notes for the next revision of the Allternit Agency API: the agent
runtime layer in [`allternit-platform`](https://github.com/Allternit/allternit-platform)
that routes decisions, runs workflows, keeps memory and evaluates agents.

Status key: **Read** means the source was read and summarised here. **Link only** means
the link was saved but not read (most X/Twitter posts could not be fetched), so the note
is just the topic label from the original list.

---

## 1. Fast local decision models (router / judge layer)

The Agency API makes many small yes/no/route decisions per task. A local model in the
millisecond range could make those calls without a round trip to a frontier model.

| Item | Source | Status | Relevance |
|------|--------|--------|-----------|
| LiquidAI d1 3B: local decisions in about 8 ms | [atomic_chat_hq](https://x.com/atomic_chat_hq/status/2107931155840299417), [liquidai](https://x.com/liquidai/status/2105003475017085167) | Link only | Candidate for an on-device `judge` / router backend |
| vLLM Semantic Router "Decision 3.0" collection | [HF collection](https://huggingface.co/collections/vllm-sr/decision-30) | Link only (host unreachable from build env) | Open decision/routing models to benchmark against d1 |
| Qwen 3.8 Flash on local devices | [yume_arasaki](https://x.com/yume_arasaki/status/2107293459556225446) | Link only | Local general model for the desktop/phone tiers |
| MAI-Code 1.1 Flash (Microsoft): reported as better and faster at a quarter of the cost | [microsoft.ai](https://microsoft.ai/news/mai-code-1-1-flash-br-better-faster-at-a-quarter-of-the-cost/) | Link only (host unreachable) | Low-cost coding model for Gizzi Code sub-tasks |
| Typesafe Jev | [Brankotrcek](https://x.com/Brankotrcek/status/2109419442643845309) | Link only | Typed decision calls; see Agentrun below |

**Action:** define a single `decide(question, criteria) -> {decision, confidence}` contract
in the API so the backend can change between local d1, a vLLM-SR model or a hosted model
without changing workflows.

## 2. Workflow DSL and orchestration

**Agentrun: the Agentrun Workflow DSL** ([repo](https://github.com/Parcha-ai/agentrun), [_aj](https://x.com/_aj/status/2102061534956662818)). **Read.**

- A workflow language that runs inside an existing agent stack. The host app keeps its own tools, model access, permissions and budgets.
- Workflows are JSON or a TypeScript builder with Zod contracts. Types are inferred, and state paths are checked at runtime.
- Nodes: `call` runs a tool, and `judge` asks a typed yes/no/uncertain question that returns a confidence score. It also has sub-workflows, parallel map, bounded loops, and an agent fallback when the fixed path fails.
- Example pattern: search, then judge (accept "yes" at ≥ 0.8), then one agent retry, then escalate to a human.
- Caveats: code nodes run with process privileges, so workflows from untrusted authors need a sandbox. A typed decision does not prove the answer is correct. Current version is `0.1.0-beta.4`.

**Fit:** this is close to the shape the Agency API needs. Adopt the `call` / `judge` / fallback / escalate
pattern, and back `judge` with the local decision layer from §1.

Related, link only: Workers ([workersio](https://x.com/workersio/status/2107923334134710653)),
Flow 1 ([arafatkatze](https://x.com/arafatkatze/status/2107165909639160268)),
Cloudflare "clef" ([MiaAI_lab](https://x.com/MiaAI_lab/status/2107134733394325810)) as a possible edge runtime.

## 3. Harness engineering (papers, from DAIR.AI Top AI Papers of the Week)

The main point across these papers is that **the harness is a learnable artifact**. Large gains come
from changing orchestration code without changing model weights. That makes the Agency API itself the lever.

- **Harness Learning (CMU).** A 4B proposer model edits harness code from execution reports and is trained with RL on the harness score. The solver model stays frozen. It beats its 35B teacher and transfers to task families it never saw. [paper](https://academy.dair.ai/papers/harness-learning-enables-generalizable-test-time-adaptation-2609.35738)
- **HERMES.** Each repository component gets a resident LLM ("Dev-Primitives"), plus dependency-aware activation and failure diagnosis. A repo migration task went from 6.5% to 31.0% with the same model. Qwen3-8B primitives stay within 4.5% of a frontier-only setup. [paper](https://academy.dair.ai/papers/harness-engineering-for-software-engineering-via-modular-executable-dev-primitiv-2610.07832) — relevant to Gizzi Code.
- **VeriHarness (Google).** A same-model verifier with a disagreement resolver and a consensus challenger. It beats majority voting and adds about 6 points over a single rollout. [paper](https://academy.dair.ai/papers/veriharness-scaling-agentic-verification-for-long-horizon-tasks-2610.00972)
- **VERA (NVIDIA).** More than 9,000 restartable, rubric-verified sandboxes. It co-evolves the model and the harness skills, and accepts a harness edit only if it gains at least 5 points on the dev set. [paper](https://academy.dair.ai/papers/vera-scaling-verifiable-environments-for-agentic-co-evolution-2610.05923)
- **MASS (Sakana).** One base model acts as agent, evaluator and optimiser. Evolutionary search finds multi-agent workflows, then the model is fine-tuned on its own traces. [paper](https://academy.dair.ai/papers/recursive-self-improvement-through-multi-agent-self-supervision-2610.12176), [SakanaAILabs](https://x.com/SakanaAILabs/status/2109385006603006435), [hardmaru](https://x.com/hardmaru/status/2109385496237678844)
- **Agent Plasticity (Meta).** Measures how well an agent turns experience into held-out gains per dollar, with weights frozen and agent-written artifacts carried between runs. Slow learners ignore their own artifacts. [paper](https://academy.dair.ai/papers/agent-plasticity-measuring-self-improvement-through-experience-2610.08902)
- **Sharpening Tax (Meta).** RL post-training raises pass@1 but lowers pass@K coverage. Base models sometimes solve tasks that their post-trained versions can no longer solve. [paper](https://academy.dair.ai/papers/sharpening-tax-in-post-training-2610.01509)
- RSI Anything, link only: [LingYang_PU](https://x.com/LingYang_PU/status/2109468200866074817)

**Actions**
1. Version harnesses (prompts, tool graph, workflow DSL) as data, so a proposer can diff, score and roll them back.
2. Give every run a structured execution report (inputs, tool calls, failures, score). Harness Learning, VERA and Plasticity all depend on one.
3. Add a verify step that targets agreement as well as disagreement, following VeriHarness, before results are accepted.
4. Persist agent-written artifacts across runs, and measure whether later runs actually reuse them.

## 4. Retrieval, memory and context

- **CorpusMap (Microsoft).** Builds an offline entity graph over the corpus and gives each recurring entity a page. Reported +6.4 to +11.7 answer quality with 34–57% fewer input tokens than raw agentic search, and the map can be built without LLM calls. [paper](https://academy.dair.ai/papers/follow-the-entities-a-corpus-map-for-agentic-search-2609.37226) — strong fit for workspace search.
- **UniiChat / "Opt chat" (Taelin gist).** **Read.** One chat that never ends and acts as its own memory. A cheap model folds messages into a binary tree of one-line summaries (≤ 512 B each). Each turn sees a 64–128 KB view that is detailed for recent messages and coarse for older ones. The agent can "zoom" a summary back down to the original messages. The view only grows at its end, so the prompt cache stays warm. [gist](https://gist.github.com/VictorTaelin/91837951a5ce5b38f341ec1ba1df6449)
- **Continuous Memory Machines (Sakana).** A recurrent model with separate short-term and long-term matrix memories. Research-stage. [paper](https://academy.dair.ai/papers/continuous-memory-machines-2610.07907)
- **Living Weights / TensorFold v1.0.7.** Teaches facts as a parameter edit. It adds about 10 neurons from a 64-neuron "pocket" beside each MoE shared expert in Nemotron 3.5 Lightning. Edits are checked for recall and for leakage onto nearby questions, and are saved into the quantised safetensors, so the model gets slightly wider and the facts survive a restart. Single personal facts worked on M5/M3 Ultra and GB10. Document learning is still weak (6/16 on a 21-fact handbook), and Mac decoding was about 16% slower with 21 facts. [release](https://github.com/ashhart/TensorFold/releases/tag/v1.0.7)
- **EmbeddingGemma 2 as the base embedder.** Link only: [mylifcc](https://x.com/mylifcc/status/2107521708223758818), [googlegemma](https://x.com/googlegemma/status/2107502533992464482), [KShivendu_](https://x.com/KShivendu_/status/2107558721568321784), [xueyu1125](https://x.com/xueyu1125/status/2109301901783429283), [mrm8488](https://x.com/mrm8488/status/2109406679045423394)

**Actions**
1. Prototype a CorpusMap-style entity layer over workspace files, using EmbeddingGemma 2 for entity resolution and search.
2. Try the UniiChat summary tree as the long-running session memory for the phone and desktop agent.
3. Treat Living Weights as an experimental, opt-in "teach" operation for local models only. Run it on a copy of the checkpoint, and keep retrieval as the default memory.

## 5. Evals, simulation and training

- **MIMESIS (Meta).** A 9B user simulator trained on real human conversations. Agents trained against it beat agents trained against LLM-played users on 9 unseen user simulators. Use a realistic simulator, not an over-cooperative assistant model, as the "user" in agent evals.
- Evals and simulation machine, link only: [Vtrivedy10](https://x.com/Vtrivedy10/status/2107300627617337801), [suraj_sharma14](https://x.com/suraj_sharma14/status/2106715961403273433)
- Question types, link only: [JeffreyBiles](https://x.com/JeffreyBiles/status/2107211222320021722)
- Training with Unsloth, link only: [UnslothAI](https://x.com/UnslothAI/status/2107868866361930236). Candidate pipeline for fine-tuning a small proposer or router model.
- World model, link only: [Charles_Y_Wu](https://x.com/Charles_Y_Wu/status/2107134454762496397)

**Actions**
1. Build an Agency API eval harness with restartable sandboxes and rubric checks (VERA-style) and a simulated user (MIMESIS-style).
2. Report pass@1 and pass@K for each model, so any Sharpening Tax regression shows up when choosing models.

---

## Proposed next steps (priority order)

1. **Decision contract plus local backend.** Define `decide()`, then benchmark LiquidAI d1 against the vLLM-SR Decision 3.0 models on latency and accuracy.
2. **Workflow DSL.** Evaluate Agentrun directly against an in-house equivalent. Code nodes must run in a sandbox.
3. **Execution reports and versioned harnesses.** These are the groundwork for harness learning and evals.
4. **Eval harness.** Sandboxes, rubrics and a simulated user.
5. **Memory.** A CorpusMap entity layer and the UniiChat summary tree. Living Weights stays a research spike.

Open item: read the "link only" X posts and fill in their notes. They could not be fetched from the build environment.
