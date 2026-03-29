# TESSA & COLE - Part 5 of 6
## Build Plan, Code Reference & Data Formats

---

## Build Phases (Production-Informed)

These phases are ordered by dependency and value. Each phase produces a working system. You can stop at any phase and have something useful.

### Phase 1: Foundation (Sessions 1-2)
**Goal:** A working CLI that routes requests and calls LLMs with Minimum Viable Context.

```
[ ] Project structure /opt/tessa/
[ ] .env loader + config.py (API keys, model configs, budget limits)
[ ] Unified LLM client (Gemini, OpenAI, Anthropic, Mistral, DeepSeek)
    - Provider routing by model ID prefix (google/, openai/, anthropic/)
    - Cost calculation per provider
    - Response dataclass (text, tokens, cost, latency)
[ ] Cost guard + circuit breaker
    - Per-person daily budgets
    - Per-call token warning threshold
    - 3-retry circuit breaker with 5-min cooldown
    - Cost ledger (JSONL append-only)
[ ] Router (rule-based classification)
    - Regex patterns for 10+ query types
    - Entity extraction
    - Tier assignment (1=local, 2=light, 3=deep)
[ ] Context Chef
    - .md recipe loader
    - Section resolver (load only needed memory files)
    - Model picker (tier -> cheapest available model)
    - Token estimator
[ ] Memory reader/writer
    - Read .md files from workspace directories
    - Atomic write with temp file + rename
    - File permission management (660 for group access)
[ ] FastAPI server
    - POST /api/chat (main endpoint)
    - GET /api/health
    - Passphrase auth middleware
```

**Verification:**
```bash
# Start server
uvicorn server.main:app --host 127.0.0.1 --port 8100

# Test classification (should return tier + workflow)
curl -X POST http://localhost:8100/api/chat \
  -H "Authorization: Bearer YOUR_PASSPHRASE" \
  -d '{"message": "tell me a joke", "person": "alex"}'

# Test with context (should load only meeting-related memory)
curl -X POST http://localhost:8100/api/chat \
  -d '{"message": "brief me on Acme Corp meeting", "person": "alex"}'

# Check cost ledger
cat data/cost_ledger.jsonl | python -m json.tool
```

### Phase 2: Knowledge System (Sessions 3-4)
**Goal:** FAISS + BM25 hybrid search with temporal decay and confidence labels.

```
[ ] FAISS indexer (memory_sync.py)
    - File watcher (watchdog library)
    - Sliding window chunking (20K chars, 2K overlap)
    - Metadata enrichment (workspace, topic, date, importance)
    - text-embedding-3-small (1536 dimensions)
    - BM25 corpus builder (alongside FAISS)
[ ] recall.py (hybrid search)
    - FAISS semantic search
    - BM25 keyword search
    - RRF fusion (K=60)
    - Temporal decay (importance-tiered: 0.99/0.95/0.85)
    - Confidence labels (HIGH/MEDIUM/LOW/NOISE)
    - Workspace/topic filtering
    - Over-fetch for filtered queries (3x candidates)
[ ] remember.py (dedup storage)
    - SHA256 exact dedup
    - Embedding near-dedup (>0.95 similarity)
    - .dedup_hashes.json index
    - --force bypass flag
[ ] SQLite FTS5 (optional, for backward compatibility with Open WebUI)
[ ] Systemd service for FAISS watcher
```

**Verification:**
```bash
# Index existing memory files
python memory_sync.py --backfill

# Test semantic search
python recall.py "Acme Corp meeting" --top 5 --verbose

# Test keyword search
python recall.py "invoice 4521" --top 5 --verbose

# Test dedup
python remember.py --content "Test memory" --topic general
python remember.py --content "Test memory" --topic general  # Should warn: duplicate

# Check index health
python recall.py --status  # Should show vector count, index size, freshness
```

