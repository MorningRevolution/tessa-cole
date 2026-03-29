# TESSA & COLE - Part 1 of 6
## Token-Efficient Self-hosted Smart Agent & Context-Optimized Learning Engine
### The Most Token-Efficient AI Command Center
### Compatible with OpenClaw and Open WebUI

**Tagline:** *No more waking up to a massive API bill.*
**Version:** 4.0 | **Date:** March 29, 2026 | **License:** AGPL v3

---

## Origin Story

See the README for the full origin story. The short version: a $10 overnight API bill caused by an error loop (900 calls, each loading 18,000 tokens of system context just to fail and retry) revealed a fundamental problem. It was not just the errors. It was every call. "Tell me a joke" loaded 18,000 tokens of context. "What time is my meeting" loaded 18,000 tokens. A transcript that failed and retried three times loaded 54,000 tokens for nothing.

The math was brutal. 200 calls a day at 18,000 tokens each = 3.6 million tokens daily. At mixed model rates, that is $30-100 a month, and most of those tokens were completely unnecessary for the task at hand.

So we built TESSA & COLE.

Then we deployed them on a real production system running OpenClaw with 9 agents, 20+ skills, processing meeting transcripts, managing a CRM with Mackay 66 relationship intelligence, tracking family health data, running a social media agent, and coordinating a household. Every principle in this document was battle-tested against real daily usage. The system prompt went from 57KB to 15KB (74% reduction). The daily API cost went from $8 to under $1. The memory system evolved through 7 phases from basic keyword search to a hybrid FAISS+BM25+Knowledge Graph pipeline.

TESSA & COLE look at every incoming request and ask: "What do you actually need to answer this?" If it is a joke, send 200 tokens to the cheapest model. If it is meeting prep, pull just that person's file and recent meetings, maybe 3,000 tokens to a mid-tier model. If it is a deep analysis of 20 years of journal data, then yes, load the full context and use the best model. But justify every token.

The result: a 70-95% reduction in token costs. Same quality. Often better, because the right model gets the right context instead of drowning in irrelevant information.

No more waking up to a massive API bill. That is the promise.

---

## What TESSA & COLE Are

TESSA & COLE are a self-hosted, privacy-first AI command center that masters the art of **Minimum Viable Context**. They are a knowledge system that auto-learns and evolves, sending just the right amount of data to the right model to get the right result while keeping track of every token spent.

TESSA & COLE are model-agnostic: Gemini, Claude, GPT, Mistral, DeepSeek, and optionally Ollama. They pick the best model for each task based on cost, quality, and capability.

TESSA & COLE are fully compatible with **OpenClaw** (AI gateway for Signal/web) and **Open WebUI**. Same .md file formats for memory, skills, agents, and workflows. Run TESSA & COLE standalone, as a router in front of OpenClaw/Open WebUI, or alongside them sharing the same memory directory.

TESSA & COLE are freedom technology, licensed under AGPL v3. Like Fedi, Mastodon, and Nextcloud, the code stays open forever.

### TESSA & COLE ARE:
- A token-efficient AI orchestration engine and command center
- A hybrid knowledge system (FAISS vectors + BM25 keywords + Knowledge Graph) that auto-learns
- Masters of Minimum Viable Context: right data, right model, right result
- A smart router that triages every request before spending a single token
- A dynamic context assembler (the Context Chef)
- A workflow execution engine with markdown-based definitions
- A multi-agent coordinator supporting hundreds of agents and thousands of skills
- A document intelligence pipeline (R2) that cross-extracts data to CRM, todos, health, finance
- A proactive insight generator (13-section morning reports, commitment tracking, trend analysis)
- A self-improving system that gets smarter and cheaper over time (JKE autolearning)
- Compatible with OpenClaw and Open WebUI file formats and conventions
- A production-grade security system with defense in depth, encrypted at rest with LUKS

### TESSA & COLE ARE NOT:
- A fork or plugin of OpenClaw or Open WebUI (standalone system that integrates with them)
- A chatbot (they orchestrate chatbots as tools)
- A cloud service (runs on your hardware, even a $12/month droplet)
- Tied to any single AI provider (model-agnostic by design)
- Dependent on local LLMs or GPUs (API-first, Ollama optional)

---

