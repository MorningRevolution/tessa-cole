# TESSA & COLE - Part 3 of 6
## Knowledge & Memory System (7-Phase Production Architecture)

---

## Overview

The TESSA & COLE knowledge system evolved through 7 phases of production hardening. It started as SQLite FTS5 keyword search and grew into a hybrid pipeline: **FAISS semantic vectors + BM25 keyword matching + Knowledge Graph relationship traversal**, merged via Reciprocal Rank Fusion (RRF), with temporal decay, importance tiers, confidence labels, and graph boosting.

This is not theoretical. Every component described here runs in production on a 4GB RAM DigitalOcean droplet, handling 9 agents, 300+ vectors, 120+ BM25 documents, 36 graph nodes, and 35 relationship edges. Total index size: ~4MB. Cost: ~$0.03/week for embeddings.

```
Document -> R2 -> graph_builder (auto) -> knowledge/graph/
                                     |
Contact change -> FAISS watcher -> graph_builder (auto)
                                     |
recall.py: FAISS + BM25 (RRF) + Graph Boost (1.3x) + Temporal Decay + Importance Tiers
                                     |
Morning Report: 13 sections (calendar, CT, finance, health, CRM, relationships,
                             social, yesterday, reminders, trends, decisions,
                             connections, bookmarks)
                                     |
JKE: corrections/confirmations -> fast_rules.jsonl (instant) -> weekly synthesis (GPT-4o-mini)
                                     |
Conflict resolver: recall + graph entity match -> GPT-4o-mini arbitration
```

---

## Phase 1: FAISS Vector Search + Metadata Enrichment

### memory_sync.py, The Indexer

Watches all workspace directories for file changes. On create/modify, chunks the file, generates embeddings via text-embedding-3-small, and stores in FAISS with rich metadata.

```python
import faiss
import numpy as np
import json
import os
import re
import hashlib
from datetime import datetime
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

DIMENSIONS = 1536  # text-embedding-3-small
MAX_CHARS = 20000  # ~5000-6000 tokens per chunk
OVERLAP_CHARS = 2000  # Sliding window overlap, prevents context loss at chunk boundaries

# Watched directories (production: 9 workspaces + shared knowledge)
WATCH_DIRS = [
    "/path/to/workspace/memory",
    "/path/to/workspace-agent2/memory",
    "/path/to/workspace-ct/memory",
    "/path/to/workspace-doc/memory",
    "/path/to/workspace-agent3/memory",
    "/path/to/workspace-agent4/memory",
    "/path/to/workspace-r2/memory",
    "/path/to/knowledge",
]


def chunk_text(text: str, max_chars: int = MAX_CHARS,
               overlap_chars: int = OVERLAP_CHARS) -> list[str]:
    """Split text into overlapping chunks.

    The overlap prevents losing context at chunk boundaries.
    Production-learned: without overlap, a person mentioned at the end
    of chunk N and the start of chunk N+1 becomes two partial matches
    instead of one strong match.
    """
    if len(text) <= max_chars:
        return [text]

    chunks = []
    start = 0
    while start < len(text):
        end = start + max_chars
        chunks.append(text[start:end])
        start = end - overlap_chars  # Slide back by overlap amount

    return chunks


def extract_metadata_from_path(filepath: str) -> dict:
    """Extract workspace, topic, and date from file path.

    Examples:
      workspace/memory/contacts/sarah-chen.md -> workspace="workspace", topic="contacts"
      workspace-ct/memory/acme-corp/meetings/2026-03-15.md -> workspace="workspace-ct", topic="acme-corp"
    """
    parts = filepath.split("/")
    workspace = None
    topic = None
    date = None

    for i, part in enumerate(parts):
        if part.startswith("workspace"):
            workspace = part
        if part == "memory" and i + 1 < len(parts):
            topic = parts[i + 1]

    # Extract date from filename: YYYY-MM-DD
    basename = os.path.basename(filepath)
    date_match = re.search(r'(\d{4}-\d{2}-\d{2})', basename)
    if date_match:
        date = date_match.group(1)

    return {"workspace": workspace, "topic": topic, "date": date}


def calculate_importance(filepath: str, content: str) -> float:
    """Score importance 0.0-1.0 based on path and content.

    Higher importance = slower temporal decay (see recall.py).
    Production tuning: journal and contacts decay at 0.99/month,
    logs decay at 0.85/month.
    """
    score = 0.3  # Base importance

    # Path-based scoring
    if "/journal/" in filepath:   score += 0.2
    if "/contacts/" in filepath:  score += 0.2
    if "/distilled/" in filepath: score += 0.1

    # Content-based scoring
    word_count = len(content.split())
    if word_count > 500:  score += 0.05
    if word_count > 1000: score += 0.1

    # Keyword scoring
    keywords = ["decision", "strategy", "agreement", "contract", "policy",
                "assessment", "profile", "health", "insurance"]
    if any(kw in content.lower() for kw in keywords):
        score += 0.1

    return min(score, 1.0)


def embed_and_store(filepath: str) -> None:
    """Embed a file and store in FAISS index with metadata."""
    with open(filepath) as f:
        content = f.read()

    if len(content.strip()) < 50:
        return  # Skip near-empty files

    # Remove old entries for this file (handles updates)
    remove_old_entries(filepath)

    metadata = extract_metadata_from_path(filepath)
    importance = calculate_importance(filepath, content)
    chunks = chunk_text(content)

    for i, chunk in enumerate(chunks):
        embedding = get_embedding(chunk)  # text-embedding-3-small
        vector = np.array([embedding], dtype="float32")

        # Add to FAISS
        idx = faiss_index.ntotal
        faiss_index.add(vector)

        # Store metadata
        metadata_store[str(idx)] = {
            "file": filepath,
            "content": chunk[:200],  # Preview only
            "chunk": i,
            "workspace": metadata["workspace"],
            "topic": metadata["topic"],
            "date": metadata["date"] or datetime.now().strftime("%Y-%m-%d"),
            "importance": importance
        }

    save_index()
    save_metadata()


class MemoryWatcher(FileSystemEventHandler):
    """Watch for file changes and re-index.

    Production integration: also triggers graph_builder.py when
    contact files change.
    """
    def on_modified(self, event):
        if event.src_path.endswith(".md"):
            embed_and_store(event.src_path)
            # Auto-update knowledge graph for contacts
            if "/contacts/" in event.src_path:
                os.system(f"python3 graph_builder.py --from-contact {event.src_path}")

    def on_created(self, event):
        if event.src_path.endswith(".md"):
            embed_and_store(event.src_path)
            if "/contacts/" in event.src_path:
                os.system(f"python3 graph_builder.py --from-contact {event.src_path}")

    def on_deleted(self, event):
        if event.src_path.endswith(".md"):
            remove_old_entries(event.src_path)
```

