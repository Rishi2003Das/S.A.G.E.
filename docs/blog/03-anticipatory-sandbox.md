# Episode 3: Answering a crisis in 50ms by computing a future that hasn't happened

*How SAGE forks a speculative scenario when risk is merely rising, so the full model is already done the moment a threshold is crossed.*

When a crude corridor like the Strait of Hormuz is about to close, the useful question is not "what happened" but "what happens next, day by day, to India's refinery feedstocks and strategic reserves." Answering that means running a full disruption cascade, then a procurement solver, then a reserve optimiser. End to end that is several seconds. Several seconds is a long time when a threshold has just been crossed and everyone is watching the screen. This post is about how we made that answer appear almost instantly.

This is Episode 3 of 5 in the Engineering SAGE series. The full system overview is in the master post.

## The latency you cannot avoid by optimising

The disruption model is an Adaptive Regional Input-Output (ARIO) cascade over India's supply chain graph. A full cold run is around 2,500ms, and it is only the first stage: procurement routing and SPR optimisation run after it. Optimising the code helps at the margins, but the real problem is structural. You are doing all the work after the crossing is confirmed, while the user waits.

## Compute the future before it arrives

The insight is that a risk score does not jump from calm to crisis. It climbs. SAGE's score is a calibrated probability that the current situation is within 24 hours of a real disruption, and it moves through named bands as evidence accumulates: `calm → watch → elevated → action → critical`. By the time it crosses the **action** threshold — the model's Youden-J optimal point, currently **0.2634** — it has usually spent time in the **elevated** band first, rising. That rising window is free compute time.

So when a score enters the elevated band, SAGE forks a sandbox: a speculative branch that assumes the crossing will happen and runs the whole cascade ahead of time. The sandbox uses a GNN surrogate (a PyTorch GraphSAGE model trained to approximate the ARIO output) that returns in 150ms or less, and it pre-stages the result as a `PendingScenario`.

When the score actually crosses the action threshold, one of two things happens. If a sandbox already ran, SAGE promotes the pre-staged scenario: reload it, refresh the parameters against live numbers, done in about 50ms. If nothing was staged, it falls back to the cold ARIO run at around 2,500ms. In the common case where risk climbed gradually, promotion wins, and the end-to-end path from crossing to ranked output drops from roughly 8,500ms to about 300ms. That is the 28× speedup.

This is speculative execution, the same bet a CPU makes with branch prediction. You do the work early on the assumption a branch will be taken. If the risk recedes back below the band instead, you throw the sandbox away and you have lost nothing but some background compute.

## The rule that keeps speculation safe

Speculative execution is dangerous in exactly one way: the speculative result must never be mistaken for reality. In SAGE that danger is concrete. A `PendingScenario` carries a projected risk score for a future that has not happened. If that projected score were written onto the live entity node, the monitor would read it, believe the crisis had arrived, and fire on a future that may never occur.

So we made it an invariant: a projected risk score is never written as a `RISK_STATE` edge on a live node. Speculative output lives only on `PendingScenario` nodes and their linked output episodes, tagged `speculative`. The live graph keeps telling the truth about the present. Promotion is the only step that turns a speculative scenario into a confirmed one, and it happens only after the real crossing.

The second rule is about isolation in time, not just data. The sandbox runs parallel to the main synthesis path, never awaited inside it. If the speculative branch is slow or fails, the live pipeline that updates risk scores and wiki pages is completely unaffected. The anticipatory feature can never slow down the thing it is trying to accelerate.

## Why a surrogate instead of just running ARIO early

We could have run the full ARIO model in the sandbox instead of a GNN surrogate. We chose the surrogate because the sandbox may fire often during a volatile period, once per rising signal, and a 2,500ms model run each time is wasteful when the score might settle back down. The surrogate at 150ms makes speculation cheap enough to do liberally. The full-fidelity ARIO still runs, but on the confirmed path, where accuracy matters more than speed and the refresh step reconciles the surrogate's estimate against the real numbers.

## Takeaway

The speedup did not come from a faster model. It came from moving the work earlier, to the window when risk is merely rising through the elevated band, and treating the result as a speculative bet that is cheap to make and safe to discard. The one non-negotiable was keeping that speculation quarantined from the live graph, so a pre-computed future could never be confused for the present.