### Phase 3: Knowledge Graph (Sessions 5-6)
**Goal:** Entity extraction, relationship tracking, graph-boosted recall.

```
[ ] graph_builder.py
    - Extract entities from CRM contacts (regex parsing)
    - Extract entities from R2 analysis output (JSON parsing)
    - Node types: person, venture, company, location
    - Edge types: works_at, works_on, lives_in, spouse_of, knows
    - Alias management (first name, slug, full name -> same node)
    - Word-overlap dedup for similar names
[ ] graph_query.py
    - Related entities (1-hop traversal)
    - Shortest path (BFS, configurable max hops)
    - Entity search (fuzzy substring matching)
    - HTML visualization (vis.js export)
[ ] Graph-boosted recall (modify recall.py)
    - Load aliases + nodes + edges
    - Extract entity mentions from query via alias matching
    - 1-hop graph traversal to find related entities
    - 1.3x boost for graph-connected results
    - --no-graph flag to disable
[ ] Auto-hooks
    - FAISS watcher triggers graph_builder on contact file changes
    - R2 triggers graph_builder after people extraction
```

**Verification:**
```bash
# Build graph from existing CRM contacts
python graph_builder.py --from-crm

# Check graph
python graph_query.py --list --type person
python graph_query.py --related-to "Sarah Chen"
python graph_query.py --path-from "Sarah" --path-to "Acme Corp"

# Test graph boost in search
python recall.py "Sarah" --verbose  # Should show [graph: N boosted]

# Export visualization
python graph_query.py --export-html graph.html
# Open graph.html in browser
```

### Phase 4: Core Skills (Sessions 7-8)
**Goal:** CRM, Control Tower, Doc, R2, the four pillars.

```
[ ] crm.py (CRM with Mackay 66)
    - add, update, find, log, followup commands
    - Base template + 12 archetype sections
    - _index.json for fast structured queries
    - Atomic file operations
    - Word-overlap similarity check for duplicate contacts
[ ] ct.py (Control Tower)
    - Ventures (CRUD, hierarchical, status tracking)
    - Todos (CRUD, dedup, venture/person assignment)
    - Meeting logs
    - CT.json state file
    - 60% word-overlap dedup for todos
[ ] doc.py (Health Intelligence)
    - JSONL time-series storage (labs, vitals, weight, prescriptions)
    - Per-person profiles (auto-generated PROFILE.md)
    - Unit conversion (16 rules, accent-insensitive)
    - Per-agent access control
    - Health alerts for morning report
[ ] r2.py (Document Intelligence)
    - Smart model selection (auto/flash/pro)
    - Routing table (document type -> destination)
    - Cross-extraction pipeline (CRM, CT, Doc, Finance, Calendar, Graph)
    - Known aliases (avoid confirmation loops)
    - Todo dedup (60% word overlap)
    - Incomplete items array (ask user when unsure)
    - Cost logging per document
```

**Verification:**
```bash
# CRM
python crm.py add --name "Test Contact" --company "Test Co" --role "CEO"
python crm.py find --name "Test"
python crm.py update --name "Test Contact" --key "Hobbies" --value "Golf"
python crm.py log --name "Test Contact" --note "Initial meeting"

# Control Tower
python ct.py add-venture --name "Test Venture" --status active
python ct.py add-todo --venture "test-venture" --task "Build TESSA & COLE" --due 2026-04-01 --priority high
python ct.py status

# Doc
python doc.py labs --person alex --test "Glucose" --value 102 --unit "mg/dL" --ref-high 110
python doc.py profile --person alex

# R2
python r2.py process --file /path/to/test-transcript.md --context "Test meeting" --model flash
```

### Phase 5: Signal Integration + Proactive Intelligence (Sessions 9-10)
**Goal:** Signal bot, morning report, cron jobs.

