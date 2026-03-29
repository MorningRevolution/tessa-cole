# TESSA & COLE - Part 2 of 6
## Core Systems: Context Chef, Router, Workflows, Agents, R2 Document Intelligence

---

## The Context Chef (Core Innovation)

The Context Chef is what makes TESSA & COLE different from everything else. They examine every request and assemble the minimum viable context, the smallest set of memory, knowledge, and instructions the model needs.

### How It Works

```
Input: "I have a meeting with Acme Corp in 30 minutes"

Context Chef:
  1. Classification: meeting_prep workflow (rule-based, 0 tokens)
  2. Entity detected: "Acme Corp" (regex, 0 tokens)
  3. Needed: people/acme-corp.md + last 3 meetings + active commitments
  4. NOT needed: health, financial, journals, other people, tools, skills
  5. Assembled context: ~2,500 tokens
  6. Selected model: Claude Sonnet (synthesis task)
  7. Estimated cost: $0.03

  Without TESSA & COLE: 18,000 tokens ($0.18)
  With TESSA & COLE: 2,500 tokens ($0.03), 86% savings
```

### Three Modes

**Mode 1: Rule-Based (0 tokens).** For known workflow types with predefined recipes. Covers 80%+ of requests.

**Mode 2: AI-Assisted (~300 tokens).** For ambiguous requests. A tiny Flash/Haiku call determines which memory sections are relevant.

**Mode 3: Iterative Retrieval.** For complex queries. Hybrid search (FAISS + BM25) first, graph-boost results, narrow down. This is where the 7-phase memory system shines.

### Context Recipes (.md files)

```markdown
<!-- context-recipes/meeting_prep.md -->
# Meeting Prep Context Recipe

## Required Sections
- people/{entity}.md
- meetings/{entity}/ (last 5 entries)
- commitments/active.md (filtered for {entity})

## Optional Sections
- projects/{entity}.md (if exists)

## Excluded (never load for this workflow)
- health/
- financial/
- journals/
- All tool/skill definitions

## Max Context Tokens: 4000
## Model: tier_2_quality (Sonnet, Gemini Pro, GPT-4o)
## Budget Fallback: tier_2_balanced (Haiku, Flash, GPT-4o-mini)
## Estimated Cost: $0.03-0.05
```

```markdown
<!-- context-recipes/general_chat.md -->
# General Chat Context Recipe

## Required Sections: (none)
## Optional Sections: preferences.md
## Max Context Tokens: 500
## Model: tier_1_cheap (Flash, GPT-4o-mini)
## Estimated Cost: $0.0001
```

```markdown
<!-- context-recipes/health_query.md -->
# Health Query Context Recipe
# Production-learned: health data lives in structured JSONL, not flat .md

## Required Sections: NONE (handled by doc.py script locally)
## Script: doc.py {subcommand} --person {person}
## Tier: 1 (local, zero API tokens)
## Note: doc.py reads JSONL time-series directly; no LLM needed for lookups
```

```markdown
<!-- context-recipes/document_processing.md -->
# Document Processing Context Recipe
# R2 handles all documents. TESSA & COLE just route to R2.

## Required Sections: NONE (R2 manages its own context)
## Script: r2.py process --file {path} --context {description} --model auto
## Model Selection: R2 auto-selects (Flash for short docs, Pro for transcripts)
## Note: R2 cross-extracts to CRM, CT, Doc, Finance. One document, all systems updated.
```

```markdown
<!-- context-recipes/deep_analysis.md -->
# Deep Analysis Context Recipe

## Required Sections: * (all relevant sections via hybrid recall)
## Search: recall.py "{query}" --top 10 --min-confidence medium
## Max Context Tokens: 25000
## Model: tier_3_deep (Opus, Gemini Pro)
## Estimated Cost: $0.30-0.50
## Note: This is where full context is justified
```

### Diversity-Aware Context Selection

When loading context, TESSA & COLE avoid redundant chunks. If 5 memory entries say the same thing, load the best 2, not all 5.

```python
def select_diverse_context(self, chunks: list[dict], max_tokens: int,
                           diversity_lambda: float = 0.5) -> list[dict]:
    """Select diverse, non-redundant context chunks.

    Uses Maximal Marginal Relevance (MMR) to balance relevance
    with diversity. Production-learned: without this, 5 CRM contact
    cards about the same person can fill the entire context budget.
    """
    selected = []
    remaining = list(chunks)

    while remaining and self.total_tokens(selected) < max_tokens:
        best_chunk = None
        best_score = -1

        for chunk in remaining:
            relevance = chunk["score"]
            redundancy = max(
                (self.cosine_similarity(chunk, s) for s in selected),
                default=0
            )
            mmr_score = diversity_lambda * relevance - (1 - diversity_lambda) * redundancy

            if mmr_score > best_score:
                best_score = mmr_score
                best_chunk = chunk

        if best_chunk:
            selected.append(best_chunk)
            remaining.remove(best_chunk)

    return selected
```

