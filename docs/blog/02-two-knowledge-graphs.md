# Episode 2: Why SAGE keeps two knowledge graphs, not one

*A computable Graphiti graph for machines and an editable wikilink graph for humans, plus why every edge is bitemporal and nothing is deleted.*

SAGE already has a knowledge graph. Every entity, every typed relationship, every risk score lives in a Graphiti graph backed by FalkorDB. So when we added a second graph, a set of Obsidian-style Markdown pages with `[[wikilinks]]`, the obvious question was whether we were just duplicating the first one. We were not, and the reason is worth explaining.

This is Episode 2 of 5 in the Engineering SAGE series. The full system overview is in the master post.

## Two graphs that hold different things

The Graphiti graph is for machines. It holds typed, computable relationships with numeric attributes: `RISK_STATE {score: 0.67}`, `EXPORTS_VIA {daily_export_mbpd: ...}`, `FEEDS`, `SUPPLIES`. The monitor reads it, the ARIO disruption model traverses it, the procurement ranker computes over it. Its edge types are a fixed, enumerated ontology.

The wikilink graph is for humans. It holds the narrative, associative context that never becomes a formal typed edge: a historical analogy ("pattern echoes the `[[2019 Tanker Attacks]]`"), a qualitative exposure note ("crude bound for `[[Jamnagar Refinery]]` is most exposed"), a pointer to the leading indicator in a contradiction. It is editable Markdown, so an analyst can correct a page by hand.

## Why not force everything into typed edges

The temptation is to collapse the two, to invent edge types like `HISTORICAL_ANALOGY` or `MOST_EXPOSED` and keep a single graph. We tried thinking this through and it falls apart quickly.

The interpretive layer is open-ended. An analyst comparing today to a 2019 precedent, noting that one source lags another, or flagging which refinery to watch, is not producing a relationship from a fixed vocabulary. It is producing prose reasoning. If you force that into typed edges, you either explode the ontology into hundreds of half-used edge types, or you flatten rich reasoning into a label that loses the "why."

So we let the graph compute and the wiki explain. Graphiti holds what needs to be traversed and calculated. The wiki holds the connective tissue of analyst reasoning that makes an assessment navigable. Having both is the architecture's strength, not its redundancy.

## The wiki is a real second brain

Each entity page is a git-versioned Markdown file with YAML frontmatter (risk score, factors, coordinates, `source_episodes`) and `[[wikilinks]]` throughout the prose, mirrored into a `links_out` list. That format buys two things for free.

It opens natively in Obsidian. Point Obsidian at the wiki folder and its graph view renders every page as a node and every `[[link]]` as an edge, sized by how many pages link to it. Hormuz naturally becomes the most-linked node. We got a working knowledge-graph visualization with zero build effort, and it doubles as a debugging tool: a page with no outgoing links means synthesis failed to find related entities.

The same `links_out` data drives the product view. The geospatial renderer reads the frontmatter, places each entity at its coordinates, and draws arcs between linked entities on a deck.gl globe. One dataset, two renderings, no extra structures.

## Why bitemporal, and why we never delete

The Graphiti side carries one more decision that is easy to skip and expensive to retrofit: every edge is bitemporal. It records `observed_at` (when the event was true in the world) and `ingested_at` (when SAGE recorded it). A fact is current when `invalid_at IS NULL`. When a value changes, the old edge is invalidated, never deleted.

This matters because intelligence is a moving target and you are constantly asked "what did we believe, and when." A risk score of 0.67 at 14:00 that later rose to 0.82 is not a correction to be overwritten, it is history. Separating world-time from record-time also keeps late-arriving signals honest: a report that arrives now but describes something from three hours ago is filed at its true `observed_at`, not stamped with the moment we happened to read it.

Bitemporality is also what lets the *structure* of the graph evolve safely, not just the risk scores. When a live signal implies a dependency itself has changed — a refinery rerouting away from a corridor, say — SAGE can revise the edge weight while keeping the old value with the time it was superseded. That mechanism, and the risk cascade that reads those weights, is its own decision; it is the subject of Episode 5.

## Takeaway

One graph could not have done both jobs. A computable graph answers "what is the supply gap"; a narrative graph answers "why does this echo 2019, and which refinery should we watch." SAGE keeps both, aligned by a shared entity registry, and lets the bitemporal history make every past belief auditable rather than overwritten.