```
[ ] Signal endpoint (/api/signal)
    - Person identification by phone number
    - Budget checking
    - Script-routed queries (Tier 1, 0 tokens)
    - LLM queries (Tier 2-3, context-assembled)
    - Cron verbatim relay
[ ] Morning report (compile.py, 13 sections)
    - Calendar, CT, Finance, Health, CRM, Relationships
    - Social platform status, Yesterday, Reminders
    - Trends, Decisions, Connections, Bookmarks
[ ] Cron architecture
    - Morning report (daily 6 AM ET)
    - Log condensation (weekly Sunday 3 AM ET)
    - JKE synthesis (weekly Sunday 3:33 AM ET)
    - Backup (daily 5 AM UTC)
[ ] JKE autolearning
    - Correction/confirmation capture (JSONL, real-time)
    - Fast rules (instant, zero LLM)
    - Weekly synthesis (GPT-4o-mini, ~$0.01)
    - Agent briefing generation
[ ] Relationship health scoring
[ ] Proactive recall (forgotten connections)
```

### Phase 6: Magic Links + Advanced Tools (Sessions 11-12)
**Goal:** Shareable links, decision journal, conflict resolution.

```
[x] Magic link sidecar (aiohttp server), done on OpenClaw (2026-03-28), port to TESSA & COLE
    - [x] Scoped chat (rate-limited, TTL, max turns)
    - [x] Visual content (HTML rendering, feedback queue)
    - [x] Live pages (WebSocket push updates)
    - [x] R2 extraction on expiry
    - [x] cleanup.py + systemd timer (daily 4AM UTC, 48h grace)
    - [x] fail2ban jail (10 hits/5min = 1h ban)
    - [x] Security watchdog health check
[ ] Decision journal (decisions.py)
    - add, list, outcome, pending commands
    - JSONL storage with venture/tag filtering
[ ] Conflict resolver
    - Recall + graph entity matching
    - GPT-4o-mini arbitration
    - Archive old versions
[ ] Trend tracking
    - Weekly topic frequency from FAISS metadata
    - Week-over-week change calculation
    - Optional LLM summary
[ ] Smart bookmarks
    - URL fetch + GPT-4o-mini summary
    - FAISS-indexed for search
    - Read/unread tracking
[ ] Skill manifest registry
```

### Phase 7: Security Hardening + Open Source (Sessions 13-14)
**Goal:** Production security, documentation, public release.

```
[ ] LUKS vault setup script
[ ] age encryption for secrets
[ ] SSH hardening config
[ ] fail2ban configuration
[ ] Security watchdog (20+ checks)
[ ] Backup encryption (AES-256 GPG)
[ ] Caddy configuration with security headers
[ ] systemd service files (tessa, tessa-memory, magic-link, magic-link-cleanup, timers)
[ ] README.md, CONTRIBUTING.md, LICENSE (AGPL v3)
[ ] Setup script (one-command install)
[ ] Docker compose (optional)
[ ] GitHub publish
```

---

## Complete Directory Structure