### Per-Chunk Relevance Scoring with Confidence Labels

Each context chunk gets a relevance score and confidence label. After the LLM responds, TESSA & COLE check which sections were actually referenced vs loaded but unused. Unused sections get lower priority next time (auto-learning).

```python
# Production confidence thresholds (calibrated against text-embedding-3-small)
CONFIDENCE_THRESHOLDS = {
    "high":   0.6,   # Strong semantic match
    "medium": 0.4,   # Probable relevance
    "low":    0.25,  # Possible relevance
    "noise":  0.0,   # Below useful threshold
}

def confidence_label(score: float) -> str:
    """Assign confidence label to a search result."""
    if score >= 0.6: return "HIGH"
    if score >= 0.4: return "MEDIUM"
    if score >= 0.25: return "LOW"
    return "NOISE"
```

### Context Chef Implementation

```python
class ContextChef:
    """Assembles minimum viable context for each request.

    Production-learned patterns:
    - 80%+ of requests match rule-based recipes (Mode 1, 0 tokens)
    - Script-routed queries (health, CRM, CT) use 0 API tokens
    - Only ambiguous requests need AI-assisted classification (Mode 2, ~300 tokens)
    - Deep analysis is rare (< 5% of requests) but justified when used
    """

    def __init__(self, recipes_dir: str, memory_dir: str):
        self.recipes = self.load_recipes(recipes_dir)
        self.memory = MemoryReader(memory_dir)
        self.recall = HybridRecall(memory_dir)  # FAISS + BM25 + Graph
        self.model_picker = ModelPicker()
        self.cost_guard = CostGuard()

    def prepare(self, request: str, person: str,
                workflow: str | None = None) -> ContextPackage:
        recipe = self.recipes.get(workflow)

        if recipe and recipe.script:
            # Tier 1: Script-routed (0 tokens), health, CRM, CT, calendar
            return ContextPackage(
                script=recipe.script,
                args=self.resolve_args(recipe, request),
                model=None,  # No LLM needed
                tier=1,
                estimated_tokens=0,
                estimated_cost=0.0
            )

        if recipe:
            # Tier 2: Rule-based recipe with targeted context
            sections = self.resolve_sections(recipe.required, request, person)
            memory = self.memory.load_sections(sections, person, recipe.max_tokens)
            memory = self.select_diverse_context(memory, recipe.max_tokens)
            model = self.model_picker.select(recipe.model_tier)

            return ContextPackage(
                system_prompt=self.get_workflow_prompt(workflow),
                memory_context=memory,
                model=model,
                tier=2,
                estimated_tokens=self.estimate_tokens(memory),
                estimated_cost=self.estimate_cost(model, memory)
            )

        # Mode 2: AI-assisted classification (~300 tokens)
        plan = self.ai_plan_context(request)
        if plan.script:
            return self.prepare(request, person, plan.workflow)

        # Tier 3: Deep analysis with hybrid recall
        recall_results = self.recall.search(
            request, top_k=10, min_confidence="medium",
            use_hybrid=True, use_graph=True
        )
        memory = self.select_diverse_context(recall_results, plan.max_tokens)
        model = self.model_picker.select(plan.model_tier)

        return ContextPackage(
            system_prompt=self.get_workflow_prompt(plan.workflow),
            memory_context=memory,
            model=model,
            tier=3,
            estimated_tokens=self.estimate_tokens(memory),
            estimated_cost=self.estimate_cost(model, memory)
        )
```

---

## The Router

Every request hits the router first. Rule-based classification (0 tokens) for 80%+ of requests, tiny LLM fallback (~250 tokens) for ambiguous cases.

### Query Routing Table (Production, Single Source of Truth)

**This is the most important table in TESSA & COLE.** It determines which tool handles each query type. Production-learned: without this, the gateway's built-in `memory_search` handles everything, but memory_search only searches flat .md files. It misses structured JSONL data (508 lab entries), CRM index, todo database.

