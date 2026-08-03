# Episode 1: Why raw signals never touch SAGE's vector store

*The synthesis-first ingest path, and why a deterministic source-aware triage gate (not an LLM) decides what gets reconciled, extracted, or dropped.*

SAGE watches four always-on feeds for signs that India's crude oil imports are about to be disrupted: vessel movements (AIS), news and GDELT, sanctions lists, and oil prices. The obvious way to build the memory for a system like this is to embed everything into a vector store and let retrieval sort it out later. We deliberately did not do that. This post is about why, and what we did instead.

This is Episode 1 of 5 in the Engineering SAGE series. The full system overview is in the master post.

## The problem with dumping raw signals into a vector store

Raw signals are noisy, repetitive, and frequently contradictory. AIS emits thousands of position pings an hour. News repeats the same event across twenty outlets. A price feed ticks every few seconds. If you embed all of that directly, three things go wrong.

First, your embeddings capture noise. The vector for "Strait of Hormuz" ends up shaped by a hundred near-identical Reuters rewrites and a stream of position pings, not by understanding.

Second, contradictions are never resolved. A news wire says traffic is normal while your own AIS detector shows four dark-vessel gaps. Dumped side by side into a vector store, both are just neighbours in embedding space. Nobody reconciles them.

Third, retrieval quality degrades exactly when you need it most, during a crisis, when signal volume spikes and the noise floor rises with it.

## Synthesis-first: reconcile before you store

Our rule is that raw signals never reach the vector store directly. The only path in is through synthesis.

When a signal arrives for an entity, Nova Pro loads that entity's current intelligence page and reconciles the new signal against it. If the news says "normal" but AIS says "four dark gaps," the synthesis step writes the contradiction explicitly and explains why the two sources measure different things. Only that reconciled understanding is embedded and written as an episode.

The vector store therefore holds contradiction-resolved episodes, not raw facts. The embedding for an entity reflects the current reconciled assessment, which is exactly what you want retrieval to surface.

## The triage gate is deterministic code, not an LLM

Before anything is synthesized, every signal passes a triage gate that decides one of four outcomes: synthesize, extract, store, or drop. The important decision here was to make triage plain code rather than an LLM call.

The routing is source-aware. AIS and price signals are numeric by nature, so they always route to extract: they update the risk factors on the graph and never trigger prose synthesis. Sanctions always route to synthesize, because a sanctions change always has implications worth writing down. News routes on cosine similarity to the current assessment, so a genuinely new development gets synthesized and a twentieth rewrite of the same story does not.

We considered letting an LLM classify each signal. We rejected it for three reasons. It costs a model call on every signal, including the thousands that should be dropped. It is non-deterministic, so the same signal could route differently on two runs. And it is unnecessary, because the source already tells you almost everything: an AIS anomaly is never going to warrant a paragraph of narrative, and no model needs to be asked whether it should.

This is also why the routing lives in one place. A single AIS anomaly that happens to score high cosine similarity should still never trigger Nova Pro to write prose about one detection. Source-aware code guarantees that. A similarity threshold alone would not.

## Resolve identity before any model runs

There is one more step before synthesis, and it is also deterministic. Every signal names its entities, and those names are resolved to a canonical registry before an LLM is ever invoked. An AIS anomaly resolves by H3 cell, a price changepoint by instrument, a sanctions record by its literal name, all through fast lookup tables.

This matters because the alternative is silent duplication. If one sub-agent writes "Hormuz" and another writes "Strait of Hormuz," you get two graph nodes and two wiki pages for the same place. Resolving to a canonical id first means every signal about the strait updates the same entity, and the synthesized text always opens with the canonical name so the graph extractor anchors to the existing node rather than inventing a new one.

## The score itself is a calibrated model, not a heuristic

Triage decides *what gets written*. A separate question is *how the risk score is computed* from the signals that survive — and here too we refused to hand-wave. The four feeds are fused into a 17-dimensional feature vector (AIS gap counts and durations, GDELT tone and its delta, Brent change and regime flags, sanctions events), and the score is the output of a **gradient-boosted classifier with Platt scaling**, trained to predict whether the current feature vector is within 24 hours of a real threshold-crossing disruption.

The honest part is the validation. We evaluate it **leave-one-crisis-out**: train on four historical crises, test on the fifth it has never seen. Across the 2019 Gulf of Oman attacks, the 2021 Suez blockage, the 2022 Ukraine energy shock, the 2025 US–Iran Hormuz standoff, and the 2026 Hormuz closure, the mean out-of-sample **LOCO AUC is 0.84**, and the Youden-J action threshold sits at a calibrated **0.2634**. Those are genuine out-of-sample numbers, not training-set fit — and when the trained model file is absent, the system falls back to a clearly-labelled weighted-sum, never a silent fabrication.

The feature importances are worth a glance because they sanity-check the model: GDELT tone (mean and delta) and AIS gap duration dominate; sanctions and price war-risk premium contribute almost nothing on this data. That matches intuition — hostile diplomatic tone and vessels going dark lead a corridor closure; the price shock is a lagging confirmation, not a leading indicator. A model whose learned weights disagreed with that would be a model to distrust.

## Takeaway