```
/opt/tessa/
├── server/
│   ├── __init__.py
│   ├── main.py                     # FastAPI app, routes, middleware
│   ├── config.py                   # Settings, env loading, model configs
│   ├── router/
│   │   ├── __init__.py
│   │   ├── classifier.py           # Rule-based + LLM fallback classification
│   │   ├── context_chef.py         # Minimum Viable Context assembler
│   │   ├── model_picker.py         # Tier -> model selection
│   │   └── cost_guard.py           # Budget checks, circuit breaker, ledger
│   ├── workflows/
│   │   ├── __init__.py
│   │   ├── engine.py               # DAG-based workflow executor
│   │   └── builtins.py             # Meeting prep, transcript, general chat
│   ├── memory/
│   │   ├── __init__.py
│   │   ├── reader.py               # .md file reader with section parsing
│   │   ├── writer.py               # Atomic write with staging/backup
│   │   ├── indexer.py              # SQLite FTS5 index
│   │   └── sync.py                 # FAISS + BM25 indexer (memory_sync.py)
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── orchestrator.py         # Agent selection and execution
│   │   └── registry.py             # Agent config loader
│   ├── skills/
│   │   ├── __init__.py
│   │   └── registry.py             # Skill manifest scanner
│   ├── api/
│   │   ├── __init__.py
│   │   ├── llm_client.py           # Unified LLM client (all providers)
│   │   ├── routes_chat.py          # POST /api/chat
│   │   ├── routes_signal.py        # POST /api/signal
│   │   ├── routes_memory.py        # GET /api/memory/search
│   │   ├── routes_metrics.py       # GET /api/metrics/*
│   │   └── routes_admin.py         # Agent/skill management
│   ├── insights/
│   │   ├── __init__.py
│   │   ├── scheduler.py            # Cron job scheduling
│   │   ├── commitments.py          # Promise/commitment tracking
│   │   ├── patterns.py             # Usage pattern detection
│   │   └── self_improver.py        # Auto-optimization suggestions
│   ├── processing/
│   │   ├── __init__.py
│   │   ├── text_extract.py         # PDF/image text extraction
│   │   ├── chunker.py              # Sliding window text chunker
│   │   ├── summarizer.py           # LLM-based summarization
│   │   └── formatter.py            # Markdown formatting utilities
│   ├── cache/
│   │   ├── __init__.py
│   │   └── response_cache.py       # TTL-based response cache
│   └── security/
│       ├── __init__.py
│       ├── auth.py                 # Passphrase + TOTP authentication
│       ├── audit.py                # Access audit logging
│       └── sanitizer.py            # Input sanitization (XSS, injection)
│
├── workspace/                      # Main agent workspace (vault symlink)
│   ├── SOUL.md                     # Agent personality
│   ├── TOOLS.md                    # Tool reference + routing table
│   ├── USER.md                     # User profile
│   ├── memory/
│   │   ├── contacts/               # CRM contacts (.md + _index.json)
│   │   ├── journal/                # Journal entries
│   │   ├── logs/                   # Session summaries ONLY
│   │   ├── finances/               # Financial docs
│   │   ├── bookmarks/              # Smart bookmarks
│   │   └── .dedup_hashes.json      # Dedup index
│   └── skills/
│       ├── memory-manager/scripts/ # recall, remember, graph, decisions, etc.
│       ├── crm/scripts/            # crm.py, relationship_health.py
│       ├── control-tower/scripts/  # ct.py
│       ├── r2/scripts/             # r2.py
│       ├── doc/scripts/            # doc.py
│       ├── morning-report/scripts/ # compile.py
│       ├── finance-tracker/scripts/
│       ├── google-calendar/scripts/
│       ├── brave-search/scripts/
│       ├── doc-processor/scripts/
│       ├── email-monitor/scripts/
│       ├── security-watchdog/scripts/
│       ├── model-switch/scripts/
│       ├── transcript-processor/scripts/
│       ├── log-condenser/scripts/
│       ├── jke-synthesizer/scripts/
│       ├── magic-link/scripts/
│       └── skill_registry.py
│
├── workspace-ct/                   # Control Tower data
│   ├── CT.json                     # State file
│   └── memory/{venture}/           # Per-venture: meetings/, transcripts/, docs/
├── workspace-doc/                  # Health data
│   └── memory/{person}/            # Per-person: PROFILE.md, vitals.jsonl, labs.jsonl
├── workspace-agent1/               # Additional agent workspace
├── workspace-r2/                   # R2 processing logs
│   └── memory/                     # Holding area for unrouted documents
│
├── knowledge/                      # Shared knowledge base (JKE)
│   ├── graph/                      # nodes.jsonl, edges.jsonl, aliases.json
│   ├── decisions.jsonl             # Decision journal
│   ├── corrections/                # JSONL signals
│   ├── confirmations/              # JSONL signals
│   ├── distilled/                  # mistakes.md, rules.md, preferences.md, insights.md
│   ├── performance/                # Engagement metrics
│   ├── agent_briefing.md           # Weekly rotating summary
│   └── archive/                    # Expired signals, conflict archives
│
├── agents/                         # Agent definitions
│   ├── main/agent/                 # SOUL.md, TOOLS.md
│   ├── agent1/agent/
│   └── .../
│
├── config/
│   ├── budgets.md                  # Per-person budget config
│   ├── models.md                   # Model ladder config
│   ├── people.md                   # Person routing config
│   └── insight_schedules.md        # Cron schedule config
│
├── context-recipes/                # .md context recipes
│   ├── meeting_prep.md
│   ├── general_chat.md
│   ├── health_query.md
│   ├── document_processing.md
│   └── deep_analysis.md
│
├── workflows/                      # .md workflow definitions
│   ├── transcript_processing.md
│   ├── meeting_prep.md
│   ├── general_chat.md
│   └── document_ingest.md
│
├── magic-links/                    # Magic link data
│   ├── links.json                  # Token store
│   ├── sessions/                   # Chat JSONL logs
│   ├── content/                    # HTML content
│   ├── feedback/                   # Feedback JSONL
│   └── summaries/                  # Auto-generated summaries
│
├── scripts/
│   ├── setup.py                    # One-command install
│   ├── migrate_from_openclaw.py    # OpenClaw migration tool
│   ├── batch_ingest.py             # Bulk memory import
│   └── build_index.py              # Rebuild all indexes
│
├── data/
│   ├── tessa.db                    # SQLite FTS5 index
│   ├── cost_ledger.jsonl           # All API calls logged
│   ├── staging/                    # Write staging area
│   ├── cache/                      # Response cache
│   ├── checkpoints/                # Workflow checkpoints
│   └── backups/                    # Encrypted daily backups
│
├── tests/
│   ├── test_router.py
│   ├── test_context_chef.py
│   ├── test_recall.py
│   ├── test_crm.py
│   ├── test_ct.py
│   └── test_r2.py
│
├── .env                            # API keys (chmod 600)
├── .env.example                    # Template
├── requirements.txt
├── docker-compose.yml              # Optional containerization
├── LICENSE                         # AGPL v3
└── README.md
```