The code is open at [github.com/BlueWaves-afk/Sage](https://github.com/BlueWaves-afk/Sage).

*Engineering SAGE · Episode 3 of 5 — Previous: Episode 2. Next: Episode 4, from_pretrained, not fit.*

---

## 🧰 The anticipation stack, from zero — and what we chose it over

- **Speculative execution — over on-demand modelling.** The crisis answer takes too long to compute *after* the question. So we compute likely futures *ahead* of the threshold crossing — the source of the **28× speedup** from threshold-crossing to ranked output. Same idea as a CPU speculatively executing past a branch: do the expensive work before you're certain you'll need it, and have the answer ready.
- **Disruption modelling over the graph.** We simulate a disruption's propagation through the supplier/route graph and rank responses (reroute procurement, recommend an SPR drawdown).
- **A LangGraph conditional loop** drives model → evaluate → re-plan, so the sandbox re-runs as new evidence lands.

## 🚢 From demo to production

- **Bound the speculation** — you can't simulate every future; a compute budget picks the most probable branches.
- **Cache invalidation** — a materialised future goes stale when the world moves; it must expire.
- **A compute/spend budget** so anticipation doesn't become an always-on cost sink.

---

## 🎓 CS Fundamentals — study companion

*This episode is a beautiful cross-over: a **Computer Architecture** idea (speculative execution) applied at the **System Design** level, with **OS** (isolation, async) and **ML** (surrogate/distilled models) alongside.*

### Computer Architecture → System Design: speculative execution

- **Branch prediction & speculative execution.** A CPU doesn't wait to know which way a branch goes — it *predicts*, executes speculatively down that path, and commits the result if the prediction was right (or discards it if wrong). SAGE does exactly this at the system level: when risk is *rising* (the "branch" likely to be taken), it speculatively runs the full crisis pipeline and stages the result. If the threshold is crossed, it *promotes* the staged result (~50ms); if risk recedes, it discards the work. **You lost only cheap background compute — same bet a CPU makes billions of times a second.**
- **The commit/rollback discipline.** The non-negotiable rule: a speculative result must never be mistaken for reality. SAGE writes speculative output only to `PendingScenario` nodes tagged "speculative," never onto the live graph — exactly like a CPU not retiring speculative instructions until the branch resolves. Getting this wrong is the system analogue of a Spectre-class bug: speculative state leaking into the real world.
- **Caching / precomputation.** Speculative execution here is a form of **precompute + cache**: trade space and background cycles for latency at read time. Same family as materialised views, CDN prewarming, and prefetching.

### Operating Systems
- **Isolation & async execution.** The sandbox runs *parallel to* the live path and is never awaited inside it — so a slow/failed speculation can't stall the real-time pipeline. This is fault isolation + asynchronous decoupling: the optional feature can never degrade the mandatory one.

### Machine Learning
- **Surrogate / distilled models.** The sandbox uses a **GNN surrogate** (a GraphSAGE net trained to approximate the expensive ARIO simulation) that returns in ~150ms instead of ~2500ms. A surrogate (a.k.a. emulator / distilled model) learns the input→output mapping of a costly function so you can call it cheaply and often. The full-fidelity model still runs on the *confirmed* path where accuracy matters more than speed — a **fast-approximate / slow-exact** two-tier design.

**Interview Q&A.**
1. *Explain speculative execution and its risk.* → Execute a likely branch before it's confirmed; commit if right, discard if wrong; risk = leaking speculative state (correctness/security, e.g. Spectre).
2. *How would you make a slow pipeline feel instant when a trigger is predictable?* → Precompute speculatively during the lead-up, stage the result, promote on trigger; isolate and discard on miss.
3. *What is a surrogate/distilled model and when is it worth it?* → A cheap learned approximation of an expensive function; worth it when you call it often and can tolerate approximate answers (with an exact fallback).
4. *How do you keep an optional expensive feature from slowing the core path?* → Run it async and never await it inline; isolate failures; the core path proceeds regardless.

### ⚖️ This vs That — the architecture decisions, and the roads not taken

| Decision | Alternatives | Why this choice |
|---|---|---|
| **Speculative precompute during "rising"** | Compute everything on-demand at the crossing | On-demand makes the user wait seconds at the worst moment. The rising window is free compute; use it. |
| **GNN surrogate in the sandbox** | Run the full ARIO simulation speculatively | Full ARIO (~2500ms) is too expensive to fire on every rising signal that might fizzle. The 150ms surrogate makes speculation cheap enough to do liberally; exact ARIO runs on the confirmed path. |
| **Speculative state quarantined on PendingScenario** | Write projected scores onto the live graph | Writing a projected future onto live nodes would make the monitor fire on a crisis that hasn't happened. Quarantine is the correctness invariant. |
| **Sandbox async, never awaited** | Call the sandbox inline in the pipeline | Inline coupling lets a slow speculation stall the real-time path — the exact thing it's meant to accelerate. |

**The one to defend:** *speculative + surrogate vs on-demand exact.* The elegant answer isn't "optimise the model." It's **move the work earlier in time and approximate it, then reconcile against the exact model once reality confirms** — trading a little wasted compute and approximation error for a 28× latency win, with a hard isolation rule so the guess never contaminates the truth.
