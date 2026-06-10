# episodic-graph-lab — contract draft v0

Three contracts for the ablation harness. Grounded in the *real* shapes:
- Nemori `Episode` / `SemanticMemory` / `Message` — `nemori/domain/models.py`
- Graphiti `add_episode(...)` — `graphiti_core/graphiti.py` (EpisodeType enum, reference_time, entity/edge config)
- REMem episodic-gist — atomic fact + `[date, time]` prefix, resolved absolute dates

Design rule: the three contracts are the **lowest common denominator that still carries provenance and time**. Every producer must be able to fill them; richer fields go in `extra` (lossless escape hatch) so we never silently drop the signal we are trying to measure.

---

## 1. RawRecord — pipeline input

```python
@dataclass
class RawRecord:
    record_id: str                 # stable, content-hash or dataset-assigned; the root of provenance
    source: str                    # dataset/file name, e.g. "locomo/conv-42.jsonl"
    text: str                      # the raw passage / message text
    timestamp: datetime | None     # event time if known (REMem needs this to resolve "last Monday")
    speaker: str | None            # role/author if dialogue ("user"/"assistant"/person name)
    extra: dict[str, Any]          # anything dataset-specific, never interpreted by the harness
```

Notes:
- `timestamp` is nullable but **strongly preferred** — contradiction/update eval is meaningless without ordered events, and REMem's temporal resolution degrades to no-op without it.
- One `.txt` line or one `.jsonl` object → one `RawRecord`. Keep it dumb.

---

## 2. Episode — the contested middle layer

This is where the experiment lives, so it must be fillable by *all three* configs, including the one that does no episode formation at all.

```python
@dataclass
class Episode:
    episode_id: str
    source_record_ids: list[str]   # PROVENANCE: which RawRecords formed this episode
    title: str | None              # Nemori fills; passthrough leaves None
    content: str                   # the episode body / gist / raw text
    occurred_at: datetime | None   # event time (Nemori created_at, or RawRecord.timestamp)
    formed_by: str                 # "nemori" | "remem-gist" | "passthrough"
    facts: list[Fact]              # extracted atomic facts, may be empty
    extra: dict[str, Any]          # lifecycle/provenance the harness shouldn't flatten
```

```python
@dataclass
class Fact:
    fact_id: str
    content: str                   # atomic statement, "John works at Acme"
    fact_type: str | None          # Nemori memory_type: identity/preference/relationship/...
    occurred_at: datetime | None   # REMem's resolved absolute date lives here
    source_episode_id: str         # PROVENANCE: back-link (mirrors SemanticMemory.source_episode_id)
    source_record_ids: list[str]   # PROVENANCE: all the way back to raw
    extra: dict[str, Any]
```

### How each producer fills it (the part Codex hand-waved)

| Config | `formed_by` | how Episode is built | how Fact is built |
|---|---|---|---|
| **1. raw → Graphiti** | `passthrough` | 1 RawRecord → 1 Episode, `content = text`, `title = None`, `facts = []` | none (Graphiti extracts entities itself) |
| **2. raw → Nemori → Graphiti** | `nemori` | Nemori `Episode`: `content`←content, `title`←title, `occurred_at`←`created_at`, `source_record_ids`←map from `source_messages`, lifecycle (`boundary_reason`, `merged_from`, `merge_timestamp`) → `extra` | Nemori `SemanticMemory` → `Fact`: `content`, `fact_type`←`memory_type`, `source_episode_id`←`source_episode_id` |
| **3. raw → REMem → Nemori/Graphiti** | `remem-gist` | 1 gist → 1 Episode, `content`←gist text (timestamp prefix stripped), `occurred_at`←parsed absolute date | gist *is* the fact: 1 Episode → 1 `Fact`, `occurred_at`←resolved date |

**Config 1's passthrough Episode is the critical fairness control.** Without it, config 1 has no Episode contract and you'd be comparing different interfaces, not different pipelines. Define it explicitly: identity transform, `facts=[]`, `formed_by="passthrough"`.

**Lossy-mapping warning to log, not hide:** Nemori's `merged_from` / `boundary_reason` and REMem's gist-completeness dimensions have no first-class field here. They go in `extra`. If your eval ever wants to *measure* merge quality, promote `merged_from` to a real field — don't let it die in a dict.

---

## 3. GraphEvent — what gets written to Neo4j

One normalized instruction stream, regardless of who produced it, so the Neo4j writer and the metrics read one shape.

```python
@dataclass
class GraphEvent:
    event_id: str
    kind: str                      # "node" | "edge"
    label: str                     # node label or edge type ("Person", "WORKS_AT")
    key: str                       # natural key for dedup ("person:john")
    valid_at: datetime | None      # event/temporal-validity time
    src_key: str | None            # edge only: source node key
    dst_key: str | None            # edge only: target node key
    provenance: Provenance         # MANDATORY — the whole point of the harness
    extra: dict[str, Any]
```

```python
@dataclass
class Provenance:
    record_ids: list[str]          # raw records that justify this event
    episode_ids: list[str]         # episodes it passed through ([] for config 1)
    fact_ids: list[str]            # facts it came from ([] when graph store did extraction)
    produced_by: str               # "graphiti" | "nemori-direct-writer" | ...
```

Mapping from Graphiti: `add_episode` returns `AddEpisodeResults{nodes, edges}` → emit one `GraphEvent` per node/edge. `reference_time` → `valid_at`. `source_description` → carry the `Provenance` so we can trace a Neo4j node back to a RawRecord. `group_id` → `extra` (partitioning).

---

## Why this shape

1. **Provenance is mandatory and unbroken** — `RawRecord.record_id` survives into `GraphEvent.provenance.record_ids` through every config. That single chain is what makes "source traceability" a *measurable number* (fraction of Neo4j nodes that resolve to a raw record) instead of a vibe.
2. **Time is first-class at every layer** (`timestamp` → `occurred_at`/`valid_at`) so contradiction/update behavior is testable.
3. **`extra` everywhere** means no producer is forced to discard signal to fit — but anything we want to *score* must be promoted out of `extra` to a real field. Treat `extra` as a staging area, not a graveyard.
4. **Config 1 is a first-class citizen**, not an afterthought, because the null-episode is explicit.

## Open decisions for Codex (real forks, not bikeshedding)

- **Fact identity / dedup key.** Content hash? Embedding cluster? This decision *is* the "duplicate rate" metric — it can't be left implicit.
- **GraphEvent `key` ownership.** Does the harness assign natural keys, or does each graph backend? If Graphiti owns keys, configs aren't comparable on dedup. Lean: harness owns `key`.
- **Where contradiction resolution happens.** In Fact extraction, in GraphEvent merge, or in Neo4j write? Whichever it is, it must be the *same stage* across configs or the comparison is rigged.
- **Promote `merged_from` to a field?** Only if merge quality is a scored metric. Decide before writing the mapper.

## Milestone 0 (before any pipeline code)
Freeze these three dataclasses + a 30–50 record gold dataset with **labeled** contradictions and recall queries. The contracts and the labels are the project; the pipelines are plumbing.