---

## Data Format Reference

### Cost Ledger Entry (JSONL)

```json
{
    "timestamp": "2026-03-29T14:23:01Z",
    "person": "alex",
    "workflow": "meeting_prep",
    "model": "anthropic/claude-sonnet-4",
    "provider": "anthropic",
    "input_tokens": 2800,
    "output_tokens": 450,
    "cost_usd": 0.03,
    "latency_ms": 2100,
    "tier": 2,
    "context_sections_loaded": ["people/acme-corp", "meetings", "commitments"],
    "context_sections_referenced": ["people/acme-corp", "commitments"],
    "context_sections_wasted": ["meetings"],
    "success": true,
    "classification": "meeting_prep",
    "classification_method": "rule_match",
    "circuit_breaker_trips": 0,
    "tags": {"entity": "acme-corp", "channel": "signal"}
}
```

### CRM Contact (Mackay 66 Markdown)

```markdown
# Sarah Chen

## Quick Reference
- **Company:** Acme Corp
- **Role:** CTO
- **Tags:** partner, technical
- **Ventures:** Acme Corp
- **Email:** sarah@acmecorp.example.com
- **Phone:** +1-555-000-1234
- **LinkedIn:** /in/sarahchen
- **Location:** Toronto, ON
- **Birthday:** 1990-03-22
- **Source:** Acme Corp kickoff meeting, 2024-01

## Mackay 66
- Marital status: Married
- Spouse name: David Chen
- Children: Lily (4, born June 2022)
- Education: MIT, Computer Science
- Hobbies: Rock climbing, board games
- Previous employment: Amazon (2016-2023)
- Hometown: Seattle

## Speaker Profile
- Topics: Distributed systems, cloud infrastructure
- Events: Tech Summit 2025
- Fee: N/A (co-founder)
- Agent: N/A

## Interaction Log
- 2026-03-15: Discussed Q2 launch timeline. Positive about direction.
- 2026-02-20: Quick call about partnership terms. Mentioned shoulder injury from climbing.
- 2026-01-10: Initial kickoff meeting. Established weekly sync.

## Notes
Prefers morning meetings. Vegetarian. Loves escape rooms.
```