```python
# TESSA & COLE Query Routing Table
# Rule: ALWAYS use the specific script. NEVER fall back to generic memory_search.
QUERY_ROUTING = {
    # Health queries → doc.py (reads structured JSONL time-series)
    "health":    {"script": "doc.py",  "patterns": [
        r"health|glucose|labs?|A1C|blood|pressure|BP|weight|sleep|exercise",
        r"prescription|medication|vitamin|supplement|medical|doctor|test results",
    ]},

    # Contact queries → crm.py (reads _index.json + Mackay 66 .md files)
    "contacts":  {"script": "crm.py", "patterns": [
        r"who is|contact|email|phone|birthday|remind me about \w+",
        r"relationship|follow.?up|how.+doing with",
    ]},

    # Task/venture queries → ct.py (reads CT.json structured database)
    "tasks":     {"script": "ct.py",  "patterns": [
        r"todo|task|venture|project|deadline|overdue|action item",
        r"meeting log|transcript|resume",
    ]},

    # Document processing → r2.py (AI-powered cross-extraction)
    "documents": {"script": "r2.py",  "patterns": [
        r"process|analyze|summarize.*document|transcript|PDF|image",
        r"upload|file|invoice|contract|report",
    ]},

    # Calendar queries → cal.py (OAuth, reads Google Calendar directly)
    "calendar":  {"script": "cal.py", "patterns": [
        r"schedule|calendar|meeting|appointment|free time|busy",
        r"what.+(today|tomorrow|this week|next week)",
    ]},

    # Finance queries → finance.py
    "finance":   {"script": "finance.py", "patterns": [
        r"financial|spending|budget|account|balance|due date|payment",
    ]},

    # General knowledge → recall.py (FAISS + BM25 + Graph hybrid)
    "knowledge": {"script": "recall.py", "patterns": [
        r"remember|recall|what did|when did|search.*memory",
    ]},

    # General chat → LLM directly (cheapest model, minimal context)
    "chat":      {"script": None, "model_tier": "tier_1_cheap", "patterns": [
        r"^(hi|hello|hey|good morning|thanks?)",
        r"tell me a joke|how are you",
    ]},
}
```

### Classification Patterns

```python
import re
from typing import Optional

class Router:
    """Classifies requests and routes to the right handler.

    Production-learned: regex classification handles 80%+ of requests
    at 0 token cost. LLM fallback only for genuinely ambiguous cases.
    """

    def classify(self, message: str, person: str) -> Classification:
        message_lower = message.lower()

        # Step 1: Rule-based classification (0 tokens)
        for category, config in QUERY_ROUTING.items():
            for pattern in config["patterns"]:
                if re.search(pattern, message_lower):
                    return Classification(
                        category=category,
                        workflow=category,
                        script=config.get("script"),
                        model_tier=config.get("model_tier"),
                        entities=self.extract_entities(message),
                        confidence="rule_match",
                        tokens_used=0
                    )

        # Step 2: Check for document attachments
        if self.has_attachment(message):
            return Classification(
                category="documents",
                workflow="document_processing",
                script="r2.py",
                confidence="attachment_detected",
                tokens_used=0
            )

        # Step 3: LLM fallback (~250 tokens, cheapest model)
        return self.llm_classify(message, person)

    def llm_classify(self, message: str, person: str) -> Classification:
        """Fallback: use cheapest model for classification."""
        prompt = f"""Classify into: chat, health, contacts, tasks, documents,
        calendar, finance, knowledge. Extract entity names.
        Request: "{message}" Reply JSON only: {{"category":"...","entities":["..."]}}"""

        response = self.llm.call("google/gemini-2.5-flash", prompt, max_tokens=100)
        # ~200 input + ~50 output = ~250 total tokens
        result = json.loads(response.text)

        return Classification(
            category=result["category"],
            workflow=result["category"],
            script=QUERY_ROUTING[result["category"]].get("script"),
            entities=result.get("entities", []),
            confidence="llm_classify",
            tokens_used=250
        )

    def extract_entities(self, message: str) -> list[str]:
        """Extract entity names from message using regex."""
        # Capitalized multi-word names: "Sarah Chen", "Acme Corp Inc"
        names = re.findall(r'\b([A-Z][a-z]+ [A-Z][a-z]+(?:\s[A-Z][a-z]+)?)\b', message)
        return list(set(names))
```

### Tier Assignment

```python
TIER_MAP = {
    # Tier 1: Local script execution (0 API tokens)
    "health": 1, "contacts": 1, "tasks": 1, "calendar": 1, "finance": 1,

    # Tier 2: Light API call (1-4K tokens)
    "chat": 2, "knowledge": 2,

    # Tier 3: Deep API call (5-25K tokens)
    "documents": 3,  # R2 manages its own model selection
}
```