## Guiding Principles

### P1: Minimum Viable Context
The core philosophy. Every token costs money and time. TESSA & COLE ensure the absolute minimum context is assembled for every task. "Tell me a joke" = 0 context tokens. "Meeting prep for Acme Corp" = only that person's file + recent meetings. Never the full blob.

**Production proof:** We reduced a 57KB system prompt (997 lines across 3 files) to 15KB (356 lines), a 74% reduction, by splitting into always-loaded core (SOUL.md + TOOLS.md) and on-demand extended tools (TOOLS_EXTENDED.md). Daily API cost dropped from $8 to under $1.

### P2: Hybrid Knowledge System (Battle-Tested)
TESSA & COLE's knowledge is not just stored. It is searchable three ways simultaneously:
- **FAISS** (semantic vectors): "Find conversations where someone seemed frustrated"
- **BM25** (keyword matching): "Find exact mentions of invoice #4521"
- **Knowledge Graph** (relationship traversal): "Who works with Sarah at Acme Corp?"

Results are merged via **Reciprocal Rank Fusion (RRF)** and boosted by graph connections (1.3x). Temporal decay ensures recent memories rank higher. Seven phases of production evolution went into this.

### P3: Right Model for the Right Job
Gemini Flash for grunt work ($0.15/1M tokens). Claude Haiku for summarization. Sonnet for synthesis. Opus for deep insight. GPT-4o-mini for cheap general tasks. TESSA & COLE pick the optimal model per task. No single provider is hardcoded.

**Production proof:** R2 document intelligence auto-selects Flash for short docs and Pro for long transcripts with 3+ participants. The model switch skill lets users change any agent's model via a single command.

### P4: Gateway Compatible (OpenClaw + Open WebUI)
Same file formats: markdown (.md) for memory, knowledge, skills, agents, and workflows. Same directory conventions. If it works in OpenClaw or Open WebUI, it works in TESSA & COLE, and vice versa.

**OpenClaw compatibility layer:** TESSA & COLE can run as middleware between Signal/Web and OpenClaw, intercepting requests to apply Minimum Viable Context before forwarding. Or they can run standalone with their own FastAPI server.