### CT State File (CT.json)

```json
{
    "ventures": [
        {
            "name": "Acme Corp",
            "slug": "acme-corp",
            "status": "active",
            "role": "Director of Product",
            "priority": 1,
            "parent": null,
            "urls": ["https://acmecorp.example.com"],
            "description": "Next-generation publishing platform"
        }
    ],
    "todos": [
        {
            "id": 1,
            "venture": "acme-corp",
            "task": "Send revised proposal to Sarah",
            "due": "2026-04-01",
            "priority": "high",
            "person": "Sarah Chen",
            "status": "open",
            "created": "2026-03-15"
        }
    ],
    "next_todo_id": 2
}
```

### Health Data (JSONL)

```json
{"date": "2026-03-29", "type": "lab", "test": "Glucose", "value": 102.0, "unit": "mg/dL", "ref_high": 110.0, "ref_low": 70.0, "person": "alex", "converted": "5.66 mmol/L", "flag": "normal"}
{"date": "2026-03-29", "type": "bp", "value": "120/80", "person": "alex", "notes": "Morning reading"}
{"date": "2026-03-29", "type": "weight", "value": 82.5, "unit": "kg", "person": "alex"}
{"date": "2026-03-29", "type": "prescription", "name": "Metformin", "dose": "500mg", "frequency": "BID", "person": "alex", "status": "active"}
```

### Knowledge Graph (JSONL)

```json
{"id": "person:sarah-chen", "type": "person", "name": "Sarah Chen", "aliases": ["sarah chen", "sarah-chen", "sarah"], "properties": {"company": "Acme Corp", "role": "CTO", "location": "Toronto, ON"}, "memory_ids": ["/path/to/contacts/sarah-chen.md"], "created": "2026-03-15", "updated": "2026-03-29"}
{"source": "person:sarah-chen", "target": "venture:acme-corp", "type": "works_on", "properties": {"role": "CTO"}, "memory_ids": ["/path/to/contacts/sarah-chen.md"], "created": "2026-03-15"}
```

### Decision Journal (JSONL)

```json
{"id": "a1b2c3d4", "decision": "Switch your assistant to Gemini Flash", "date": "2026-03-27", "venture": "acme-corp", "expected": "88% cost savings, same quality", "actual": "Daily cost dropped from $8 to $0.80. Quality unchanged for routine tasks.", "outcome_date": "2026-03-29", "tags": ["cost", "optimization"], "context": "Gemini Pro costs $8/day for 190 requests. Flash is 8x cheaper."}
```

### FAISS Metadata (JSON)

```json
{
    "0": {
        "file": "/path/to/workspace/memory/contacts/sarah-chen.md",
        "content": "# Sarah Chen\n\n## Quick Reference\n- **Company:** Acme Corp...",
        "chunk": 0,
        "workspace": "workspace",
        "topic": "contacts",
        "date": "2026-03-15",
        "importance": 0.55
    }
}
```

### JKE Correction Signal (JSONL)

```json
{"timestamp": "2026-03-29T14:23:00Z", "agent": "main", "type": "correction", "what": "Used doc-processor instead of R2 for insurance document", "should_be": "ALL documents go through R2", "context": "User uploaded insurance policy PDF"}
```

### Magic Link Token (JSON)