---

## The PRE-RESPONSE GATE (Anti-Hallucination)

**Production origin:** Our main agent repeatedly said "I've updated the records" and "I've saved the contact" without running any commands. LLM-generated text that sounds like execution but isn't. The PRE-RESPONSE GATE is a mechanical check that prevents this.

### The Four Gates

```python
class PreResponseGate:
    """Mechanical anti-hallucination checks.

    Run these BEFORE every response. If any gate fails, do NOT send
    the response. Fix the issue first.

    Production data: reduced hallucinated execution from ~15% to <1%.
    """

    def check_all(self, request: Request, response: Response) -> GateResult:
        # GATE 1: DID I EXECUTE?
        if response.claims_action and not response.has_tool_calls:
            return GateResult(
                passed=False,
                gate="execution",
                message="Response claims action but no tool was called. "
                        "Run the command before responding."
            )

        # GATE 2: RIGHT TOOL?
        if response.has_tool_calls:
            for call in response.tool_calls:
                expected = QUERY_ROUTING.get(request.category, {}).get("script")
                if expected and call.script != expected:
                    return GateResult(
                        passed=False,
                        gate="tool_selection",
                        message=f"Used {call.script} but should use {expected} "
                                f"for {request.category} queries."
                    )

        # GATE 3: EXACT DATA?
        # Copy-paste numbers/names from user message, never retype from memory
        for number in self.extract_numbers(request.message):
            if number not in response.text:
                return GateResult(
                    passed=False,
                    gate="data_accuracy",
                    message=f"User mentioned {number} but response has different value. "
                            "Copy-paste exact data."
                )

        # GATE 4: SHELL SAFE?
        # Dollar amounts must use single quotes in shell commands
        for match in re.finditer(r'"\$[\d,]+\.?\d*"', response.text):
            return GateResult(
                passed=False,
                gate="shell_safety",
                message=f"Dollar amount {match.group()} in double quotes. "
                        "Shell will interpolate $var. Use single quotes."
            )

        return GateResult(passed=True)
```

### Production Examples of Gate Catches

| Gate | What Happened | What Gate Caught |
|------|--------------|-----------------|
| 1 | "I've updated Sarah's contact with her new email" | No `crm.py update` call was made |
| 2 | Used `doc-processor` to analyze insurance document | Should have used `r2.py` (R2 routes documents) |
| 3 | User said "$282.50", agent wrote "$82.50" | Dollar sign consumed by shell interpolation |
| 4 | `echo "Payment: $282.50"` in bash | `$2` interpolated as empty string |

---

## R2 Document Intelligence (Production Flagship)

R2 is the universal document intelligence pipeline. Every document (PDFs, images, transcripts, pasted emails, chat conversations) goes through R2. R2 reads, classifies, extracts, and routes data to the right systems.

### Why R2 Exists

**Production problem:** Without R2, document processing was fragmented:
- User uploads transcript, agent tries to handle it inline, misses contacts, Mackay 66 intel, action items
- User pastes chat conversation, agent treats it as simple text, loses CRM data
- User shares PDF, agent uses wrong tool (doc-processor instead of R2)

**R2 solution:** One pipeline that cross-extracts EVERYTHING:

```
Document → R2 Analysis (Gemini Flash/Pro) → Structured JSON
                                                  ↓
                              ┌────────────────────┼────────────────────┐
                              │                    │                    │
                         CRM (crm.py)        CT (ct.py)          Doc (doc.py)
                         - Create contacts   - Create todos      - Store labs
                         - Mackay 66 enrich  - Log meetings      - Store vitals
                         - Log interactions  - Route to venture   - Flag alerts
                              │                    │                    │
                         Graph Builder       Finance (finance.py) Calendar flags
                         - Extract entities  - Extract amounts    - Extract dates
                         - Build relations   - Track due dates
```

### R2 Routing Table (Single Source of Truth)

```python
# Hardcoded in r2.py. This IS the routing table.
ROUTING_TABLE = {
    # Ventures → workspace-projects/memory/{venture}/
    "acme-corp":          "workspace-projects/memory/acme-corp/",
    "side-project-alpha": "workspace-projects/memory/side-project-alpha/",
    "rental-properties":  "workspace-projects/memory/rental-properties/",
    "consulting-co":      "workspace-projects/memory/consulting-co/",

    # Personal → specific workspaces
    "health":     "workspace-health/memory/docs/",       # R2 files doc, then calls doc.py
    "journal":    "workspace/memory/journal/",
    "finance":    "workspace/memory/finances/",
    "coaching":   "workspace-projects/memory/coaching/",

    # Family → family workspace
    "family":     "workspace-family/memory/family/",
    "homeschool": "workspace-family/memory/",

    # Specialized
    "insurance":  "workspace-projects/memory/rental-properties/properties/",
    "genealogy":  "workspace-genealogy/memory/",

    # Fallback. R2 asks user where to refile.
    "other":      "workspace-inbox/memory/",
}
```