### P5: Privacy by Architecture
All data stays on your hardware. API calls send only the minimum context needed. Memory files encrypted at rest with LUKS, with defense in depth through layered encryption. All access audited. Per-agent access control (e.g., Jordan's agent can't read Alex's health data).

### P6: Fail Cheaply
Circuit breakers stop retry loops after 3 attempts. Budget caps prevent runaway costs. Every error logged with its token cost. The $10 nightmare never happens again.

**Production proof:** We implemented per-person daily budgets, per-call token warnings, and model-tier restrictions. Family members get cheaper models and restricted workflow access.

### P7: Proactive Intelligence
Scheduled analysis: 13-section morning reports, weekly trend reports, commitment tracking, relationship health scoring. "You told Acme Corp three times you'd send the proposal and haven't." TESSA & COLE work for you even when you don't ask.

**Production proof:** Morning report now includes: Calendar, Control Tower todos, Payment Alerts, Health Alerts, CRM Follow-ups, Relationships to Nurture, Social Platform Status, Yesterday Highlights, Reminders, Trends, Pending Decisions, Memory Connections, Unread Bookmarks.

### P8: Self-Improving (JKE Autolearning)
TESSA & COLE capture corrections and confirmations from every interaction, store them as JSONL signals, and run weekly GPT-4o-mini synthesis to extract patterns. Mistakes get consolidated into avoidance rules. Preferences get codified. Cost: ~$0.01/week.

**Production proof:** The Jarvis Knowledge Engine (JKE) runs on all 9 agents. Fast rules are appended instantly (zero LLM cost). Weekly synthesis consolidates into distilled knowledge files (mistakes.md, rules.md, preferences.md, insights.md). An agent briefing is generated every Sunday.

### P9: Script-First Data Management
**Never manually create .md files for structured data.** Always use scripts (crm.py for contacts, ct.py for todos, doc.py for health, transcript.py for meetings). Manual files bypass indexing and are invisible to future queries.

**Production proof:** We lost data twice by manually creating contact .md files that bypassed `_index.json`. The scripts handle atomic writes, dedup checks, index updates, and FAISS re-indexing automatically.

### P10: Markdown Everything
All configuration, memory, skills, agents, and workflows are .md files. Human-readable, version-controllable, gateway-compatible.

### P11: Anti-Hallucination by Design (PRE-RESPONSE GATE)
Four mechanical checks every agent must run before every reply:
1. **GATE 1: DID I EXECUTE?** Find the tool call. No tool call = hallucination. Stop and run the command.
2. **GATE 2: RIGHT TOOL?** Document->R2, contact->crm.py, todo->ct.py. Decision table, not paragraphs.
3. **GATE 3: EXACT DATA?** Copy-paste numbers/names from user message. Never retype from memory.
4. **GATE 4: SHELL SAFE?** Dollar amounts in single quotes (`'$282.50'`), never double quotes.

**Production proof:** Without this gate, your assistant repeatedly said "I've updated the records" without running any commands. The gate reduced hallucinated execution to near-zero.

### P12: Freedom Technology (AGPL v3)
Free and open source forever. Even if someone offers it as a hosted service, they must share modifications. Like Fedi, Mastodon, and Nextcloud.

---

## Architecture Overview

```
                         +----------------------------+
                         |      INPUT CHANNELS        |
                         |  Signal . Web UI . API     |
                         |  Magic Links . Cron        |
                         +-------------+--------------+
                                       |
                      +----------------v----------------+
                      |           THE ROUTER            |
                      |  Classify . Identify Person     |
                      |  Check Budget . Select Workflow  |
                      |  PRE-RESPONSE GATE              |
                      +----------------+----------------+
                                       |
                      +----------------v----------------+
                      |        THE CONTEXT CHEF         |
                      |     (Minimum Viable Context)    |
                      |  Determine needed knowledge     |
                      |  Assemble only what is needed   |
                      |  Pick optimal model             |
                      |  Estimate token cost            |
                      +----------------+----------------+
                                       |
           +---------------------------+---------------------------+
           |                           |                           |
+----------v-----------+  +-----------v------------+  +-----------v------------+
|   TIER 1: LOCAL      |  |   TIER 2: LIGHT        |  |   TIER 3: DEEP         |
|   Zero tokens        |  |   1-4K tokens          |  |   5-25K tokens         |
|   Script execution   |  |   Flash/Haiku/Mini     |  |   Sonnet/Pro/Opus      |
|   Calendar lookups   |  |   Focused context      |  |   Full analysis        |
|   CRM/CT queries     |  |   Single workflow      |  |   Multi-step R2        |
|   Health data        |  |   Simple chat          |  |   Deep synthesis       |
+----------------------+  +------------------------+  +------------------------+
                                       |
                      +----------------v----------------+
                      |      AGENT ORCHESTRATOR         |
                      |  Select agents . Assign skills  |
                      |  DAG parallel execution         |
                      |  R2 document intelligence       |
                      |  Cross-extraction pipeline      |
                      +----------------+----------------+
                                       |
        +----------+------------------++--------------+---------------+
        |          |                  |               |               |
   KNOWLEDGE   SKILL            COST          INSIGHT         AUTO-
   SYSTEM      REGISTRY         INTEL         ENGINE          LEARNER
   FAISS+BM25  Script-first    Budget         13-section      JKE
   +Graph      CRM/CT/Doc     Circuit         Morning         Corrections
   Hybrid      R2/Magic       Breaker         Report          Fast Rules
   Recall      Links          Ledger          Trends          Weekly Synth
```

### Production Topology (OpenClaw Reference Implementation)

```
Internet -> Caddy (443/TLS) -+-> TESSA Router (127.0.0.1:8100) -> OpenClaw Gateway (127.0.0.1:YOUR_GATEWAY_PORT)
                              |                                          |
                              |                                   messaging daemon (127.0.0.1:YOUR_MSG_PORT)
                              +-> Magic Link Sidecar (127.0.0.1:YOUR_SIDECAR_PORT) [/m/* routes]
                                        |
                            LUKS Encrypted Vault (/mnt/tessa_vault)
                                        |
                            All config, keys, agent data live here
                            Symlinked from /home/tessa/.tessa
```

### Token Savings Table (Production Measured)

| Task | Without TESSA & COLE | With TESSA & COLE | Savings |
|------|---------------------|-------------------|---------|
| "Tell me a joke" | 18,000 | 200 | 99% |
| "What is my schedule?" | 18,000 | 0 (local cal.py) | 100% |
| Meeting prep brief | 18,000 | 3,000 | 83% |
| Process 1 transcript | 18,000 x N retries | 5,000 (R2 auto-select) | 72% |
| Weekly planning | 18,000 | 6,000 | 67% |
| Error retry x 3 | 54,000 | 0 (circuit breaker) | 100% |
| CRM contact lookup | 18,000 | 0 (local crm.py) | 100% |
| Health data query | 18,000 | 0 (local doc.py) | 100% |
| Morning report (13 sections) | N/A | ~5,000 (batched) | New capability |
| Deep analysis | 18,000 | 20,000 (justified) | Cost matches value |

### Monthly Cost Comparison (Production Data)

**Without TESSA & COLE (typical OpenClaw/Open WebUI):**
- 200 calls/day x 18,000 tokens x 30 days = 108M tokens/month
- Plus error loops, retry storms
- Monthly API cost: $30-100+ (measured: $8/day = $240/month on Gemini Pro)

**With TESSA & COLE on $12/month DigitalOcean droplet:**
- System prompt: 15KB (was 57KB), 74% reduction
- Model: Gemini Flash (was Pro), 88% per-token savings
- 160 calls at ~1,000 avg + 40 calls at ~5,000 avg = ~11M tokens/month
- R2 auto-selects Flash for 80% of documents, Pro only for complex transcripts
- Monthly API cost: $5-15. Plus droplet: $12. Total: $17-27/month
- **Measured: under $1/day after optimization**

---

## Model-Agnostic Architecture

### Supported Providers

| Provider | Models | Best For | Cost (Input) |
|----------|--------|----------|------|
| Google Gemini | 2.5 Flash, 2.5 Pro | Cheap grunt work, long context, instruction following | $0.15-1.25/1M |
| Anthropic Claude | Haiku 4.5, Sonnet 4, Opus 4 | Synthesis, reasoning, deep analysis | $0.80-15.00/1M |
| OpenAI | GPT-4o-mini, GPT-5-mini, GPT-5 | General purpose, multimodal, health intelligence | $0.15-2.50/1M |
| Mistral | Small, Medium | European languages, efficient | $0.10-0.60/1M |
| DeepSeek | V3 | Code generation, technical | $0.27/1M |
| Ollama (optional) | Any local model | Zero-cost if you have 20GB+ RAM | $0 |

### Model Ladder (Production-Tested)

```python
MODEL_LADDER = {
    "tier_1_cheap": [
        "google/gemini-2.5-flash",     # $0.15/1M - best value
        "openai/gpt-4o-mini",          # $0.15/1M - good general
        "mistral-small",               # $0.10/1M - European languages
    ],
    "tier_2_balanced": [
        "google/gemini-2.5-flash",     # Handles 25KB+ instruction files
        "anthropic/claude-haiku-4-5",  # Fast summarization
        "openai/gpt-4o-mini",          # Cheap general
    ],
    "tier_2_quality": [
        "anthropic/claude-sonnet-4",   # Best synthesis
        "google/gemini-2.5-pro",       # Long context analysis
        "openai/gpt-5-mini",           # Health intelligence (good at structured data)
    ],
    "tier_3_deep": [
        "anthropic/claude-opus-4",     # Deepest reasoning
        "google/gemini-2.5-pro",       # Long document analysis
        "openai/gpt-5",               # Maximum capability
    ],
}

# Production-learned model shortcuts
MODEL_SHORTCUTS = {
    "flash":    "google/gemini-2.5-flash",
    "pro":      "google/gemini-2.5-pro",
    "gpt":      "openai/gpt-4o-mini",
    "gpt5":     "openai/gpt-5-mini",
    "gpt5full": "openai/gpt-5",
    "haiku":    "anthropic/claude-haiku-4-5",
    "sonnet":   "anthropic/claude-sonnet-4",
    "opus":     "anthropic/claude-opus-4",
}
```

### Smart Model Selection (R2 Pattern)

R2 document intelligence auto-selects the right model based on document characteristics:

```python
def select_model(document_text: str, context: str, model_override: str = "auto") -> str:
    """Auto-select the cheapest model that can handle this document well."""
    if model_override != "auto":
        return MODEL_SHORTCUTS.get(model_override, model_override)

    doc_len = len(document_text)
    context_lower = context.lower() if context else ""

    # Long transcripts with multiple speakers need Pro
    if doc_len > 8000 and any(kw in context_lower for kw in ["transcript", "meeting"]):
        print(f"  Auto-selected Gemini 2.5 Pro (document: {doc_len:,} chars, transcript)")
        return "google/gemini-2.5-pro"

    # Everything else: Flash is sufficient
    print(f"  Auto-selected Gemini 2.5 Flash (document: {doc_len:,} chars)")
    return "google/gemini-2.5-flash"
```

### Capability-Aware Routing

Models tagged with capabilities: code, reasoning, math, creative, multilingual, health, structured_data. On failure, TESSA & COLE escalate to a stronger model in the same capability class, not just any bigger model.

### Unified LLM Client

```python
class UnifiedLLMClient:
    """Single interface to all providers. Production-tested."""

    providers = {
        "google":    GeminiProvider(),
        "anthropic": AnthropicProvider(),
        "openai":    OpenAIProvider(),
        "mistral":   MistralProvider(),
        "deepseek":  DeepSeekProvider(),
        "ollama":    OllamaProvider(),
    }

    def get_provider_for_model(self, model: str) -> LLMProvider:
        """Route model ID to provider. Handles 'google/gemini-2.5-flash' format."""
        if "/" in model:
            provider_name = model.split("/")[0]
        else:
            provider_name = self.infer_provider(model)
        return self.providers[provider_name]

    async def call(self, model: str, system_prompt: str, user_message: str,
                   max_tokens: int = 1000) -> LLMResponse:
        provider = self.get_provider_for_model(model)
        response = await provider.complete(model, system_prompt, user_message, max_tokens)
        cost = self.calculate_cost(model, response.usage)

        # Log to cost ledger (every call, always)
        self.cost_ledger.log(CostEntry(
            model=model, provider=provider.name,
            input_tokens=response.usage.input,
            output_tokens=response.usage.output,
            cost_usd=cost, latency_ms=response.latency
        ))

        return LLMResponse(
            text=response.text, model=model,
            input_tokens=response.usage.input,
            output_tokens=response.usage.output,
            cost=cost, latency_ms=response.latency
        )
```

---

## System Prompt Architecture (Production-Optimized)

### The Token Floor Problem

Every AI gateway (OpenClaw, Open WebUI, etc.) has a minimum system prompt that loads on every call. In OpenClaw production:

| Source | Tokens | Controllable? |
|--------|--------|---------------|
| Gateway framework (tool descriptions, safety rules) | ~10,000 | No |
| Native skills (always included) | ~1,000 | No |
| Workspace files (SOUL.md, TOOLS.md, etc.) | ~2,500 | **Yes** |

**Total floor: ~13,500 tokens per call.** This cannot be reduced without modifying the gateway's source code.

### TESSA & COLE's Solution: Layered Context Loading

Instead of loading everything every time, TESSA & COLE split context into layers:

| Layer | Loaded | Tokens | Purpose |
|-------|--------|--------|---------|
| **Core Identity** (SOUL.md) | Always | ~570 | Who the agent is, cost-aware orchestrator framing |
| **Core Tools** (TOOLS.md) | Always | ~960 | Top 7 most-used tools with routing table |
| **Extended Tools** (TOOLS_EXTENDED.md) | On demand | ~850 | Finance, homeschool, email, web search, image gen |
| **Context Recipe** | Per request | 0-5,000 | Only the memory sections needed for this specific task |

**TESSA & COLE add their own router layer BEFORE the gateway**, so the gateway's 13K token floor only applies to calls that actually reach it. Tier 1 (local) calls never hit the gateway at all.

### Production Lesson: What Gets Loaded Matters

OpenClaw loads ONLY these specific .md files from the workspace root:

| File | Auto-Loaded | Notes |
|------|-------------|-------|
| SOUL.md | Yes | Agent personality |
| TOOLS.md | Yes | Tool instructions. **ALL operational rules must be here** |
| AGENTS.md | Yes | Workspace instructions |
| USER.md | Yes | User profile |
| ASSISTANT_RULES.md | **No** | NOT loaded! Rules here are invisible to the agent |
| CALENDAR.md | **No** | Not loaded |
| Any other .md | **No** | Not loaded |

**Critical lesson:** Rules in ASSISTANT_RULES.md had zero effect. We spent days debugging why agents ignored cardinal rules before discovering this. TESSA & COLE avoid this by having a single, authoritative config loading path.

---

## Deployment: API-First on DigitalOcean

### Recommended Setup

- **DigitalOcean Basic Shared CPU:** 4 GB RAM / 2 vCPU / 80 GB SSD
- **Cost:** ~$24/month USD (can start at 2GB/$12, resize as needed)
- **OS:** Ubuntu 24.04 LTS
- **No Ollama. No GPU. No 32 GB RAM needed.**
- **Runs:** TESSA (FastAPI), FAISS indexer, Signal bot, Caddy, Magic Link sidecar

### Quick Setup

```bash
git clone https://github.com/yourusername/tessa.git
cd tessa
python3.12 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env  # Add your API keys
python scripts/setup.py
# Creates vault, directories, indexes, systemd services
uvicorn server.main:app --host 127.0.0.1 --port 8100
```

### Security-First Setup (Production Grade)

```bash
# 1. Create encrypted vault
dd if=/dev/zero of=/root/tessa_vault.img bs=1M count=2048
LOOP=$(sudo losetup --find)
sudo losetup $LOOP /root/tessa_vault.img
sudo cryptsetup luksFormat $LOOP   # Set passphrase
sudo cryptsetup luksOpen $LOOP tessa_crypt
sudo mkfs.ext4 /dev/mapper/tessa_crypt
sudo mount -o nosuid,nodev /dev/mapper/tessa_crypt /mnt/tessa_vault

# 2. Create data structure inside vault
sudo mkdir -p /mnt/tessa_vault/.tessa/{workspace,agents,knowledge,config}
sudo ln -s /mnt/tessa_vault/.tessa /home/tessa/.tessa

# 3. Install TESSA
git clone https://github.com/yourusername/tessa.git /opt/tessa
cd /opt/tessa && python3.12 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# 4. Configure Caddy (reverse proxy + TLS)
# See Part 4 for full Caddyfile

# 5. Start services
sudo systemctl enable --now tessa tessa-memory tessa-backup.timer
```

---

## OpenClaw Compatibility

### Three Integration Modes

**Mode 1: Router (TESSA & COLE in front of OpenClaw)**
```
Signal/Web -> Caddy -> TESSA Router -> Context Chef -> OpenClaw Gateway
                                     ^ (Tier 1: handled locally, never hits OpenClaw)
```
TESSA & COLE intercept all requests. Tier 1 (lookups, CRM queries, calendar) handled locally, 0 API tokens. Tier 2-3 forwarded to OpenClaw with assembled Minimum Viable Context.

**Mode 2: Parallel (shared memory)**
```
Signal -> OpenClaw (existing setup, unchanged)
Web UI -> TESSA (new interface, same memory files)
Both read/write the same /mnt/vault/.tessa/ directory
```

**Mode 3: Standalone (no OpenClaw)**
```
Signal/Web/API -> Caddy -> TESSA -> LLM APIs directly
```

### Compatibility Guarantees
- Same .md file formats for memory, contacts, journals, skills
- Same directory conventions (workspace/, memory/, skills/, agents/)
- Shared FAISS index (brain.index + metadata.json)
- Shared knowledge graph (nodes.jsonl, edges.jsonl, aliases.json)
- Same CRM format (Mackay 66 .md + _index.json)
- Same JSONL formats (health data, cost logs, decision journal)

---

## Open Source Strategy: AGPL v3

TESSA & COLE are freedom technology. The AGPL guarantees the code stays open forever, even if someone runs it as a cloud service.

What users CAN do: use, modify, distribute freely. Run on their own server. Build on top. Use commercially.

What users MUST do: keep license notices. Share source code of modifications if distributed or offered as a service. License derivative works under AGPL.

---

*Continue to Part 2: Core Systems (Context Chef, Router, Workflows, Agents, R2 Document Intelligence)*