The code is open at [github.com/BlueWaves-afk/Sage](https://github.com/BlueWaves-afk/Sage).

*Engineering SAGE · Episode 2 of 5 — Previous: Episode 1. Next: Episode 3, Answering a crisis in 50ms.*

---

## 🎓 CS Fundamentals — study companion

*This is the **DBMS** episode — graph databases, temporal data modelling, and the append-only/never-delete principle — with **DSA** (graphs) alongside. Bitemporal modelling is a genuinely advanced topic that impresses interviewers.*

### DBMS — data modelling

- **Graph vs relational modelling.** A graph database stores entities as nodes and relationships as first-class, typed edges with properties. Queries traverse relationships directly (no repeated JOINs), which is why SAGE's cascade and ARIO models — inherently relationship-walks — use a graph. Rule of thumb: **relationships-as-data → graph DB; tabular set operations → relational.**
- **Polyglot persistence.** Two graphs on purpose: a *computable* typed graph (for machines to traverse/calculate) and a *human wiki* graph (editable prose with wikilinks). Different access patterns → different stores, aligned by a shared entity registry. This is the same instinct as caching, search indexes, and read-models in CQRS.
- **Bitemporal data — the star concept.** Every edge records **two** time axes: `observed_at` (valid time — when it was true in the world) and `ingested_at` (transaction time — when the DB learned it). A fact is current when `invalid_at IS NULL`. This lets you answer *"what did we believe, and when did we believe it?"* — impossible if you overwrite. Late-arriving data is filed at its true valid-time, not the moment of insert.
- **Append-only / never delete.** Updates *invalidate* the old edge and insert a new one; history is preserved. This is **event-sourcing / temporal tables / MVCC**-style thinking: the log of changes *is* the source of truth, and any past state is reconstructable and auditable.
- **Why "typed edges vs prose."** Forcing open-ended analyst reasoning into a fixed edge ontology either explodes the schema or flattens meaning — the classic **rigid-schema vs schema-flexible** tension. SAGE splits it: strict typed edges for computation, free Markdown for interpretation.

**Interview Q&A.**
1. *When would you use a graph database over a relational one?* → When relationships are the primary query (multi-hop traversals, variable-depth paths); relational when you need set-based aggregation over uniform rows.
2. *Explain bitemporal data (valid time vs transaction time).* → Valid time = when true in reality; transaction time = when recorded. Two axes answer "what was true" and "what we knew, when."
3. *Why keep history instead of updating in place?* → Auditability, reproducibility, time-travel queries, correctness for late-arriving data; the update-in-place loses the "as-of" answer.
4. *What is polyglot persistence / CQRS-style read models?* → Use multiple stores each optimised for one access pattern, kept consistent from a single source of truth.

### DSA
- **Graphs 101.** Nodes + typed, weighted, directed edges; adjacency for traversal. The wiki's `[[wikilinks]]` form an implicit graph whose most-linked node (Hormuz) is its highest-degree vertex — degree centrality, for free, and a debugging signal (a node with no out-links = failed synthesis).

### ⚖️ This vs That — the architecture decisions, and the roads not taken

| Decision | Alternatives | Why this choice |
|---|---|---|
| **Two graphs (computable + human)** | One graph for everything | One structure can't be both a strict computable ontology *and* an open-ended prose canvas — you'd explode the edge types or lose the "why." Two aligned graphs let each do one job well. |
| **Typed edges for machines, Markdown for humans** | Invent edge types like `HISTORICAL_ANALOGY` | Open-ended reasoning isn't a fixed vocabulary; forcing it into edges bloats the schema and flattens meaning. Prose holds the interpretive layer. |
| **Bitemporal, never-delete** | Overwrite values in place | Overwriting destroys the "what did we believe at 14:00" answer and mishandles late data. Bitemporal costs storage but buys auditability and correctness. |
| **Graph DB (FalkorDB/Graphiti)** | Relational tables with join tables | The core queries are relationship walks (cascade, dependency traversal); a graph engine makes those natural and fast. |

**The one to defend:** *bitemporal vs overwrite.* Most candidates model "current state." The senior move is to recognise that **in any system asked to explain past decisions, history is data** — separate valid-time from transaction-time, invalidate instead of delete, and every past belief becomes auditable. It's the difference between a database that *stores* and one that *remembers*.
