# Episode 4: from_pretrained, not fit: loading a worldview instead of simulating one

*How SAGE instantiates its foundational knowledge from a versioned, provenance-tracked context bundle, and why every row needs a source.*

A live intelligence system has a cold-start problem. Before the first signal arrives, SAGE already needs to know that Jamnagar is a refinery, that a large share of India's crude transits Hormuz, that Arab Light has a certain API gravity, and that the 2019 tanker attacks are a relevant precedent. Where does that knowledge come from, and how do you keep it honest? This post is about the answer we settled on: a context bundle.

This is Episode 4 of 5 in the Engineering SAGE series. The full system overview is in the master post.

## A load, not a train

The mental model we kept coming back to is that instantiating SAGE's foundational knowledge is a load, not a train. It is `from_pretrained`, not `fit`. Live signals from the sensory agents layer continual updates on top, but the starting worldview is loaded from a versioned, provenance-tracked artifact rather than learned or, worse, invented.

That artifact is a `.context` bundle: a directory you can diff, swap, and contribute to. Point SAGE at `india-energy-2026.context` and it knows the 2026 Indian crude picture. Point it at a different bundle and it knows a different year, region, or domain. The worldview is data, not code.

## Three kinds of knowledge, loaded three different ways

The bundle has three layers, and the important decision was to load each one differently rather than shove everything through the same path.

**Facts** are structured ground truth in CSVs: refinery capacity, crude assays, throughput shares. These are written directly as graph attributes. No LLM reconciles a known number, because there is nothing to reconcile. If Jamnagar's capacity is 1.40 mbpd, that is a value to load, not a question to reason about.

**Sources** are real fetched text: the EIA or PPAC or assay document behind an entity. These are not written as facts. They are the grounding evidence the LLM is allowed to summarise. When SAGE authors an entity's narrative, it writes only from that fetched text plus the structured facts, which is what keeps the narrative from drifting into parametric memory.

**Narratives** are hand-authored Markdown with `[[wikilinks]]`, and they go through the exact same synthesis path that live signals use. This is deliberate. The prose that describes why Hormuz matters is reconciled, linked, and embedded by the same machinery that will later reconcile a breaking news signal, so the foundational pages and the live pages are structurally identical.

## The rule that makes it trustworthy: no unsourced row

The single most important line in the loader is that it rejects any row without a source. Every facts row carries a tier (`real`, `derived`, or `estimated`) and a source. A row with a bare number and no provenance is a hard validation error, not a warning.

This is the machine-checked version of a "no simulated data" guarantee. For a system whose whole pitch is anticipating real supply shocks, a plausible-looking made-up number is the worst kind of failure, because it is invisible. Forcing every value to name its origin means a reviewer can audit the entire data footprint: real values trace to EIA, PPAC, ISPRL, and published assays; derived values state their method; estimated values are labelled honestly as estimates. Nothing hides.

## Why no constant is hardcoded in agent code

There is a related decision that follows from the same instinct. No numeric constant that affects model behaviour lives in agent code. The ARIO elasticities, the TOPSIS ranking weights, the SPR drawdown limits, all of them live in the bundle's parameter CSVs.

The reason is that hardcoded constants are undocumented assumptions. A price elasticity buried in a Python file is a claim about the world with no source and no easy way to change it. Moving every such value into a sourced CSV means tuning the model is editing data and bumping a version, not patching code, and every assumption is visible and attributed in one place.

## Upgrading without losing the live state

Because the worldview is a versioned artifact, updating it is a first-class operation. Point SAGE at a newer bundle and it upserts the structural facts and model parameters while preserving everything the live system produced: risk scores, dynamically registered vessels, every episode, and the synthesised wiki pages. Changed entities get a short context-update note appended rather than a full rewrite, and an audit episode records the transition. The static worldview can move forward without erasing the dynamic history layered on top of it.

## Takeaway

Treating foundational knowledge as a loadable, sourced, versioned bundle solved three problems at once: the cold start, the hallucination risk, and the "where did this number come from" question. Facts load as facts, real text grounds the prose, and nothing enters the graph without naming its origin.

The code is open at [github.com/BlueWaves-afk/Sage](https://github.com/BlueWaves-afk/Sage).

*Engineering SAGE · Episode 4 of 5 — Previous: Episode 3. Next: Episode 5, The graph that learns its own weights.*