---

## Phase 2: BM25 Hybrid Search with Reciprocal Rank Fusion

FAISS alone misses exact keyword matches (invoice numbers, specific names). BM25 alone misses semantic similarity. RRF merges both result sets into a unified ranking.

### BM25 Corpus (Built Alongside FAISS)

```python
# Built by memory_sync.py during indexing
# Stored as bm25_corpus.json (~1MB for 120+ documents)

def build_bm25_corpus(metadata: dict) -> dict:
    """Build BM25 corpus from FAISS metadata.

    Groups chunks by file, tokenizes content for keyword matching.
    """
    corpus = {}
    for meta_id, meta in metadata.items():
        filepath = meta["file"]
        if filepath not in corpus:
            with open(filepath) as f:
                full_text = f.read()
            corpus[filepath] = {
                "tokens": tokenize(full_text),
                "chunks": []
            }
        corpus[filepath]["chunks"].append({
            "meta_id": meta_id,
            "token_count": len(meta.get("content", "").split())
        })
    return corpus


def tokenize(text: str) -> list[str]:
    """Simple tokenizer for BM25."""
    tokens = re.findall(r'[a-z0-9]+(?:\'[a-z]+)?', text.lower())
    return [t for t in tokens if len(t) > 1]  # Skip single chars
```

### Reciprocal Rank Fusion (RRF)

```python
RRF_K = 60  # Standard RRF constant

def reciprocal_rank_fusion(faiss_results: list[dict],
                           bm25_results: dict[str, int],
                           metadata: dict) -> list[dict]:
    """Merge FAISS semantic + BM25 keyword results via RRF.

    Each result gets: 1/(K + rank_faiss) + 1/(K + rank_bm25)
    Normalized to 0-1 range.

    Production data: hybrid search finds 20-30% more relevant results
    than FAISS alone, especially for exact names and dates.
    """
    combined = {}

    # FAISS rankings
    for rank, result in enumerate(faiss_results):
        filepath = result["file"]
        combined[filepath] = combined.get(filepath, {"faiss_rank": None, "bm25_rank": None})
        if combined[filepath]["faiss_rank"] is None:
            combined[filepath]["faiss_rank"] = rank + 1
        combined[filepath]["meta"] = result

    # BM25 rankings (sorted by score descending)
    bm25_sorted = sorted(bm25_results.items(), key=lambda x: x[1], reverse=True)
    for rank, (filepath, score) in enumerate(bm25_sorted):
        if filepath not in combined:
            # BM25-only result, look up metadata
            meta = find_metadata_by_file(filepath, metadata)
            if meta:
                combined[filepath] = {"faiss_rank": None, "bm25_rank": rank + 1, "meta": meta}
        else:
            combined[filepath]["bm25_rank"] = rank + 1

    # Calculate RRF scores
    max_rrf = 2.0 / (RRF_K + 1)  # Theoretical maximum
    results = []
    for filepath, data in combined.items():
        faiss_contrib = 1.0 / (RRF_K + data["faiss_rank"]) if data["faiss_rank"] else 0
        bm25_contrib = 1.0 / (RRF_K + data["bm25_rank"]) if data["bm25_rank"] else 0
        rrf_score = (faiss_contrib + bm25_contrib) / max_rrf  # Normalized 0-1

        results.append({
            **data["meta"],
            "rrf_score": rrf_score,
            "faiss_rank": data["faiss_rank"],
            "bm25_rank": data["bm25_rank"],
        })

    return sorted(results, key=lambda x: x["rrf_score"], reverse=True)
```

### BM25 Scoring Implementation

```python
import math

def bm25_search(query_tokens: list[str], corpus: dict,
                workspace: str | None = None,
                topic: str | None = None,
                top_k: int = 20) -> dict[str, float]:
    """BM25 keyword search with Okapi scoring.

    Parameters tuned for markdown document collections:
    k1=1.5 (term frequency saturation)
    b=0.75 (document length normalization)
    """
    k1, b = 1.5, 0.75
    N = len(corpus)  # Total documents
    avg_dl = sum(len(doc["tokens"]) for doc in corpus.values()) / max(N, 1)

    # Calculate IDF for query terms
    idf = {}
    for term in query_tokens:
        df = sum(1 for doc in corpus.values() if term in doc["tokens"])
        idf[term] = math.log((N - df + 0.5) / (df + 0.5) + 1.0)

    scores = {}
    for filepath, doc in corpus.items():
        # Apply workspace/topic filters
        if workspace and workspace not in filepath:
            continue
        if topic and f"/{topic}/" not in filepath:
            continue

        dl = len(doc["tokens"])
        score = 0.0

        for term in query_tokens:
            tf = doc["tokens"].count(term)
            if tf == 0:
                continue
            numerator = tf * (k1 + 1)
            denominator = tf + k1 * (1 - b + b * dl / avg_dl)
            score += idf.get(term, 0) * numerator / denominator

        if score > 0:
            scores[filepath] = score

    return dict(sorted(scores.items(), key=lambda x: x[1], reverse=True)[:top_k])
```

---

## Phase 3: Knowledge Graph

Entity extraction and relationship tracking across all memory. This enables queries like "Who works at Acme Corp?" and "What connects Sarah to Project Alpha?"

### Data Formats

```python
# nodes.jsonl, one entity per line
{
    "id": "person:sarah-chen",
    "type": "person",           # person, venture, company, location, topic
    "name": "Sarah Chen",
    "aliases": ["sarah chen", "sarah-chen", "sarah"],
    "properties": {
        "company": "Acme Corp",
        "role": "CTO",
        "location": "Toronto, ON",
        "tags": "partner,technical"
    },
    "memory_ids": ["/path/to/contacts/sarah-chen.md"],
    "created": "2026-03-15",
    "updated": "2026-03-29"
}

# edges.jsonl, one relationship per line
{
    "source": "person:sarah-chen",
    "target": "venture:acme-corp",
    "type": "works_on",         # works_at, works_on, lives_in, spouse_of, knows
    "properties": {"role": "CTO"},
    "memory_ids": ["/path/to/contacts/sarah-chen.md"],
    "created": "2026-03-15"
}

# aliases.json, fast lookup (entity name -> node ID)
{
    "sarah chen": "person:sarah-chen",
    "sarah-chen": "person:sarah-chen",
    "sarah": "person:sarah-chen",
    "acme-corp": "venture:acme-corp",
    "acme corp": "venture:acme-corp"
}
```