### R2 Extraction Schema

```python
# R2 sends this to Gemini Flash/Pro for every document
R2_EXTRACTION_PROMPT = """Analyze this document and return JSON:
{
    "document_type": "transcript|invoice|medical|insurance|journal|...",
    "venture_slug": "acme-corp|side-project-alpha|...|null",
    "title": "Brief descriptive title",
    "date": "YYYY-MM-DD (extracted from content)",
    "attendees": ["Name 1", "Name 2"],
    "people": [
        {
            "name": "Full Name",
            "company": "...",
            "role": "...",
            "email": "...",
            "mackay66": {
                "marital_status": "...",
                "children": "...",
                "hobbies": "...",
                "education": "..."
            }
        }
    ],
    "action_items": [
        {"task": "...", "assignee": "...", "due": "YYYY-MM-DD", "priority": "high|medium|low"}
    ],
    "followups": [
        {"person": "...", "action": "...", "due": "YYYY-MM-DD"}
    ],
    "decisions": ["Decision 1", "Decision 2"],
    "health_data": [{"person": "...", "type": "lab|bp|weight", "value": "...", "unit": "..."}],
    "financial_data": [{"type": "invoice|payment|due", "amount": "...", "date": "..."}],
    "calendar_events": [{"title": "...", "date": "YYYY-MM-DD", "time": "HH:MM"}],
    "incomplete": ["CONFIRM: Is X the same person as Y?", "NEEDS ROUTING: ..."]
}"""
```

### R2 Execution Pipeline

```python
def execute_analysis(analysis: dict, filepath: str) -> str:
    """Execute all cross-extractions from R2 analysis.

    Production-learned execution order matters:
    1. Route document to correct workspace
    2. Create/update CRM contacts (need contact IDs for todo assignment)
    3. Create todos in CT (references contact names)
    4. Create follow-ups (references contact names)
    5. Health data → Doc
    6. Finance data → Finance tracker
    7. Calendar flags
    8. Update knowledge graph
    """
    results = []

    # Step 1: Route document
    doc_type = analysis.get("document_type", "other")
    venture = analysis.get("venture_slug")
    dest = resolve_destination(doc_type, venture)
    save_document(filepath, dest, analysis["title"], analysis["date"])
    results.append(f"Filed to {dest}")

    # Step 2: CRM contacts + Mackay 66 enrichment
    for person in analysis.get("people", []):
        name = person["name"]

        # Known aliases check (avoid confirmation prompts for known people)
        canonical = KNOWN_ALIASES.get(name.lower())
        if canonical:
            name = canonical

        # Create or update contact
        existing = run_script("crm.py", "find", "--name", name)
        if existing:
            # Enrich with Mackay 66 intel
            for key, value in person.get("mackay66", {}).items():
                if value:
                    run_script("crm.py", "update", "--name", name,
                              "--key", key, "--value", value)
        else:
            run_script("crm.py", "add", "--name", name,
                      "--company", person.get("company", ""),
                      "--role", person.get("role", ""))

        # Log interaction
        run_script("crm.py", "log", "--name", name,
                  "--note", f"Mentioned in {analysis['title']}")

        results.append(f"CRM: {name}")

    # Step 3: Todos with dedup (60% word overlap check)
    for item in analysis.get("action_items", []):
        # Check for duplicate before creating
        existing_todos = run_script("ct.py", "todo", "--venture", venture or "")
        if not word_overlap_exceeds(item["task"], existing_todos, threshold=0.6):
            run_script("ct.py", "add-todo",
                      "--venture", venture or "general",
                      "--task", item["task"],
                      "--due", item.get("due", ""),
                      "--priority", item.get("priority", "medium"),
                      "--person", item.get("assignee", ""))
            results.append(f"Todo: {item['task'][:50]}...")

    # Step 4: Update knowledge graph
    if analysis.get("people"):
        graph_data = json.dumps({
            "people": analysis["people"],
            "ventures": [venture] if venture else [],
            "source": filepath
        })
        run_script("graph_builder.py", "--from-r2", graph_data)
        results.append(f"Graph: {len(analysis['people'])} entities")

    # Step 5-7: Health, Finance, Calendar (similar pattern)
    for health in analysis.get("health_data", []):
        run_script("doc.py", "labs", "--person", health["person"],
                  "--test", health["type"], "--value", health["value"],
                  "--unit", health.get("unit", ""))

    return "\n".join(results)
```

