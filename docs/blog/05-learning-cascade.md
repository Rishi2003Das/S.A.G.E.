# Episode 5: The graph that learns its own weights

*Why one crisis lights up a whole dependency chain, and how SAGE refines those dependency weights bitemporally from live signals instead of leaving them as static config.*

The first four episodes were about getting a single entity's assessment right: reconcile signals before storing them, keep a computable graph beside a human one, compute the future early, and load a sourced worldview to start from. This one is about the thing that makes SAGE a *network* model rather than a set of isolated scores — because a crisis at the Strait of Hormuz is never really about Hormuz.

This is Episode 5 of 5 in the Engineering SAGE series. The full system overview is in the master post.

## An isolated score misses the actual crisis

Fusion assigns each entity a *primary* risk from its own direct signals: the corridor with dark-vessel gaps and hostile tone gets a high score. But the reason a Hormuz closure matters is everything *downstream* of it — the refineries it exposes, the ports it feeds, the suppliers that can only export through it. If those dependents keep their calm scores because no signal fired on them directly, the dashboard is telling a comforting lie during the exact moment it should be alarming.

The naive fix is to hard-code a dependency map and flat-decay risk along it. We did something more careful, because two things about that naive version are wrong: the decay should not be flat, and the map should not be static.

## Risk cascades along real, exposure-weighted edges

When an entity's score is high enough to matter, `cascade.py` walks its dependents breadth-first and raises their risk — a bounded propagation, not a global flood. Three rules keep it honest:

- **It follows real dependency edges, directionally.** Risk flows from the risky entity to whatever depends on it: `Corridor —EXPOSES→ Refinery`, `Port —FEEDS→ Refinery`, `Supplier —SUPPLIES→ Refinery`, and the reverse case `Supplier —EXPORTS_VIA→ Corridor` (the supplier depends on the corridor). These are the same typed edges the ARIO model traverses — the cascade is not a parallel invention.
- **The weight is exposure, not a flat decay.** Each hop multiplies by the edge's real `throughput_share_pct` from the sourced context bundle, so a port 45%-dependent on Hormuz inherits more than one that is 42%-dependent. Flat decay would have told them the same story. It only falls back to a decay constant when an edge has no learned share yet.
- **It only ever raises, and it is bounded.** Cascaded scores use max semantics — they never lower a signal-driven score — and propagation stops after two hops or when the inherited risk falls below a floor. A cascade should sharpen the picture, not manufacture panic three hops away.

Every cascaded score is written as a normal `RISK_STATE` edge tagged `cascade-v1`, with a rationale that names the source and the exact path it travelled (`"cascaded risk 0.41 from Strait of Hormuz (1 hop, exposure-weighted: Hormuz —FEEDS(0.45)→ Vadinar)"`). It traces back exactly like every other number in the system.

## The weights are learned, not frozen

Here is the decision the episode is really about. Those exposure weights start as sourced seed data in the context bundle — but a real supply chain reroutes, and a weight that was true last quarter can be wrong today. So the dependency graph is allowed to *learn*.

When a live System-1 signal implies a dependency has changed — "Vadinar cuts its Hormuz intake to about 25% after rerouting via the Cape" — the synthesis LLM detects that a share changed and extracts the new value. `edge_learning.py` then writes it **bitemporally**: the prior weight is kept, stamped with the time it was superseded, and the new weight is written with `tier="learned"` and the *source signal itself* as its provenance. The next cascade reads the updated weight and propagates risk accordingly.

The guardrail here is the same instinct that runs through the whole series: **the LLM reconciles evidence into a number, it never invents one.** The detector only fires when a signal genuinely describes a dependency change; absent that, the seed weight stands. There is no path by which the model can quietly drift a weight because it "felt" different.

```
seed (.context, sourced)  →  reconciled onto edges  →  cascade reads weights
        ▲                                                      │
        │  bitemporal update, cited to the signal              ▼
   LLM reconciles  ◄──  live System-1 signal implies a dependency shift
```

## Why this is a second brain, not a database

A database stores what you put in it. SAGE's dependency graph starts from sourced seed knowledge and then gets *more accurate the longer it runs*, because every genuine rerouting or capacity change is folded back into the weights that drive the cascade — and every version of every weight is traceable to either the `.context` bundle (seed) or a specific signal (learned). The risk propagation you see during the next crisis is shaped by the evidence accumulated since the last one.

That is the payoff of all the earlier discipline. Because identities are resolved before anything is written (Episode 1), because the graph is bitemporal and nothing is deleted (Episode 2), and because every seed value already carried a source (Episode 4), the graph can safely revise its own structure without ever losing the trail of why it believes what it believes.

## Takeaway