### graph_builder.py, Entity Extraction

```python
import json
import re
import os
from datetime import datetime

GRAPH_DIR = "/path/to/knowledge/graph"
NODES_FILE = os.path.join(GRAPH_DIR, "nodes.jsonl")
EDGES_FILE = os.path.join(GRAPH_DIR, "edges.jsonl")
ALIASES_FILE = os.path.join(GRAPH_DIR, "aliases.json")


def slugify(name: str) -> str:
    """'Sarah Chen' -> 'sarah-chen'"""
    return re.sub(r'[^a-z0-9]+', '-', name.lower()).strip('-')


def find_or_create_node(nodes: dict, aliases: dict, name: str,
                        node_type: str, properties: dict | None = None,
                        memory_ids: list | None = None) -> str:
    """Find existing node by alias or create new one.

    Uses word-overlap dedup: if >50% words match and 2+ words overlap,
    treat as same entity. Prevents "Sarah Chen" and "Sarah M. Chen"
    from becoming separate nodes.
    """
    # Check aliases first
    name_lower = name.lower().strip()
    if name_lower in aliases:
        node_id = aliases[name_lower]
        # Update existing node
        if node_id in nodes:
            node = nodes[node_id]
            if memory_ids:
                node["memory_ids"] = list(set(node.get("memory_ids", []) + memory_ids))
            if properties:
                node["properties"].update({k: v for k, v in properties.items() if v})
            node["updated"] = datetime.now().strftime("%Y-%m-%d")
        return node_id

    # Word overlap dedup check
    name_words = set(name_lower.split())
    for existing_id, existing_node in nodes.items():
        if existing_node["type"] != node_type:
            continue
        existing_words = set(existing_node["name"].lower().split())
        overlap = name_words & existing_words
        if len(overlap) >= 2 and len(overlap) / max(len(name_words), len(existing_words)) > 0.5:
            # Same entity, add alias
            aliases[name_lower] = existing_id
            aliases[slugify(name)] = existing_id
            return existing_id

    # Create new node
    node_id = f"{node_type}:{slugify(name)}"
    nodes[node_id] = {
        "id": node_id,
        "type": node_type,
        "name": name,
        "aliases": [name_lower, slugify(name)],
        "properties": properties or {},
        "memory_ids": memory_ids or [],
        "created": datetime.now().strftime("%Y-%m-%d"),
        "updated": datetime.now().strftime("%Y-%m-%d"),
    }
    aliases[name_lower] = node_id
    aliases[slugify(name)] = node_id
    # Add first-name alias for people
    if node_type == "person" and " " in name:
        first = name.split()[0].lower()
        if first not in aliases:  # Don't overwrite existing
            aliases[first] = node_id

    return node_id


def extract_from_contact(filepath: str, nodes: dict, edges: dict,
                         aliases: dict) -> None:
    """Extract entities and relationships from a CRM contact .md file.

    Parses Mackay 66 format fields:
    **Company:** -> works_at edge
    **Ventures:** -> works_on edge
    **Location:** -> lives_in edge
    - Spouse name: -> spouse_of edge
    """
    with open(filepath) as f:
        content = f.read()

    # Extract name from first heading
    name_match = re.search(r'^#\s+(.+)$', content, re.MULTILINE)
    if not name_match:
        return
    name = name_match.group(1).strip()

    # Create person node
    person_id = find_or_create_node(nodes, aliases, name, "person",
                                     memory_ids=[filepath])

    # Extract and link company
    company_match = re.search(r'\*\*Company:\*\*\s*(.+)', content)
    if company_match and company_match.group(1).strip():
        company = company_match.group(1).strip()
        company_id = find_or_create_node(nodes, aliases, company, "company")
        role_match = re.search(r'\*\*Role:\*\*\s*(.+)', content)
        role = role_match.group(1).strip() if role_match else None
        add_edge(edges, person_id, company_id, "works_at",
                 properties={"role": role} if role else None,
                 memory_ids=[filepath])

    # Extract and link ventures
    ventures_match = re.search(r'\*\*Ventures:\*\*\s*(.+)', content)
    if ventures_match:
        for venture in ventures_match.group(1).split(","):
            venture = venture.strip()
            if venture:
                venture_id = find_or_create_node(nodes, aliases, venture, "venture")
                add_edge(edges, person_id, venture_id, "works_on",
                         memory_ids=[filepath])

    # Extract and link location
    location_match = re.search(r'\*\*Location:\*\*\s*(.+)', content)
    if location_match and location_match.group(1).strip():
        loc = location_match.group(1).strip()
        loc_id = find_or_create_node(nodes, aliases, loc, "location")
        add_edge(edges, person_id, loc_id, "lives_in", memory_ids=[filepath])

    # Extract spouse
    spouse_match = re.search(r'- Spouse name:\s*(.+)', content)
    if spouse_match and spouse_match.group(1).strip():
        spouse = spouse_match.group(1).strip()
        spouse_id = find_or_create_node(nodes, aliases, spouse, "person")
        add_edge(edges, person_id, spouse_id, "spouse_of", memory_ids=[filepath])


def extract_from_r2_analysis(analysis_json: str, nodes: dict, edges: dict,
                             aliases: dict) -> None:
    """Extract entities from R2 document analysis output.

    R2 sends: {"people": [...], "ventures": [...], "source": "filepath"}
    This auto-hooks R2 into the knowledge graph. Every processed document
    enriches the graph automatically.
    """
    data = json.loads(analysis_json)

    for person in data.get("people", []):
        name = person["name"]
        props = {k: v for k, v in person.items()
                 if k not in ("name", "mackay66") and v}
        person_id = find_or_create_node(
            nodes, aliases, name, "person",
            properties=props,
            memory_ids=[data.get("source", "")]
        )

        # Link to ventures
        for venture in data.get("ventures", []):
            if venture:
                venture_id = find_or_create_node(nodes, aliases, venture, "venture")
                add_edge(edges, person_id, venture_id, "works_on",
                         memory_ids=[data.get("source", "")])

        # Link to company
        if person.get("company"):
            company_id = find_or_create_node(
                nodes, aliases, person["company"], "company")
            add_edge(edges, person_id, company_id, "works_at",
                     properties={"role": person.get("role")},
                     memory_ids=[data.get("source", "")])
```