```json
{
    "token": "a1b2c3d4e5f6...",
    "type": "chat",
    "scope": "playdate coordinator for Alex and Lily",
    "context": "Available dates: April 5-10. Location: Riverdale Park.",
    "model": "google/gemini-2.5-flash",
    "ttl_hours": 24,
    "max_turns": 20,
    "rate_limit_per_min": 3,
    "one_time": false,
    "used": false,
    "created": "2026-03-29T10:00:00Z",
    "expires": "2026-03-30T10:00:00Z"
}
```

---

## API Reference

### Core Endpoints

```
POST /api/chat              # Main chat endpoint (router + context chef + LLM)
POST /api/signal            # Signal bot webhook
POST /api/upload            # Document upload -> R2 processing
GET  /api/health            # Service health check
POST /api/auth/login        # Passphrase + optional TOTP
```

### Memory Endpoints

```
GET  /api/memory/search?q=...&top=10&workspace=...&topic=...
GET  /api/memory/sections?person=...   # List available memory sections
POST /api/memory/remember              # Store new memory (with dedup)
GET  /api/memory/graph?entity=...      # Knowledge graph query
GET  /api/memory/graph/html            # Graph visualization
```

### Metrics Endpoints

```
GET  /api/metrics/today                # Today's cost/token summary
GET  /api/metrics/history?days=30      # Historical cost data
GET  /api/metrics/by-workflow          # Cost breakdown by workflow
GET  /api/metrics/by-model             # Cost breakdown by model
GET  /api/metrics/by-person            # Cost breakdown by person
```

### Agent & Skill Endpoints

```
GET  /api/agents                       # List all agents
GET  /api/agents/{id}                  # Agent config
PUT  /api/agents/{id}/model            # Switch agent model
GET  /api/skills                       # List all skills
GET  /api/skills/{name}               # Skill manifest
```

### Insight Endpoints

```
GET  /api/insights/daily               # Today's morning report
GET  /api/insights/weekly              # Weekly analysis
GET  /api/insights/commitments         # Open commitments
GET  /api/insights/decisions?pending   # Pending decisions
GET  /api/insights/trends              # Topic trends
GET  /api/insights/relationships       # Relationship health scores
GET  /api/learnings                    # JKE learnings summary
```

### Workflow Endpoints

```
GET  /api/workflows                    # List available workflows
POST /api/workflows/execute            # Execute a workflow manually
GET  /api/workflows/{name}/status      # Workflow execution status
```

### Magic Link Endpoints (matches OpenClaw implementation)

```
# Internal API (Bearer auth via MAGIC_LINK_SECRET)
POST /internal/create                  # Create new link (chat/visual/live)
POST /internal/update/{token}          # Push HTML content (visual/live)
GET  /internal/feedback/{token}        # Read pending feedback
DELETE /internal/{token}               # Revoke link + generate summary
POST /internal/summarize/{token}       # Generate summary without revoking
GET  /internal/list                    # List active/recent links

# Public endpoints (Caddy at /m/*)
GET  /m/{token}                        # Render chat/visual/live page
POST /m/{token}/message                # HTTP fallback for chat
WS   /m/{token}/ws                     # WebSocket (chat + visual/live push + feedback)
```

---

## Migration from OpenClaw