Isolated per-node scores describe symptoms; a cascade over real, exposure-weighted dependencies describes the crisis. Making those weights learnable — bitemporally, cited to the signal that changed them, and only ever reconciled from evidence — is what turns SAGE's knowledge base from a store into something that gets better at its job the longer it watches the world.

The code is open at [github.com/BlueWaves-afk/Sage](https://github.com/BlueWaves-afk/Sage).

*Engineering SAGE · Episode 5 of 5 — Previous: Episode 4. Back to the series overview.*

---

## 🎓 CS Fundamentals — study companion

*This finale is a **DSA** episode at heart — graph traversal, weighted propagation, BFS — fused with **DBMS** (bitemporal updates) and **ML** (online learning). Graph algorithms are interview staples; here's one in a real system.*

### Data Structures & Algorithms

- **Graph traversal — BFS.** The risk cascade is a **breadth-first search** from a high-risk node along dependency edges, raising each dependent's risk. BFS explores level by level (hop by hop) using a queue — which is exactly why the cascade is naturally hop-bounded. (DFS would dive deep first; BFS matches "propagate outward by degrees of separation.")
- **Weighted graphs & propagation.** Each hop multiplies risk by the edge's real `throughput_share_pct` — a **weighted** propagation, not a flat decay. A dependent 45%-exposed inherits more than one 42%-exposed. This is shortest-path/flow intuition: edge weights encode real influence.
- **Bounded traversal & convergence.** Propagation stops after a max hop count or when inherited risk falls below a floor, and uses **max-semantics** (only ever *raise* a dependent's score). Bounding + monotonic updates guarantee termination and prevent runaway feedback — the graph-algorithm version of "make it converge."
- **Cycle safety.** A dependency graph can have cycles; a naive propagation could loop forever. Visited-tracking + the decay floor + max-semantics keep it finite (same care as `visited` sets in graph traversal).

### DBMS + ML — a graph that learns

- **Online / incremental learning.** Edge weights start as sourced seed values, then get *refined from live signals* — a live signal that implies a dependency changed updates the weight. This is **online learning**: the model improves as data arrives, versus batch training on a fixed set.
- **Bitemporal weight updates (callback to Ep 2).** A learned weight is written bitemporally — old value kept with its supersede time, new value tagged `tier="learned"` with the *source signal* as provenance. So the graph's *structure* evolves with full history, not just its risk scores.
- **Reconcile, never invent.** The LLM only extracts a new weight when a signal genuinely describes a change; absent evidence, the seed stands. This is the guardrail that separates "a knowledge base that learns" from "a model that drifts."

**Interview Q&A.**
1. *BFS vs DFS — when do you use each?* → BFS for shortest-hops / level-order / "spread outward"; DFS for deep exploration, topological sort, cycle detection. Cascade = BFS.
2. *How do you propagate a value through a weighted graph without infinite loops?* → Visited set, per-hop decay, a stopping threshold, bounded hops, monotonic (max) updates → guaranteed termination.
3. *Batch vs online learning?* → Train once on a fixed dataset vs update incrementally as data streams; online adapts to drift but must guard against learning from noise.
4. *How do you evolve a graph's structure safely over time?* → Bitemporal edges (keep old, add new with provenance), reconcile only on real evidence, audit every change.

### System Design
- **Self-improving systems.** The cascade reads weights that get more accurate as evidence accumulates — a feedback loop where the system's model of the world sharpens with use. The design discipline is bounding that loop (only-raise, decay, evidence-gated) so "learns" never becomes "drifts."

### ⚖️ This vs That — the architecture decisions, and the roads not taken

| Decision | Alternatives | Why this choice |
|---|---|---|
| **Risk cascade (propagate to dependents)** | Isolated per-node scores | A crisis at a chokepoint should light up everything downstream; isolated scores show calm dependents during the exact moment they're most at risk — a comforting lie. |
| **Exposure-weighted propagation** | Flat per-hop decay | Flat decay treats a 45%-dependent and a 42%-dependent port the same. Real `throughput_share` weights encode actual influence. |
| **Bounded BFS, max-semantics** | Unbounded / additive propagation | Unbounded or additive propagation can loop or manufacture panic three hops away. Bounding + only-raise keeps it finite and honest. |
| **Learned weights (bitemporal, evidence-gated)** | Static config weights; freely LLM-adjusted weights | Static weights go stale as routes change; freely adjusted weights drift. Evidence-gated bitemporal updates evolve the graph *and* keep the full audit trail. |

**The one to defend:** *cascade + learned weights vs isolated static scores.* The naive model scores each entity alone. The systems-thinking answer: **risk is a network property — propagate it along real, weighted dependencies, and let those weights learn from evidence (bounded and bitemporal) so the model sharpens over time without drifting.** That's the difference between a database that stores scores and a knowledge base that reasons about a system.