### graph_query.py, Traversal and Visualization

```python
def cmd_related_to(entity_name: str, nodes: dict, edges: list,
                   aliases: dict, edge_type: str | None = None) -> list[dict]:
    """Find all entities related to a given entity (1-hop)."""
    entity_id = resolve_id(entity_name, nodes, aliases)
    if not entity_id:
        return []

    adjacency = build_adjacency(edges)
    related = []
    for target_id, rel_type, edge in adjacency.get(entity_id, []):
        if edge_type and rel_type != edge_type:
            continue
        if target_id in nodes:
            related.append({
                "entity": nodes[target_id]["name"],
                "type": nodes[target_id]["type"],
                "relationship": rel_type,
                "properties": edge.get("properties", {})
            })
    return related


def cmd_path(from_entity: str, to_entity: str, nodes: dict, edges: list,
             aliases: dict, max_hops: int = 3) -> list[str] | None:
    """BFS shortest path between two entities."""
    from_id = resolve_id(from_entity, nodes, aliases)
    to_id = resolve_id(to_entity, nodes, aliases)
    if not from_id or not to_id:
        return None

    adjacency = build_adjacency(edges)
    queue = [(from_id, [from_id], [])]
    visited = {from_id}

    while queue:
        current, path, edge_types = queue.pop(0)
        if len(path) > max_hops + 1:
            continue

        for neighbor_id, rel_type, _ in adjacency.get(current, []):
            if neighbor_id == to_id:
                return {
                    "path": [nodes[nid]["name"] for nid in path + [neighbor_id]],
                    "relationships": edge_types + [rel_type],
                    "hops": len(path)
                }
            if neighbor_id not in visited:
                visited.add(neighbor_id)
                queue.append((neighbor_id, path + [neighbor_id],
                             edge_types + [rel_type]))

    return None  # No path found


def export_html(nodes: dict, edges: list, output_path: str = "graph.html") -> None:
    """Generate interactive vis.js HTML visualization.

    Color map: person=blue, venture=red, company=yellow, location=green
    Physics: barnesHut with gravitationalConstant=-3000, springLength=150
    Dark theme for readability.
    """
    COLOR_MAP = {
        "person": "#4A90D9", "venture": "#E74C3C",
        "company": "#F1C40F", "location": "#2ECC71",
        "topic": "#9B59B6", "event": "#E67E22", "document": "#95A5A6",
    }

    vis_nodes = []
    for node_id, node in nodes.items():
        vis_nodes.append({
            "id": node_id,
            "label": node["name"],
            "color": COLOR_MAP.get(node["type"], "#BDC3C7"),
            "title": f"{node['type']}: {node['name']}"
        })

    vis_edges = []
    for edge in edges:
        vis_edges.append({
            "from": edge["source"],
            "to": edge["target"],
            "label": edge["type"],
            "arrows": "to"
        })

    html = f"""<!DOCTYPE html>
<html><head>
<script src="https://unpkg.com/vis-network/standalone/umd/vis-network.min.js"></script>
<style>body {{ background: #1a1a2e; margin: 0; }} #graph {{ width: 100vw; height: 100vh; }}</style>
</head><body>
<div id="graph"></div>
<script>
new vis.Network(document.getElementById('graph'), {{
    nodes: new vis.DataSet({json.dumps(vis_nodes)}),
    edges: new vis.DataSet({json.dumps(vis_edges)})
}}, {{
    physics: {{ barnesHut: {{ gravitationalConstant: -3000, springLength: 150 }} }},
    nodes: {{ font: {{ color: '#fff' }} }},
    edges: {{ font: {{ color: '#888', size: 10 }}, color: '#555' }}
}});
</script></body></html>"""

    with open(output_path, "w") as f:
        f.write(html)
```

---

## Phase 4: CRM Relationship Health Scoring

```python
import math
from datetime import datetime, timedelta

class RelationshipHealth:
    """Score each CRM contact's relationship health.

    Three dimensions:
    - Recency: exponential decay (half-life = 14 days)
    - Frequency: interactions per month (6-month window)
    - Importance: tag-weighted (family=1.0, partner=0.9, acquaintance=0.3)

    Final score = 0.6 * recency + 0.4 * frequency
    """

    IMPORTANCE_WEIGHTS = {
        "family": 1.0, "inner_circle": 0.95, "partner": 0.9,
        "mentor": 0.85, "client": 0.8, "cofounder": 0.8,
        "investor": 0.75, "speaker": 0.7, "community": 0.6,
        "friend": 0.6, "vendor": 0.5, "media": 0.5,
        "acquaintance": 0.3, "self": 0.0,
    }

    RECENCY_HALFLIFE_DAYS = 14

    def score_contact(self, contact: dict) -> dict:
        """Score a single contact's relationship health."""
        interactions = self.parse_interactions(contact["content"])
        tags = contact.get("tags", "").lower().split(",")

        # Recency: exponential decay from last interaction
        if interactions:
            last = max(interactions, key=lambda x: x["date"])
            days_since = (datetime.now() - last["date"]).days
            recency = math.exp(-0.693 * days_since / self.RECENCY_HALFLIFE_DAYS)
        else:
            recency = 0.0

        # Frequency: interactions per month (6-month window)
        cutoff = datetime.now() - timedelta(days=180)
        recent = [i for i in interactions if i["date"] > cutoff]
        frequency = min(len(recent) / 6.0, 1.0)  # Cap at 1.0

        # Importance: highest matching tag weight
        importance = max(
            (self.IMPORTANCE_WEIGHTS.get(t.strip(), 0.3) for t in tags),
            default=0.3
        )

        # Final score
        health = 0.6 * recency + 0.4 * frequency

        return {
            "name": contact["name"],
            "health": round(health, 2),
            "recency": round(recency, 2),
            "frequency": round(frequency, 2),
            "importance": round(importance, 2),
            "days_since_contact": days_since if interactions else None,
            "interactions_6mo": len(recent),
            "needs_attention": health < 0.3 and importance > 0.5,
        }

    def parse_interactions(self, content: str) -> list[dict]:
        """Parse interaction log from CRM contact markdown."""
        interactions = []
        for match in re.finditer(r'- (\d{4}-\d{2}-\d{2}):\s*(.+)', content):
            try:
                date = datetime.strptime(match.group(1), "%Y-%m-%d")
                interactions.append({"date": date, "note": match.group(2)})
            except ValueError:
                pass
        return interactions
```