```python
# scripts/migrate_from_openclaw.py

"""
Migration tool: OpenClaw -> TESSA & COLE

What it does:
1. Copies workspace directories (preserving structure)
2. Copies knowledge/ directory (graph, decisions, JKE data)
3. Copies agent definitions
4. Copies skill scripts
5. Copies FAISS index + metadata + BM25 corpus
6. Generates TESSA & COLE config from openclaw.json
7. Creates context recipes from existing TOOLS.md
8. Sets up systemd services

What it does NOT do:
- Move API keys (copy .env manually)
- Transfer Signal sessions (messaging daemon state stays)
- Transfer encrypted vault (create new LUKS or mount existing)

Usage:
  python migrate_from_openclaw.py \
    --source /path/to/openclaw/data \
    --dest /mnt/tessa_vault/.tessa \
    --dry-run
"""

OPENCLAW_TO_TESSA_MAP = {
    # OpenClaw path -> TESSA & COLE path
    "workspace/": "workspace/",
    "workspace-agent1/": "workspace-agent1/",
    "workspace-ct/": "workspace-ct/",
    "workspace-doc/": "workspace-doc/",
    "workspace-agent2/": "workspace-agent2/",
    "workspace-agent3/": "workspace-agent3/",
    "workspace-r2/": "workspace-r2/",
    "workspace-agent4/": "workspace-agent4/",
    "workspace-agent5/": "workspace-agent5/",
    "knowledge/": "knowledge/",
    "agents/": "agents/",
}
```

---

## Claude Code Quick Start

```
"I am building TESSA & COLE (Token-Efficient Self-hosted Smart Agent) v4.0.
Here are the 6-part build documents: [paste all 6 parts]

Start Phase 1: Foundation.
1. Create project /opt/tessa/ with the directory structure from Part 5
2. Build config.py -- loads .env, model configs, budget limits
3. Build llm_client.py -- UnifiedLLMClient with Gemini, OpenAI, Anthropic
4. Build cost_guard.py -- per-person budgets, circuit breaker, cost ledger (JSONL)
5. Build classifier.py -- regex patterns from Part 2 QUERY_ROUTING table
6. Build context_chef.py -- .md recipe loader, section resolver, model picker
7. Build reader.py + writer.py -- .md memory file I/O with atomic writes
8. Build main.py -- FastAPI with /api/chat, /api/signal, /api/health
9. Add passphrase auth middleware

Python 3.12 + type hints. Model-agnostic. AGPL v3.
All configs in .md files. All data in .md or JSONL.
Real code from production, not pseudocode.
Priority: working router + context chef + cost guard.
Test with: curl -X POST http://localhost:8100/api/chat -d '{\"message\":\"tell me a joke\"}'
"
```

---

## Success Metrics

### Cost
- Average tokens/request under 3K (was 18K)
- Daily API cost under $2 (was $8+)
- Zero-token requests (Tier 1) at 40%+ of total
- No error loops past 3 retries (circuit breaker)
- Monthly total under $30 (including droplet)

### Quality
- Meeting briefs useful 90%+ of the time
- Commitments caught 90%+ (no more "you promised and forgot")
- Search relevant at 95%+ (hybrid FAISS+BM25+Graph)
- R2 cross-extraction catches all people, todos, health data
- PRE-RESPONSE GATE catches 99%+ of hallucinated execution

### Learning
- 5+ JKE optimizations suggested per week
- Context waste (loaded but unused sections) down 10%/month
- Model costs down 5%/month via auto-downgrade suggestions

### Compatibility
- Open WebUI .md files work without modification
- OpenClaw migration under 1 hour
- Signal switch = one endpoint config change

---

## Appendix: Requirements

```
# requirements.txt
fastapi>=0.110.0
uvicorn>=0.29.0
httpx>=0.27.0             # Async HTTP client for LLM APIs
python-dotenv>=1.0.0
pydantic>=2.0
faiss-cpu>=1.8.0          # Vector search (no GPU needed)
numpy>=1.26.0
openai>=1.14.0            # OpenAI + text-embedding-3-small
google-generativeai>=0.5.0 # Gemini API
anthropic>=0.25.0         # Claude API
watchdog>=4.0.0           # File system watcher
aiohttp>=3.9.0            # Magic link sidecar
python-multipart>=0.0.9   # File uploads
```

---

**End of TESSA & COLE v4.0 Build Documents**

*These 6 documents represent the complete architecture, informed by production deployment on a 4GB DigitalOcean droplet running multiple agents, 20+ skills, and processing hundreds of daily requests. Every pattern described here has been battle-tested against real usage.*
