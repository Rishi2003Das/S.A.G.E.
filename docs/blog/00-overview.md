# Engineering SAGE: an anticipatory intelligence system for India's crude oil supply chain

*How we turned reactive crisis response into an anticipatory pipeline, and the five design decisions behind it.*

India imports most of its crude oil, and a large share of it passes through a single stretch of water: the Strait of Hormuz. A serious disruption there is not a market inconvenience, it is a national supply-security event. The usual way institutions handle this is reactively: something happens, analysts scramble, models are run, and a recommendation lands hours later. SAGE is our attempt to make that process anticipatory instead of reactive.

This is the overview. It links out to five deep dives, each on one decision that shaped the system. The code is open at [github.com/BlueWaves-afk/Sage](https://github.com/BlueWaves-afk/Sage).

## What SAGE does

SAGE continuously ingests geopolitical and logistics signals from four always-on sensory sub-agents (vessel movements, news, sanctions, and prices), synthesizes them into a bitemporal knowledge graph and a human-readable wiki, and autonomously triggers disruption modelling, procurement rerouting, and strategic-reserve drawdown recommendations. The result turns a reactive crisis response into a managed, anticipatory one, with a 28× speedup from the moment a risk threshold is crossed to a ranked set of recommendations.

The shape is five systems. System 1 is the sensory layer, the only writer of raw signals. Everything it produces flows through a triage gate and a synthesis step into the knowledge base. Systems 2 to 4 are pure consumers: an ARIO disruption cascade, a procurement solver, and a reserve optimiser. System 5 renders it all as a geospatial digital twin. A LangGraph orchestration loop ties them together and runs the anticipatory sandbox.

## The five decisions worth reading about

Rather than one long article, we pulled out the five choices that were genuinely non-obvious, the ones where we picked the harder path for a reason. Each is a standalone post.

**Episode 1: Why raw signals never touch the vector store.** The four feeds are noisy and contradictory. Instead of embedding them directly, SAGE reconciles every signal against what it already believes before anything is stored, and a deterministic, source-aware triage gate (not an LLM) decides what deserves synthesis at all. The risk score itself is a calibrated model, validated out-of-sample across five real crises, not a hand-tuned heuristic. *[Read Episode 1 →](https://ridingbluewaves.hashnode.dev/why-raw-signals-never-touch-sages-vector-store)*

**Episode 2: Why SAGE keeps two knowledge graphs, not one.** A computable Graphiti graph for machines and an editable wikilink graph for humans. The redundancy is the point: one graph computes the supply gap, the other explains why today echoes 2019 and which refinery to watch. Plus why every edge is bitemporal and nothing is ever deleted. *[Read Episode 2 →](https://ridingbluewaves.hashnode.dev/why-sage-keeps-two-knowledge-graphs-not-one)*

**Episode 3: Answering a crisis in 50ms by computing a future that hasn't happened.** When risk is merely rising, SAGE forks a speculative sandbox and runs the whole cascade ahead of time with a GNN surrogate, so the answer is already staged when the threshold is crossed. Speculative execution, and the isolation rule that keeps a pre-computed future from being mistaken for the present. *[Read Episode 3 →](https://ridingbluewaves.hashnode.dev/answering-a-crisis-in-50ms-by-computing-a-future-that-hasnt-happened)*

**Episode 4: from_pretrained, not fit: loading a worldview instead of simulating one.** SAGE's foundational knowledge comes from a versioned, provenance-tracked context bundle. Facts load as facts, real fetched text grounds the prose, and the loader rejects any row without a source, a machine-checked "no simulated data" guarantee. *[Read Episode 4 →](https://ridingbluewaves.hashnode.dev/frompretrained-not-fit-loading-a-worldview-instead-of-simulating-one)*

**Episode 5: The graph that learns its own weights.** A crisis at one chokepoint should raise the risk of everything downstream of it. SAGE propagates risk along real, exposure-weighted dependency edges, and those weights start from sourced seed data and then get refined bitemporally from live signals. A knowledge base that learns, not a database that stores. *[Read Episode 5 →](https://ridingbluewaves.hashnode.dev/the-graph-that-learns-its-own-weights)*

## The thread running through all five

Looking back, the same instinct shows up in every one of these decisions. Push each choice to the layer that can make it most cheaply and most honestly. Let deterministic code route signals so an LLM is never asked a question the source already answers. Let the graph compute and the wiki explain, rather than forcing one structure to do both. Do expensive work speculatively and early, but quarantine the result until reality confirms it. Never let a number into the system without making it name where it came from. And when the model does have to reconcile a number from a live signal, let it reconcile evidence into a value — never invent one.

None of these are exotic techniques on their own. The interesting part was deciding, for a system meant to anticipate real-world supply shocks, where to spend a model call, where to trust plain code, and where to simply refuse to guess.

The full system is open at [github.com/BlueWaves-afk/Sage](https://github.com/BlueWaves-afk/Sage). This is part of an ongoing build series.

---

## 🧰 The stack, from zero — and what we chose it over

SAGE is a Python system with a deliberately unusual data core. The whole toolbox, and the road not taken for each:

- **Knowledge store — Graphiti + FalkorDB (a bitemporal knowledge graph) over a plain vector store or relational DB.** A vector store retrieves *similar text*; a supply-chain risk problem needs *entities and relationships over time* — which tanker, which strait, which sanction, when. **FalkorDB** is a fast Redis-module graph database (lighter to embed than **Neo4j**); **Graphiti** layers bitemporal versioning and entity resolution on top. Bitemporality lets us ask "what did we believe on date X," which a flat vector store can't.
- **Orchestration — LangGraph (an autonomous, stateful loop) over a linear chain or a fully autonomous agent framework.** LangGraph gives an explicit state graph with conditional edges — clean signal goes straight through, a threshold crossing triggers modelling and re-planning — within a bounded, auditable path (the same reasoning as the VIGIASearch series).
- **LLM — Amazon Bedrock (Nova Pro), managed, and triage-gated.** Managed inference over self-hosting; the triage gate (Episode 1) is what keeps the token bill bounded.
- **Ingestion transport — Redis (asyncio) queue** decoupling the four always-on sensory sub-agents (AIS, news, sanctions, prices) from the synthesis core, so a news burst doesn't block price ingestion.
- **ML — PyTorch + torch-geometric (GNNs) + scikit-learn** for the graph that learns its own weights (Episode 5).
- **API — FastAPI**, served today from a single EC2 instance.

## 🚢 From demo to production

The live demo is honest about its limits: one **EC2 free-tier** box with Google OAuth disabled (no public callback domain). Production means naming the substrate:
- **Managed, clustered graph DB** with snapshots and provenance retention, not a single-node Redis module.
- **Autoscaling the sensory agents** and backpressure on the Redis queue with a dead-letter path for failed synthesis.
- **LLM spend as a first-class SLO** — the triage gate is the cost control; a production LLM pipeline without a spend gate is a runaway bill.
- **Auditability** — a system that recommends an SPR drawdown must trace every recommendation to dated evidence (the bitemporal graph is what makes that possible).

---

## 🎓 CS Fundamentals — study companion

*SAGE is a systems-design masterclass wearing a domain problem. This overview maps to **System Design**, **DBMS**, and **Software Architecture**; the episodes go deeper. Read this before interviews to talk fluently about designing an event-driven, data-intensive AI system.*

### System Design

- **Reactive vs anticipatory systems.** A *reactive* system responds after an event; an *anticipatory* one pre-computes likely futures so the response is already staged. SAGE's 28× speedup comes from doing expensive work *before* the trigger — the same idea as CPU speculative execution and cache prefetching, applied at the system level.
- **The "single writer" principle.** System 1 is the *only* writer of raw signals; Systems 2–5 are pure readers through a typed API. One writer, many readers is a classic way to eliminate whole classes of concurrency and consistency bugs — the write path is the only place invariants must hold.
- **Pipeline / staged architecture.** SENSE → TRIAGE → SYNTHESIZE → (SANDBOX) → SCENARIO → PROCURE → RESERVE is a staged dataflow, each stage independently reasoned about and scaled. This is the backbone pattern of data-intensive systems.
- **Layered separation of concerns.** Five "systems," each with a single responsibility, coupled only through explicit contracts. This is the monolith-vs-modular tradeoff resolved toward modular-with-strict-interfaces.

**Interview Q&A.**
1. *What's the difference between reactive and anticipatory/speculative architectures?* → Respond-after vs precompute-likely-futures; the latter trades wasted work for latency, like branch prediction.
2. *Why designate a single writer for shared state?* → Concentrates invariant-enforcement in one place, avoids write-write conflicts, simplifies consistency.
3. *How do you keep a multi-stage system from becoming a coupled mess?* → Explicit typed contracts between stages, one responsibility per stage, readers never reach into the writer's internals.

### DBMS / Data systems (preview)
- SAGE stores knowledge in **three stores** (episodic log, semantic graph, human wiki) — a polyglot-persistence design where each store fits a different access pattern. Episodes 2 and 4 go deep on graph + vector + bitemporal modelling and provenance.

### ⚖️ This vs That — the architecture decisions, and the roads not taken

| Decision | Alternatives | Why this choice |
|---|---|---|
| **Anticipatory (speculative) pipeline** | Purely reactive: run the models when the crisis fires | Reactive means the user waits several seconds *after* the threshold crosses. Precomputing during the "rising" window makes the answer appear in ~300ms. You spend cheap background compute to buy latency when it matters. |
| **Five separated systems** | One monolithic agent/service | A monolith is simpler to start but couples ingestion, reasoning, and serving; a change in one risks all. Strict-contract modules let the four sub-systems be built and reasoned about independently. |
| **Single writer + typed read API** | Every agent reads/writes the graph directly | Shared write access to a knowledge graph is a consistency nightmare. One writer (System 1) + read-only consumers keeps the truth in one place. |

**The one to defend:** *anticipatory vs reactive.* Most systems are reactive because it's simpler. The senior insight: **latency at the moment of crisis is the product**, and you can pay for it in advance with speculative execution — as long as you quarantine the speculative result until reality confirms it (Episode 3). That single tradeoff is SAGE's headline.