---

## Phase 5: Temporal Decay with Importance Tiers

### recall.py, The Complete Search Pipeline

```python
def temporal_decay(score: float, file_date_str: str | None,
                   importance: float | None = None,
                   decay_rate_override: float | None = None,
                   period_days: int = 30) -> float:
    """Apply temporal decay to a search score.

    Importance-tiered decay rates (production-tuned):
    - High importance (>=0.7): journals, contacts, rules -> 0.99/month (barely fades)
    - Standard (0.4-0.69): meeting notes, documents -> 0.95/month
    - Low importance (<0.4): session logs, status -> 0.85/month (fades fast)

    This means a 6-month-old journal entry retains 94% of its score,
    while a 6-month-old session log retains only 38%.
    """
    if not file_date_str:
        return score

    try:
        file_date = datetime.strptime(file_date_str[:10], "%Y-%m-%d")
    except (ValueError, TypeError):
        return score

    days_old = (datetime.now() - file_date).days
    if days_old <= 0:
        return score

    # Select decay rate based on importance tier
    if decay_rate_override:
        decay = decay_rate_override
    elif importance is not None:
        if importance >= 0.7:
            decay = 0.99  # Journal, contacts, distilled rules
        elif importance >= 0.4:
            decay = 0.95  # Standard documents
        else:
            decay = 0.85  # Logs, status updates
    else:
        decay = 0.95  # Default

    return score * (decay ** (days_old / period_days))


def search(query: str, top_k: int = 10, verbose: bool = False,
           workspace: str | None = None, topic: str | None = None,
           use_decay: bool = True, decay_rate: float | None = None,
           min_confidence: str = "low",
           use_hybrid: bool = True, use_graph: bool = True) -> list[dict]:
    """Complete hybrid search pipeline.

    1. FAISS semantic search (embeddings)
    2. BM25 keyword search
    3. RRF merge
    4. Knowledge graph boost (1.3x for connected entities)
    5. Temporal decay (importance-tiered)
    6. Confidence labeling (HIGH/MEDIUM/LOW/NOISE)
    7. Filtering (workspace, topic, min confidence)
    """
    # Over-fetch when filters are active (3x candidates)
    fetch_k = top_k * 3 if (workspace or topic) else top_k * 2

    # Step 1: FAISS semantic search
    query_embedding = get_embedding(query)
    query_vector = np.array([query_embedding], dtype="float32")
    distances, indices = faiss_index.search(query_vector, fetch_k)

    faiss_results = []
    for dist, idx in zip(distances[0], indices[0]):
        if idx < 0:
            continue
        meta = metadata_store.get(str(idx))
        if not meta:
            continue
        # Apply workspace/topic filters
        if workspace and meta.get("workspace") != workspace:
            continue
        if topic and meta.get("topic") != topic:
            continue

        score = max(0, 1 - dist)  # Convert L2 distance to similarity
        faiss_results.append({**meta, "score": score, "source": "faiss"})

    # Step 2: BM25 keyword search
    bm25_scores = {}
    if use_hybrid:
        query_tokens = tokenize(query)
        bm25_scores = bm25_search(query_tokens, bm25_corpus,
                                   workspace=workspace, topic=topic)

    # Step 3: RRF merge
    if use_hybrid and bm25_scores:
        results = reciprocal_rank_fusion(faiss_results, bm25_scores, metadata_store)
        score_key = "rrf_score"
    else:
        results = faiss_results
        score_key = "score"

    # Step 4: Knowledge graph boost
    graph_boosted = 0
    if use_graph:
        aliases, nodes, edges = load_graph()
        boosted_files = graph_boost_files(query, aliases, nodes, edges)
        GRAPH_BOOST = 1.3
        for result in results:
            if result["file"] in boosted_files:
                result[score_key] *= GRAPH_BOOST
                result["graph_boosted"] = True
                graph_boosted += 1

    # Step 5: Temporal decay
    if use_decay:
        for result in results:
            result[score_key] = temporal_decay(
                result[score_key],
                result.get("date"),
                importance=result.get("importance"),
                decay_rate_override=decay_rate
            )

    # Step 6: Sort and apply confidence labels
    results.sort(key=lambda x: x[score_key], reverse=True)
    for result in results:
        result["confidence"] = confidence_label(result[score_key])

    # Step 7: Filter by min confidence
    min_score = CONFIDENCE_THRESHOLDS.get(min_confidence, 0)
    results = [r for r in results if r[score_key] >= min_score]

    if verbose:
        hybrid_str = f"hybrid: FAISS+BM25 (RRF)" if use_hybrid else "FAISS only"
        graph_str = f"graph: {graph_boosted} boosted" if use_graph else "graph: off"
        print(f"[{hybrid_str}] [{graph_str}] [{len(results)} results]")

    return results[:top_k]


def graph_boost_files(query: str, aliases: dict, nodes: dict,
                      edges: list) -> set[str]:
    """Find files connected to entities mentioned in the query.

    1. Extract entity mentions from query via alias substring matching
    2. Look up entity node IDs
    3. Traverse 1 hop in graph to find related entities
    4. Collect all memory_ids from matched + related nodes
    """
    query_lower = query.lower()
    matched_ids = set()

    for alias, node_id in aliases.items():
        if len(alias) > 2 and alias in query_lower:
            matched_ids.add(node_id)

    # 1-hop traversal
    adjacency = build_adjacency(edges)
    related_ids = set()
    for node_id in matched_ids:
        for neighbor_id, _, _ in adjacency.get(node_id, []):
            related_ids.add(neighbor_id)

    # Collect all memory files from matched + related entities
    boosted_files = set()
    for nid in matched_ids | related_ids:
        if nid in nodes:
            boosted_files.update(nodes[nid].get("memory_ids", []))

    return boosted_files
```

