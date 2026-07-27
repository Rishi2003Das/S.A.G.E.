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
