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

---

## 🧰 The context stack, from zero — and what we chose it over

- **Load a worldview, don't simulate one.** Instead of making the model reason from a blank slate, we assemble a rich **context bundle** from the graph — relevant entities, their relationships, recent dated evidence, and provenance — and hand *that* to the model. It's RAG's cousin, but graph-grounded rather than similarity-grounded.
- **GNN embeddings (torch-geometric)** encode the graph's *structure* into the bundle, so the model sees not just facts but how they connect.
- **Over fine-tuning / dumping raw docs.** Baking knowledge into weights is stale the day after training and unauditable; retrieval-and-assembly is current, cheaper, and every claim traces to a source.

## 🚢 From demo to production

- **Context-window budgeting** — pick the highest-value subgraph, don't overflow the prompt.
- **Freshness** — the bundle must reflect the latest evidence, not a cached snapshot.
- **Grounding/citation** so a recommendation is defensible against the exact evidence that produced it.

---

## 🎓 CS Fundamentals — study companion

*This episode is about **data provenance, schema validation, versioning, and reproducibility** — core **DBMS / data-engineering** topics — plus **Software Engineering** principles (config-as-data, no magic constants).*

### DBMS / Data Engineering

- **Provenance & lineage.** Every fact row carries a `tier` (real/derived/estimated) and a `source`; the loader **rejects any unsourced row** as a hard error. Data provenance = tracking where each value came from and how it was derived. It's what makes a system auditable and is increasingly a compliance requirement (you can trace every number to its origin).
- **Schema validation at the boundary.** Rejecting malformed/unsourced rows at load time is **fail-fast validation** — catch bad data at ingestion, not three stages downstream where the error is unattributable. (Same principle as constraints/CHECKs in SQL and schema validation in pipelines.)
- **Versioned artifacts & reproducibility.** The `.context` bundle is a versioned, diffable directory; `bundle_version` in a manifest gates upgrades. Point at `2026` vs `2027` and you load a different worldview. This is **immutable, versioned data artifacts** — the data equivalent of a pinned dependency / container image — and it makes runs reproducible.
- **Idempotent, non-destructive migrations.** The upgrade *upserts* structural facts while *preserving* dynamic state (risk scores, episodes, learned edges), appends a "context update" note, and writes an audit episode. That's a safe **schema/data migration**: forward-only, auditable, preserving live data.
- **Seed data vs live data.** Foundational knowledge is loaded (`from_pretrained`), then live signals layer on top — a clean separation of **seed/reference data** from **transactional data**, each managed differently.

### Software Engineering / System Design

- **Configuration as data, not code.** No numeric constant that affects behaviour lives in agent code — ARIO elasticities, TOPSIS weights, SPR limits all live in sourced CSVs. A magic constant buried in code is an undocumented, unsourced, hard-to-change assumption. Externalising config makes tuning *edit-data-and-version*, not *patch-and-redeploy*.
- **Dependency injection of knowledge.** Loading a worldview instead of hardcoding it is DI applied to knowledge: swap the bundle, get a new domain, with zero code change.

**Interview Q&A.**
1. *What is data provenance/lineage and why does it matter?* → Tracking origin and transformation of each value; enables audit, debugging, compliance, trust.
2. *Why validate/reject bad data at ingestion (fail-fast)?* → Errors are cheapest to attribute at the boundary; downstream, a bad value is untraceable and corrupts everything.
3. *How do you make a data-driven system reproducible?* → Immutable versioned data artifacts + pinned config + recorded provenance; a run is defined by (code version, data version).
4. *Why move constants out of code into config?* → Visibility, sourcing, and change without redeploy; a hardcoded elasticity is an untested claim about the world.
5. *How do you migrate a schema without losing live data?* → Forward-only upserts, preserve dynamic rows, append an audit record, keep it idempotent.

### ⚖️ This vs That — the architecture decisions, and the roads not taken

| Decision | Alternatives | Why this choice |
|---|---|---|
| **Load a versioned worldview (`from_pretrained`)** | Hardcode entities/relationships; or "learn" them at runtime | Hardcoding buries assumptions in code; learning from scratch invents facts. A sourced, versioned bundle is auditable, swappable, and honest about where knowledge came from. |
| **Reject unsourced rows (hard fail)** | Warn and continue on missing provenance | A warning gets ignored; a plausible unsourced number is the *worst* failure because it's invisible. Fail-fast makes "no simulated data" machine-checked. |
| **Three load paths (facts direct / sources ground / narratives synthesise)** | Push everything through one pipeline | A known number needs no LLM; fetched text should *ground* prose (anti-hallucination); narratives need the synthesis path. Matching the load path to the data type is what keeps it honest. |
| **Config in sourced CSVs** | Constants in agent code | Externalised, sourced params make every assumption visible and tunable without a redeploy. |

**The one to defend:** *load vs train, and sourced-or-reject.* For a system that must not fabricate, the discipline is **treat foundational knowledge as a versioned, provenance-checked artifact you load** — not weights you train or constants you bury. Every value names its origin, or it doesn't get in. That's the whole trust model in one rule.