---

## Phase 6: Dedup-Aware Memory Storage

### remember.py

```python
import hashlib
import json
import os
from datetime import datetime

DEDUP_INDEX = "/path/to/workspace/memory/.dedup_hashes.json"


def content_hash(text: str) -> str:
    """SHA256 of normalized text for exact dedup."""
    normalized = " ".join(text.lower().split())  # Collapse whitespace
    return hashlib.sha256(normalized.encode()).hexdigest()


def check_near_duplicate(content: str) -> tuple[bool, str | None, float, str | None]:
    """Check for near-duplicate via embedding similarity.

    Calls recall.py with --no-decay --top 1 and checks if score >= 0.95.
    Returns (is_dup, preview, score, filepath).
    """
    result = subprocess.run(
        [RECALL_BIN, RECALL_PY, content[:200], "--top", "1",
         "--no-decay", "--min-confidence", "noise"],
        capture_output=True, text=True
    )
    # Parse recall output for top result score
    # If score >= 0.95, it's a near-duplicate
    if top_score >= 0.95:
        return (True, preview, top_score, filepath)
    return (False, None, top_score, None)


def remember(content: str, topic: str = "general", files: list | None = None,
             title: str | None = None, force: bool = False) -> dict:
    """Store a memory with dedup protection.

    1. Exact dedup: SHA256 hash check (instant, free)
    2. Near dedup: embedding similarity check (~$0.0001)
    3. Write markdown with frontmatter
    4. Update dedup index

    --force bypasses all dedup checks.
    """
    # Step 1: Exact dedup
    if not force:
        dedup_index = load_dedup_index()
        h = content_hash(content)
        if h in dedup_index:
            return {"status": "duplicate", "existing": dedup_index[h]}

    # Step 2: Near dedup
    if not force:
        is_near, preview, score, existing_file = check_near_duplicate(content)
        if is_near:
            return {
                "status": "near_duplicate",
                "similarity": score,
                "existing": existing_file,
                "preview": preview
            }

    # Step 3: Write memory file
    date_str = datetime.now().strftime("%Y-%m-%d")
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    slug = slugify(title or topic)[:60]
    filename = f"{date_str}_{slug}.md"
    filepath = os.path.join(MEMORY_BASE, TOPIC_DIRS.get(topic, ""), filename)

    header = f"# {title or topic.capitalize()} -- {date_str}\n"
    header += f"_Stored: {timestamp}_\n\n"
    body = f"## Entry ({timestamp})\n{content}\n"

    # Append attached files
    if files:
        for fpath in files[:5]:  # Max 5 attachments
            with open(fpath) as f:
                body += f"\n### Source: {os.path.basename(fpath)}\n```\n{f.read()}\n```\n"

    mode = "a" if os.path.exists(filepath) else "w"
    with open(filepath, mode) as f:
        if mode == "w":
            f.write(header)
        f.write(body)

    # Step 4: Update dedup index
    os.chmod(filepath, 0o660)  # Group accessible (production fix: mkstemp creates 600)
    dedup_index[content_hash(content)] = filepath
    save_dedup_index(dedup_index)

    return {"status": "ok", "file": filepath, "topic": topic, "mode": "create" if mode == "w" else "append"}
```

---

## Phase 7: Advanced Tools

### Decision Journal

```python
# decisions.py, zero LLM cost, pure JSONL CRUD
# Storage: knowledge/decisions.jsonl

DECISION_SCHEMA = {
    "id": "a1b2c3d4",          # SHA256[:8] of decision+date
    "decision": "Switch assistant to Flash model",
    "date": "2026-03-29",
    "venture": "acme-corp",     # Optional venture context
    "expected": "88% cost savings",
    "actual": None,              # Filled later via 'outcome' command
    "outcome_date": None,
    "tags": ["cost", "optimization"],
    "context": "Gemini Pro costs $8/day for 190 requests..."
}

# Commands:
# decisions.py add --decision "..." --venture "..." --expected "..." --tags "..."
# decisions.py list --venture "..." --tag "..." --since "2026-03-01"
# decisions.py outcome --id a1b2c3d4 --actual "Actual: $0.80/day, quality unchanged"
# decisions.py pending --days 30   (decisions with no outcome after N days)
```

### Conflict Resolution

```python
# conflict_resolver.py, detects contradictions between memories
# Uses: recall.py (similarity) + graph (shared entities) + GPT-4o-mini (arbitration)

def check_conflict(new_content: str, dry_run: bool = False) -> dict | None:
    """Check if new content conflicts with existing memories.

    1. Run recall.py to find HIGH-confidence matches
    2. Extract entities from both texts
    3. Check if entities share a knowledge graph node
    4. If shared entity + high similarity, potential conflict
    5. Call GPT-4o-mini to arbitrate: merge, replace, keep-both, discard
    """
    # Find similar existing content
    similar = search(new_content[:200], top_k=5, min_confidence="high")

    for match in similar:
        # Extract entities and check graph overlap
        new_entities = extract_entities(new_content)
        old_entities = extract_entities(match["content"])
        shared = find_shared_graph_entity(new_entities, old_entities)

        if shared and match["score"] >= CONFLICT_THRESHOLD:
            if dry_run:
                return {"conflict": True, "existing": match["file"],
                        "shared_entity": shared, "score": match["score"]}

            # LLM arbitration (~$0.001 per conflict)
            decision = llm_arbitrate(match["full_content"], new_content)
            return {
                "conflict": True,
                "action": decision["action"],  # merge|replace|keep-both|discard
                "reason": decision["reason"],
                "existing": match["file"]
            }

    return None  # No conflict
```

### Trend Tracking