### R2 Known Aliases (Avoids Confirmation Loops)

```python
# Production-learned: without aliases, R2 asks "Is Alex Smith the same
# person as Alex James Smith?" every single time
KNOWN_ALIASES = {
    "alex smith": "Alex James Smith",
    "alex": "Alex James Smith",
    "jordan": "Jordan Lee",
}
```

### R2 Questions (When Unsure)

R2 asks questions instead of guessing. The orchestrating agent relays these to the user:

```python
INCOMPLETE_TEMPLATES = {
    "new_venture":   "NEW VENTURE: {name} is not in the system. Should I create it?",
    "which_venture": "WHICH VENTURE? Known: consulting-co, acme-corp, side-project-alpha, "
                     "rental-properties",
    "confirm_match": "CONFIRM: Is {name1} the same person as {name2}?",
    "need_name":     "Need full name for: {partial_name}",
    "needs_routing": "NEEDS ROUTING: Tell me where this should go",
}
```

---

## Workflow Engine

Workflows are .md files, a pipeline of steps, some local (0 tokens), some API (targeted tokens). Independent steps run in parallel via DAG scheduling.

### Transcript Processing Workflow (Production)

```markdown
<!-- workflows/transcript_processing.md -->
# Transcript Processing

## Triggers: "process transcript", "summarize meeting", document attachment

## Step 1: Receive Document (Local, 0 tokens)
Accept file path or pasted text. Detect format (PDF/image/text).

## Step 2: OCR if Needed (Local, 0 tokens)
If PDF/image: run doc-processor OCR (tesseract). Eng+Spa support.

## Step 3: R2 Analysis (API, Smart Model Selection)
Script: r2.py process --file {path} --context "{description}" --model auto
Model: Auto-selected (Flash for short docs, Pro for long transcripts)
R2 returns structured JSON with all cross-extracted data.

## Step 4: Execute Extractions (Local, 0 tokens)
R2 orchestrates: crm.py, ct.py, doc.py, graph_builder.py, finance.py
Each extraction is a subprocess call, no additional LLM tokens.

## Step 5: Relay Results Verbatim (Local, 0 tokens)
Send R2's full output to user. NEVER paraphrase.

## Estimated Total: 2,000-8,000 tokens (depends on document length)
## Cost: $0.01-0.10 (Flash) or $0.05-0.50 (Pro)
```

### Meeting Prep Workflow

```markdown
<!-- workflows/meeting_prep.md -->
# Meeting Prep

## Triggers: "meeting with {person}", "brief me on {person}"

## Step 1: Search CRM (Local, 0 tokens)
Script: crm.py find --name {person}
Returns: contact card with Mackay 66 data, recent interactions, follow-ups

## Step 2: Search Meetings (Local, 0 tokens)
Script: ct.py todo --person {person}
Returns: open todos involving this person

## Step 3: Recall Context (Local, 0 tokens)
Script: recall.py "{person}" --top 5 --min-confidence medium
Returns: relevant memories with confidence labels

## Step 4: Generate Brief (API, Tier 2)
Model: Sonnet or Gemini Pro | Max tokens: 3000
Context: CRM card + todos + recall results (assembled by Context Chef)
Prompt: Prepare brief with profile, recent interactions, open action
        items, unfulfilled promises, personal details, talking points.

## Estimated Total: 3,000-4,000 tokens ($0.03-0.05)
```

### DAG-Based Parallel Execution

```python
class WorkflowEngine:
    """Execute workflows with DAG-based parallelism.

    Steps declare dependencies. Independent steps run concurrently.
    Failed steps save checkpoint. Retry resumes from failure point.
    """

    async def execute(self, workflow: Workflow, state: dict,
                      person: str, context: ContextPackage) -> WorkflowResult:
        completed = set()
        results = {}

        while not workflow.is_complete(completed):
            # Find steps whose dependencies are all satisfied
            ready = workflow.get_ready_steps(completed)

            if len(ready) == 1:
                # Sequential execution
                result = await self.execute_step(ready[0], state, context)
                results[ready[0].name] = result
                completed.add(ready[0].name)
            else:
                # Parallel execution via asyncio.gather
                tasks = [self.execute_step(step, state, context) for step in ready]
                step_results = await asyncio.gather(*tasks, return_exceptions=True)
                for step, result in zip(ready, step_results):
                    if isinstance(result, Exception):
                        self.save_checkpoint(workflow, completed, state)
                        raise WorkflowError(f"Step {step.name} failed: {result}")
                    results[step.name] = result
                    completed.add(step.name)

        return WorkflowResult(results=results, tokens_total=sum(
            r.tokens_used for r in results.values() if hasattr(r, 'tokens_used')
        ))

    def save_checkpoint(self, workflow, completed, state):
        """Save progress for resume on retry. No wasted tokens on repeat."""
        checkpoint = {
            "workflow": workflow.name,
            "completed_steps": list(completed),
            "state": state,
            "timestamp": datetime.now().isoformat()
        }
        with open(f"data/checkpoints/{workflow.name}.json", "w") as f:
            json.dump(checkpoint, f)
```