The discipline that made the rest of SAGE work was refusing to let raw signals into the store. A deterministic, source-aware gate decides what deserves synthesis, an LLM reconciles the survivors against what we already believed, only that reconciled understanding gets embedded, and the risk score on top is a model we can defend with out-of-sample numbers rather than a dial we tuned by feel. The vector store ends up holding judgement, not noise.

The code is open at [github.com/BlueWaves-afk/Sage](https://github.com/BlueWaves-afk/Sage).

*Engineering SAGE · Episode 1 of 5 — Next: Episode 2, Why SAGE keeps two knowledge graphs.*

---

## 🎓 CS Fundamentals — study companion

*This post is rich in **DBMS** (vector stores, dedup), **Machine Learning / Statistics** (calibration, GBM, out-of-sample validation), and **System Design** (deterministic routing vs an LLM). Great prep for data-eng and ML-systems interviews.*

### DBMS / Information Retrieval

- **Vector stores & embeddings.** An embedding maps text to a point in high-dimensional space where *semantic similarity ≈ geometric closeness* (cosine similarity). A vector store indexes these for approximate nearest-neighbour search (ANN, e.g. HNSW). SAGE's rule — *only synthesised, contradiction-resolved episodes get embedded* — is a **data-quality-at-write-time** decision: garbage in the index means garbage retrieval, so it curates before indexing.
- **Deduplication & canonicalization.** Resolving "Hormuz" and "Strait of Hormuz" to one canonical entity *before* writing prevents duplicate nodes — the same problem as **entity resolution / record linkage** in databases. Do it with deterministic lookup tables (alias → id), not an LLM.
- **Cosine similarity.** `cos(θ) = (A·B)/(‖A‖‖B‖)` — the workhorse similarity metric for embeddings; SAGE routes news on a cosine threshold against the current assessment.

### Machine Learning & Statistics

- **The fusion model: GBM + Platt scaling.** A **Gradient-Boosted** classifier (ensemble of shallow trees, each correcting the last's residuals) predicts "within 24h of a disruption." **Platt scaling** calibrates raw scores into real probabilities (fits a logistic on the outputs) so a "0.7" means roughly 70%.
- **Honest evaluation — LOCO / out-of-sample.** *Leave-One-Crisis-Out*: train on 4 crises, test on the 5th unseen one; the reported **AUC 0.84** is genuine generalisation, not training-set fit. **AUC-ROC** = probability the model ranks a random positive above a random negative (0.5 = coin flip, 1.0 = perfect).
- **Youden's J threshold.** The operating point that maximises `sensitivity + specificity − 1` — how SAGE picks its action threshold (0.2634) rather than a hand-waved 0.5.
- **Feature importance as a sanity check.** GDELT tone + AIS gaps dominating (and price war-premium ≈ 0) matches domain intuition — a model whose importances disagreed with reality would be one to distrust.

**Interview Q&A.**
1. *What is an embedding and how do you search a vector store?* → Map to a semantic vector space; ANN search (HNSW) by cosine/dot similarity.
2. *Precision/recall, ROC, AUC — define them.* → TP/(TP+FP), TP/(TP+FN); ROC plots TPR vs FPR across thresholds; AUC = ranking quality.
3. *Why validate leave-one-crisis-out instead of a random split?* → Random splits leak correlated samples from the same event; LOCO tests true generalisation to an unseen regime.
4. *What is model calibration and why does it matter?* → Making predicted probabilities match observed frequencies (Platt/isotonic); essential when a downstream system acts on the probability.
5. *How do you choose a classification threshold?* → By the operating goal — Youden's J, or the cost-weighted point on the ROC/PR curve, not a default 0.5.

### System Design
- **Deterministic routing vs an LLM classifier.** SAGE routes signals with plain code because it's *cheaper* (no model call on events that will be dropped), *deterministic* (same input → same route), and *unnecessary* (the source already tells you). Push each decision to the cheapest layer that can make it correctly — a recurring senior-design principle.

### ⚖️ This vs That — the architecture decisions, and the roads not taken

| Decision | Alternatives | Why this choice |
|---|---|---|
| **Synthesis-first (reconcile, then embed)** | Embed raw signals directly, sort it out at retrieval | Embedding raw feeds captures noise, never resolves contradictions, and degrades exactly during a crisis when volume spikes. Storing *judgement* not *noise* keeps retrieval sharp. |
| **Deterministic triage gate** | An LLM classifier deciding synthesize/extract/drop | An LLM costs a call on every event (including the 1000s to drop), is non-deterministic, and answers a question the *source* already answers. Code is cheaper, reproducible, and sufficient. |
| **Calibrated GBM for the score** | Hand-tuned weighted-sum of features | A weighted sum is a guess; a GBM + Platt is validated out-of-sample (LOCO AUC 0.84) and calibrated — you can *defend* the number, and it degrades to the weighted-sum only as a labelled fallback. |
| **Resolve identity before the LLM** | Let the LLM extract entities | Pre-resolving to a canonical registry prevents duplicate graph nodes and anchors synthesis to existing entities. |

**The one to defend:** *deterministic code vs an LLM for routing.* The trendy answer is "use an LLM for everything." The mature answer: **an LLM is the most expensive, least predictable tool — use it only where the source genuinely can't answer.** Routing by source type is a solved problem in code; spending a model call (and non-determinism) on it is a cost and reliability regression.