```python
# trends.py, weekly topic frequency analysis from FAISS metadata
# Output: knowledge/distilled/trends.md

def build_frequency_table(metadata: dict, weeks_back: int = 4) -> dict:
    """Group FAISS metadata by (workspace, topic, ISO_week)."""
    table = {}
    for meta in metadata.values():
        workspace = meta.get("workspace", "unknown")
        topic = meta.get("topic", "unknown")
        date_str = meta.get("date")
        if not date_str:
            continue
        week = date_to_iso_week(date_str)
        key = (workspace, topic)
        table.setdefault(key, {}).setdefault(week, 0)
        table[key][week] += 1
    return table


def calculate_changes(table: dict) -> list[dict]:
    """Calculate week-over-week changes."""
    changes = []
    for (workspace, topic), weeks in table.items():
        sorted_weeks = sorted(weeks.keys())
        if len(sorted_weeks) < 2:
            continue
        current = weeks[sorted_weeks[-1]]
        previous = weeks[sorted_weeks[-2]]
        pct = ((current - previous) / previous * 100) if previous > 0 else 100
        trend = "rising" if pct > 10 else "falling" if pct < -10 else "stable"
        changes.append({
            "workspace": workspace, "topic": topic,
            "current": current, "previous": previous,
            "pct_change": pct, "trend": trend
        })
    return sorted(changes, key=lambda x: abs(x["pct_change"]), reverse=True)
```

### Proactive Recall (Forgotten Connections)

```python
# proactive_recall.py, surfaces memories you've forgotten about
# Scans yesterday's session logs, extracts entities, finds old connections

MIN_AGE_DAYS = 14  # Only surface memories older than 2 weeks

def get_yesterday_topics() -> list[str]:
    """Extract entities + headers from yesterday's session logs."""
    yesterday = (datetime.now() - timedelta(days=1)).strftime("%Y-%m-%d")
    topics = []

    for workspace in WORKSPACES:
        pattern = os.path.join(workspace, "memory", "logs", f"{yesterday}*.md")
        for filepath in glob.glob(pattern):
            with open(filepath) as f:
                content = f.read()
            # Extract capitalized names
            names = re.findall(r'\b([A-Z][a-z]+ [A-Z][a-z]+)\b', content)
            topics.extend(names)
            # Extract markdown headers
            headers = re.findall(r'^##\s+(.+)$', content, re.MULTILINE)
            topics.extend(headers)

    return list(set(topics))[:10]  # Cap at 10


def find_forgotten_connections() -> list[dict]:
    """For each topic from yesterday, find related old memories."""
    topics = get_yesterday_topics()
    connections = []

    for topic in topics:
        results = search(topic, top_k=3, use_decay=False,
                         min_confidence="medium")
        for result in results:
            days_old = (datetime.now() - parse_date(result["date"])).days
            if days_old >= MIN_AGE_DAYS:
                connections.append({
                    "topic": topic,
                    "memory": result["content"][:100],
                    "file": result["file"],
                    "days_old": days_old,
                })

    # Dedup by file, sort by age (oldest = most forgotten)
    seen = set()
    unique = []
    for c in sorted(connections, key=lambda x: x["days_old"], reverse=True):
        if c["file"] not in seen:
            seen.add(c["file"])
            unique.append(c)

    return unique[:5]  # Top 5 forgotten connections
```

### Smart Bookmarks

```python
# bookmarks.py, AI-summarized read-later queue, FAISS-indexed
# Storage: workspace/memory/bookmarks/ (auto-indexed by memory_sync.py)

def save_bookmark(url: str, tags: str = "", context: str = "") -> dict:
    """Fetch URL, summarize with GPT-4o-mini, save as searchable markdown.

    Cost: ~$0.001 per bookmark (GPT-4o-mini summary of 6000 chars)
    """
    # Fetch content (try node content.js first, fallback to urllib)
    content = fetch_content(url)
    if not content:
        return {"status": "error", "message": "Could not fetch URL"}

    title = extract_title(content, url)

    # Summarize with GPT-4o-mini (3 bullet points)
    summary = llm_client.call(
        "openai/gpt-4o-mini",
        "Summarize in 3 bullet points. Be concise.",
        content[:6000],
        max_tokens=200
    )

    # Write markdown (auto-indexed by FAISS watcher)
    date = datetime.now().strftime("%Y-%m-%d")
    slug = slugify(title)[:60]
    filepath = f"workspace/memory/bookmarks/{date}_{slug}.md"

    md = f"""# {title}
**URL:** {url}
**Saved:** {date}
**Tags:** {tags}
**Context:** {context}
**Read:** no

## Summary
{summary.text}

## Excerpt
{content[:2000]}
"""
    with open(filepath, "w") as f:
        f.write(md)
    os.chmod(filepath, 0o660)

    return {"status": "ok", "file": filepath, "title": title}
```

---

## JKE: Knowledge Engine (Autolearning)

### Architecture

```
Signal Sources -> Capture (real-time) -> Fast Rules (instant) -> Synthesis (weekly) -> Knowledge Base
```

### Signal Capture (Real-Time, Zero LLM Cost)

```python
# Every agent captures corrections and confirmations in JSONL format
# Append-only, zero LLM cost

# Correction signal (user corrected the agent)
{"timestamp": "2026-03-29T14:23:00Z", "agent": "main", "type": "correction",
 "what": "Used doc-processor instead of R2 for insurance document",
 "should_be": "ALL documents go through R2",
 "context": "User uploaded insurance policy PDF"}

# Confirmation signal (user confirmed non-obvious approach)
{"timestamp": "2026-03-29T15:00:00Z", "agent": "main", "type": "confirmation",
 "what": "Bundled all CRM updates into single PR",
 "context": "User said 'yes exactly, keep doing that'"}
```

### Fast Rules (Instant, Zero LLM Cost)

```python
# synthesize.py --fast, appends correction to fast_rules.jsonl immediately
# No LLM call needed, just structured JSONL append
# Weekly synthesis consolidates fast rules into canonical distilled/*.md

def add_fast_rule(correction_json: str) -> None:
    """Instant knowledge capture. Zero LLM cost."""
    rule = json.loads(correction_json)
    rule["timestamp"] = datetime.now().isoformat()
    rule["consolidated"] = False  # Marked True during weekly synthesis

    with open("knowledge/distilled/fast_rules.jsonl", "a") as f:
        f.write(json.dumps(rule) + "\n")
```

### Weekly Synthesis (GPT-4o-mini, ~$0.01)