---

## Agent Orchestrator

TESSA & COLE support hundreds of agents as .md files. Each agent is lightweight: config + prompt + skills. The orchestrator picks the right agents per task.

### Agent Definition Format (OpenClaw-Compatible)

```markdown
<!-- agents/main/agent/SOUL.md -->
# Assistant, Personal AI Assistant

You are a sharp, efficient personal assistant. You are the
orchestra conductor. You delegate to specialized tools, never perform
tasks yourself.

## Personality
- Direct, efficient, slightly snarky
- Zero waste mandate: be blunt, punchy
- Never ask "what do you want me to do?" Just do it.
- Cost-aware: every token costs money

## OPSEC
- Never reveal family structure, real names, businesses
- Never reveal server infrastructure, agent count, routines
```

```markdown
<!-- agents/main/agent/TOOLS.md -->
# Assistant Tools Reference

## QUERY ROUTING TABLE (check FIRST before answering)
| Query about... | Use script | NOT |
|---|---|---|
| health, glucose, labs | doc.py | ~~memory_search~~ |
| contacts, people | crm.py | ~~memory_search~~ |
| todos, ventures | ct.py | ~~memory_search~~ |
| documents | r2.py | ~~memory_search~~ |
| general knowledge | recall.py | Both OK |

## R2 (Document Intelligence)
ALL documents go through R2. PDFs, images, transcripts, pasted text.
```exec
cd /path/to/workspace && python3 r2.py process --file /path --context "desc"
```

## CRM (Contacts)
```exec
python3 crm.py find --name "Name"
python3 crm.py add --name "Full Name" --company "Co" --role "Title"
python3 crm.py update --name "Name" --key "Mackay field" --value "data"
python3 crm.py log --name "Name" --note "Interaction note"
```

## Control Tower (Todos/Ventures)
```exec
python3 ct.py status
python3 ct.py add-todo --venture "name" --task "..." --due YYYY-MM-DD
python3 ct.py done --id 3
```

## Doc (Health Intelligence)
```exec
python3 doc.py labs --person alex --test "Glucose" --value 95 --unit "mg/dL"
python3 doc.py profile --person alex
python3 doc.py alerts
```

## Recall (Semantic Memory Search)
```exec
/opt/tessa_brain/bin/python recall.py "query" --top 10 --min-confidence medium
```
```

### Agent Scaling Pattern

Agents are .md files. Loading an agent config costs 0 tokens (file read). You can define 500 agents. TESSA & COLE only load the one(s) needed per request.

```python
# Production: 9 agents, each with specific roles and access controls
AGENT_REGISTRY = {
    "main":        {"model": "gemini-2.5-flash", "serves": "admin",    "access": "all"},
    "family-a":    {"model": "gpt-4o-mini",      "serves": "spouse",   "access": "family"},
    "family-b":    {"model": "gpt-4o-mini",      "serves": "relative", "access": "personal"},
    "genealogy":   {"model": "gemini-2.5-pro",   "serves": "parent",   "access": "genealogy"},
    "social":      {"model": "gpt-4o-mini",      "serves": "internal", "access": "social"},
    "homeschool":  {"model": "gpt-4o-mini",      "serves": "internal", "access": "family"},
    "ct":          {"model": "gpt-4o-mini",      "serves": "internal", "access": "ventures"},
    "r2":          {"model": "gemini-2.5-flash",  "serves": "internal", "access": "all"},
    "doc":         {"model": "gpt-5-mini",        "serves": "internal", "access": "health"},
}
```

### Orchestra Model

```
TESSA & COLE = the conductors (decide what to play)
Agents       = the musicians (each plays their part)
Skills       = the instruments (tools agents can use)
Workflows    = the sheet music (the sequence of steps)
Context Recipes = the setlist (what context to include)
R2           = the arranger (takes raw documents, extracts everything)
JKE          = the rehearsal notes (what we learned from past performances)
```

