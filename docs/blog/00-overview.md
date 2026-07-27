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