```python
# synthesize.py, runs every Sunday 3:33 AM ET
# Reads: corrections/, confirmations/, session logs, fast_rules.jsonl
# Calls: GPT-4o-mini for pattern extraction
# Updates: distilled/mistakes.md, rules.md, preferences.md, insights.md, agent_briefing.md

def weekly_synthesis():
    """Consolidate all signals into actionable knowledge."""
    # Collect inputs
    corrections = load_jsonl("knowledge/corrections/")
    confirmations = load_jsonl("knowledge/confirmations/")
    fast_rules = load_jsonl("knowledge/distilled/fast_rules.jsonl",
                            filter=lambda r: not r.get("consolidated"))
    session_logs = load_recent_sessions(days=7, max_chars=3000)

    prompt = f"""Analyze these AI assistant interaction signals and extract patterns.

CORRECTIONS (agent made mistakes):
{format_signals(corrections)}

CONFIRMATIONS (approach validated):
{format_signals(confirmations)}

FAST RULES (instant corrections):
{format_signals(fast_rules)}

SESSION CONTEXT (last 7 days):
{session_logs}

Return JSON:
{{
    "mistakes": [{{"pattern": "...", "what_went_wrong": "...", "correct_approach": "..."}}],
    "rules": [{{"rule": "...", "context": "..."}}],
    "preferences": [{{"preference": "...", "evidence": "..."}}],
    "insights": [{{"insight": "...", "source": "..."}}],
    "briefing_bullets": ["Top 5 things agents must know this week"]
}}

Only include NEW patterns not already in the existing distilled files."""

    result = llm_client.call("openai/gpt-4o-mini", prompt, max_tokens=2000)
    knowledge = json.loads(result.text)

    # Append to distilled files (timestamped sections)
    week_header = f"## Week of {datetime.now().strftime('%Y-%m-%d')}\n"
    append_to_file("knowledge/distilled/mistakes.md", week_header,
                   format_mistakes(knowledge["mistakes"]))
    append_to_file("knowledge/distilled/rules.md", week_header,
                   format_rules(knowledge["rules"]))
    # ... similar for preferences and insights

    # Update agent briefing
    write_briefing("knowledge/agent_briefing.md", knowledge["briefing_bullets"])

    # Mark fast rules as consolidated
    mark_consolidated("knowledge/distilled/fast_rules.jsonl")
```

### Knowledge Base Structure

```
knowledge/
|-- graph/                          # Knowledge graph
|   |-- nodes.jsonl                 # Entity nodes (person, venture, company, location)
|   |-- edges.jsonl                 # Relationships (works_at, spouse_of, etc.)
|   +-- aliases.json                # Entity alias -> node ID lookup
|-- decisions.jsonl                 # Decision journal with outcomes
|-- corrections/                    # Raw correction signals (JSONL, append-only)
|   |-- 2026-03-25.jsonl
|   +-- 2026-03-29.jsonl
|-- confirmations/                  # Raw confirmation signals (JSONL)
|-- distilled/                      # Weekly synthesis output
|   |-- mistakes.md                 # Consolidated mistake patterns
|   |-- rules.md                    # Procedural rules
|   |-- preferences.md              # User working style
|   |-- insights.md                 # System patterns
|   |-- trends.md                   # Weekly topic frequency
|   +-- fast_rules.jsonl            # Instant corrections (pre-synthesis)
|-- performance/                    # Engagement metrics
|   |-- social_patterns.md          # Social platform analysis
|   +-- social_log.jsonl            # Engagement time-series
|-- agent_briefing.md               # Weekly rotating summary (all agents read at startup)
|-- archive/                        # Expired signals (>90 days)
|   +-- conflicts/                  # Conflict resolution archives
+-- JKE.md                          # System documentation
```

---

## Morning Report (13-Section Aggregator)

```python
# compile.py, aggregates data from 10+ subsystems into a daily brief
# Runs as cron: 6:00 AM ET daily
# Each section has independent error handling (fails gracefully)

SECTIONS = [
    ("Calendar",            "cal.py list --from today --to tomorrow --max 20"),
    ("Control Tower",       "ct.py status"),
    ("Payment Alerts",      "finance.py alerts --days 7"),
    ("Health Alerts",       "doc.py alerts"),
    ("CRM Follow-ups",     "crm.py followup --list --days 7"),
    ("Relationships",       "relationship_health.py --top 5 --format morning-report"),
    ("Social Status",       None),  # Read ghost_status.md directly
    ("Yesterday",           None),  # Read yesterday's session logs
    ("Reminders",           None),  # Read REMINDERS.md
    ("Trends",              None),  # Read knowledge/distilled/trends.md
    ("Pending Decisions",   "decisions.py pending --days 30"),
    ("Memory Connections",  "proactive_recall.py --format morning-report"),
    ("Unread Bookmarks",    "bookmarks.py unread"),
]

def compile_morning_report() -> str:
    """Build 13-section morning report.

    Each section runs independently with 15-30s timeout.
    If a section fails, it's omitted (not the whole report).
    """
    report = f"# Morning Report -- {datetime.now().strftime('%A, %B %d, %Y')}\n\n"

    for section_name, command in SECTIONS:
        try:
            if command:
                result = subprocess.run(
                    command.split(),
                    capture_output=True, text=True, timeout=30,
                    cwd=WORKSPACE_DIR
                )
                content = result.stdout.strip()
            else:
                content = read_static_section(section_name)

            if content:
                report += f"## {section_name}\n{content}\n\n"
        except (subprocess.TimeoutExpired, Exception) as e:
            # Graceful degradation, skip this section
            report += f"## {section_name}\n_Error: {str(e)[:50]}_\n\n"

    return report
```

---

## Memory Directory Structure

```
workspace/
|-- memory/
|   |-- contacts/                   # CRM (Mackay 66 .md + _index.json)
|   |-- journal/                    # Private journal entries (YYYY-MM-DD_entry.md)
|   |-- logs/                       # Session summaries + weekly condensations ONLY
|   |-- finances/                   # Financial documents
|   |-- travel/                     # Travel plans
|   |-- bookmarks/                  # Smart bookmarks (FAISS-indexed)
|   |-- assessments/                # Personality assessments
|   |-- social/                     # Social platform data
|   +-- .dedup_hashes.json          # Hash-based dedup index
|-- workspace-ct/memory/            # Venture data (per-venture subdirs)
|-- workspace-doc/memory/           # Health data (per-person subdirs, JSONL)
|-- workspace-agent3/memory/        # Family/homeschool data
|-- workspace-agent4/memory/        # Genealogy data
|-- workspace-r2/memory/            # R2 processing logs + holding area
+-- knowledge/                      # Shared knowledge base (see JKE section)
```

---

*Continue to Part 4: Skills & Operations*