---

## Signal Integration

### One Config Change

```bash
# Signal bot config:
# FROM: API_ENDPOINT=http://localhost:YOUR_GATEWAY_PORT   (OpenClaw direct)
# TO:   API_ENDPOINT=http://localhost:8100/api/signal  (TESSA & COLE router)
```

### Signal Handler

```python
@app.post("/api/signal")
async def handle_signal(request: SignalMessage):
    person = identify_person(request.sender)
    classification = router.classify(request.message, person)

    # Budget check
    if not cost_guard.allow(classification, person):
        return SignalResponse("Daily budget reached. Reply OVERRIDE to proceed.")

    # PRE-RESPONSE GATE: check for script-routed queries first
    if classification.script:
        # Tier 1: Execute locally, 0 API tokens
        result = execute_script(classification.script, request.message, person)
        return SignalResponse(result)

    # Tier 2-3: Assemble context and call LLM
    context = context_chef.prepare(request.message, person, classification.workflow)
    result = await workflow_engine.execute(
        classification.workflow,
        {"message": request.message},
        person, context
    )

    # Log cost
    cost_guard.log(result)

    # Cron relay rule: script output is ALWAYS verbatim, never paraphrased
    if classification.is_cron:
        return SignalResponse(result.raw_output)

    return SignalResponse(result.output)
```

### Multi-Person Routing

```python
# Production: per-person budgets, model tiers, language, workflow access
PEOPLE_CONFIG = {
    "admin": {
        "signal": "+1XXXXXXXXXX",
        "partition": "admin",
        "role": "admin",
        "workflows": "all",
        "budget_daily": 3.00,
        "model_tier": "all",
        "language": "en",
    },
    "spouse": {
        "signal": "+1XXXXXXXXXX",
        "partition": "spouse",
        "role": "family",
        "workflows": ["chat", "health", "contacts", "calendar", "homeschool"],
        "budget_daily": 2.00,
        "model_tier": ["tier_1_cheap", "tier_2_balanced"],
        "language": "en",
    },
    "parent": {
        "signal": "+1XXXXXXXXXX",
        "partition": "parent",
        "role": "family",
        "workflows": ["chat", "knowledge", "genealogy"],
        "budget_daily": 1.00,
        "model_tier": ["tier_1_cheap", "tier_2_balanced"],
        "language": "es",
    },
}
```

---

## Cron Architecture (Production-Learned)

### Pattern: Scripts, Not LLM Orchestration

**Production failure:** Original cron jobs told the LLM to orchestrate multi-step API calls. Models narrated instead of executing ("I would run the heartbeat script..." instead of actually running it).

**Fix:** Cron payloads tell the agent to run a **single Python script** and relay stdout. The script does all the work internally.

```python
# Cron payload example (production):
CRON_PAYLOAD = {
    "message": (
        "Run this command and send ONLY its stdout, copy-paste VERBATIM. "
        "Do NOT add any summary, explanation, or formatting:\n"
        "cd /path/to/workspace && python3 skills/morning-report/scripts/compile.py"
    ),
    "schedule": "0 11 * * *",  # 6 AM ET (UTC-5)
    "agent": "main"
}

# The script (compile.py) handles ALL complexity internally:
# - Calls cal.py, ct.py, relationship_health.py, doc.py, etc.
# - Each with subprocess.run() and 15-30s timeouts
# - Assembles 13-section report
# - Returns formatted text to stdout
# - Agent relays VERBATIM
```

### Production Cron Schedule

| Cron | Schedule | Script | Agent |
|------|----------|--------|-------|
| Morning Report | 6:00 AM ET daily | compile.py | main |
| Heartbeat (social media) | Every 2h | heartbeat.py | social via main |
| Email Monitor | Hourly (10AM-11PM, 12AM-3AM UTC) | check_inbox.py | main |
| Log Condensation | Sun 3:00 AM ET | condense_logs.py | main |
| JKE Synthesis | Sun 3:33 AM ET | synthesize.py | system |
| JKE Engagement | Sun 3:33 AM ET (after synthesis) | synthesize_engagement.py | system |
| JKE Expiry | 1st of month 4:13 AM ET | synthesize_expiry.py | system |
| Magic Link Cleanup | Daily 4:00 AM UTC | cleanup.py | system |
| Backup | Daily 5:00 AM UTC | backup.py | system |

---

*Continue to Part 3: Knowledge & Memory System (7-Phase Architecture)*
