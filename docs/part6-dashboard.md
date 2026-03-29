# TESSA & COLE - Part 6a of 6
## Dashboard: The Command Center

**Design ethos:** Apple meets Cyberpunk. Modern, elegant, minimalistic. Sovereignty and privacy focused. Clean enough for non-technical users, deep enough for power users who want brain surgery.

---

## The Moat

The dashboard isn't a settings panel. It's a window into your growing AI brain. The more TESSA & COLE know you, your contacts, your decisions, your health, your patterns, the more irreplaceable they become. You don't switch command centers. Everything else is satellite work.

The Memory screen is the centerpiece. It shows your knowledge growing: new synapses forming between entities, relationships deepening, patterns emerging. It's your neural network, and it's yours alone. On your hardware. Under your control.

---

## Design Language: Apple x Cyberpunk x Sovereignty

### Visual Identity
- **Typography:** Inter (system font stack fallback). Large, confident headings. Monospace for data.
- **Colors (Dark, default):**
  - Background: `#0a0a0f` (deep space black)
  - Surface: `#12121a` (card background)
  - Border: `#1e1e2e` (subtle grid lines)
  - Text primary: `#e0e0e8` (off-white, easy on eyes)
  - Text secondary: `#6b6b80` (muted)
  - Accent: `#00d4aa` (teal/cyan, sovereignty green, not corporate blue)
  - Accent warm: `#ff6b35` (alert/cost orange)
  - Accent cool: `#7b68ee` (purple for knowledge/memory)
  - Success: `#00d4aa`
  - Warning: `#ffb347`
  - Error: `#ff4757`
- **Colors (Light):**
  - Background: `#fafafa`
  - Surface: `#ffffff`
  - Border: `#e8e8ed`
  - Text primary: `#1a1a2e`
  - Same accent colors, slightly adjusted for contrast
- **Effects:**
  - Subtle glow on accent elements (box-shadow with accent color at 20% opacity)
  - Glass morphism on cards (backdrop-filter: blur)
  - No gradients. No skeuomorphism. Flat with depth via shadow only.
  - Micro-animations: 200ms ease transitions on hover/focus
- **Icons:** Minimal line icons (Lucide or inline SVG, no icon font dependency)
- **Privacy motif:** Small lock icon next to "Your Data" labels. Subtle but present.

### Layout
- Sidebar navigation (collapsible on mobile)
- Content area: max-width 1200px, centered
- Cards: 16px border-radius, 1px border, subtle shadow
- Grid: CSS Grid, responsive (1-3 columns)
- Spacing: 8px base unit (8, 16, 24, 32, 48)

---

## Screen 1: The Pulse (Home)

Everything at a glance. One screen, zero clicks to understand your system.

```
+-----------------------------------------------------------+
|  TESSA & COLE                       Your Data / Light/Dark |
+--------+--------------------------------------------------+
|        |                                                   |
| Pulse  |  Today's Cost          System Health              |
|        |  +----------+         +--------------+            |
| Costs  |  |  $0.23   |         |  All Clear   |            |
|        |  |  7d trend |         |  FAISS ok    |            |
| Memory |  +----------+         |  Graph ok    |            |
|        |                        |  Vault ok    |            |
| Skills |  Tier Breakdown        +--------------+            |
|        |  +--------+ +--------+ +--------+                  |
| Config |  | FREE   | | LIGHT  | | DEEP   |                  |
|        |  | 47     | | 12     | | 3      |                  |
|        |  | calls  | | calls  | | calls  |                  |
|        |  +--------+ +--------+ +--------+                  |
|        |                                                   |
|        |  Recent Activity                                  |
|        |  +------------------------------------------+     |
|        |  | 2m ago  Alex  "meeting prep Sarah"       |     |
|        |  |         T2 Sonnet / 3,200 tok / $0.04    |     |
|        |  | 15m ago Alex  "glucose levels"            |     |
|        |  |         T1 doc.py / 0 tok / FREE          |     |
|        |  | 1h ago  Jordan "schedule today"            |     |
|        |  |         T1 cal.py / 0 tok / FREE          |     |
|        |  +------------------------------------------+     |
|        |                                                   |
|        |  Memory Growth             Budget                 |
|        |  +--------------+    +--------------+             |
|        |  | 328 vectors  |    | Alex         |             |
|        |  | 43 entities  |    | ========..   |             |
|        |  | 35 relations |    | $0.23/$3.00  |             |
|        |  | 81 contacts  |    |              |             |
|        |  | +12 this week|    | Jordan       |             |
|        |  +--------------+    | ==........   |             |
|        |                      | $0.05/$2.00  |             |
|        |                      +--------------+             |
+--------+--------------------------------------------------+
```

---

## Screen 2: Cost Intelligence

Where money goes. No surprises.

### Components
- **30-day cost chart:** Bar chart, daily totals. Hover for breakdown.
- **Savings calculator:** "Without TESSA & COLE: ~$X. With them: $Y. Saved: $Z (N%)"
  - Calculates by multiplying total calls x 18,000 tokens (full context) vs actual tokens used
- **Breakdown donuts:** By model, by workflow, by person. Click to filter.
- **Budget meters:** Per-person progress bars with daily limits.
- **Top-5 expensive calls:** Table with timestamp, person, workflow, model, tokens, cost
- **Tier distribution:** Pie chart showing what % is Tier 1 (free) vs 2 vs 3

---

## Screen 3: Memory (The Brain)

This is the screen that makes people say "whoa." It shows knowledge growing, the neural network forming around YOU.

### 3a. Knowledge Growth Timeline
A chart showing cumulative knowledge over time:
```
Vectors
   350 |                           /---- 328
       |                      /---/
   250 |                 /---/
       |            /---/
   150 |       /---/
       |  /---/
    50 |//
       +--------------------------------------
        Mar 1    Mar 8    Mar 15   Mar 22   Mar 29

        Entities: 12 -> 43 (+258%)
        Relations: 8 -> 35 (+338%)
        Contacts: 45 -> 81 (+80%)
```

### 3b. Neural Network Visualization
Interactive vis.js graph. Enhanced:
- **Nodes pulse** based on recency (recently touched = brighter glow)
- **Edge thickness** based on interaction frequency
- **Color coding:** People (teal), Ventures (orange), Companies (purple), Locations (green)
- **Click a node** for slide-in panel with entity details, related memories, interaction history
- **"Brain health" metric:** Average relationship depth across all contacts
  - Shallow (just a name): 1/5
  - Basic (company, role): 2/5
  - Connected (3+ interactions logged): 3/5
  - Deep (Mackay 66 filled >50%): 4/5
  - Intimate (weekly contact, personal details): 5/5

### 3c. Memory Search
- Clean search bar at top
- Results show: confidence badge (HIGH/MEDIUM/LOW), file, preview, date
- Filter pills: workspace, topic, date range
- "Ask TESSA" button: sends query through the full pipeline (Tier 2/3)

### 3d. Browse
- Tabs: Contacts | Journal | Decisions | Bookmarks
- Contacts: card grid with relationship health indicators (green/yellow/red dots)
- Journal: timeline view, newest first
- Decisions: table with outcome status (pending/resolved)
- Bookmarks: read/unread badges

### 3e. Memory Stats
- Total vectors, growth rate
- Index freshness (time since last re-index)
- BM25 corpus size
- Knowledge graph: nodes, edges, avg degree
- Autolearning: corrections captured, rules synthesized, last synthesis date
- Dedup stats: duplicates blocked

---

## Screen 4: Skills

What TESSA & COLE can do.

### Skill Registry
- Card per skill: name, description, agent access, last used
- Status indicator: available (green) / error (red) / untested (gray)
- Total calls and cost per skill (from cost ledger)

### Skill Runner (Power User)
- Dropdown: select skill
- Args input field
- "Run" button
- Output panel (monospace, scrollable)
- History of recent runs

---

## Screen 5: Settings

### People
- Table: name, phone, role, daily budget, allowed tiers
- Add/edit person (modal form)
- Budget usage sparkline per person

### Models
- Model ladder visualization (tier 1 -> 2 -> 3, which models in each)
- Current defaults per agent
- Cost per 1M tokens reference
- "Switch model" action

### Context Recipes
- List of .md recipes with parsed view
- Edit in place (with save)

### System
- Service status (TESSA, FAISS memory, gateway if coexisting)
- Restart button
- Backup status (last backup date, size)
- Disk/RAM usage
- Log viewer (tail last 50 lines)

---

## Technical Implementation

### Zero Build Dependencies
```
dashboard/
+-- index.html          # Single file: HTML + embedded CSS + JS
+-- favicon.svg         # Logo (inline SVG)
+-- (nothing else)
```

No npm. No webpack. No React. No node_modules.
One HTML file. Served by FastAPI at `/dashboard`.
Works offline (except API calls). Loads in <100ms.

### Stack
- **HTML5:** semantic elements, no divs-for-everything
- **CSS:** custom properties for theming, Grid layout, media queries
- **JS:** vanilla ES6+, fetch() for API calls, no framework
- **Charts:** Pure SVG generated by JS (sparklines, bar charts, donuts)
- **Graph:** vis.js from CDN (only external dependency, and only for Memory screen)
- **Auth:** passphrase stored in sessionStorage after login

### API Endpoints Used
```
GET  /api/health                      -> Pulse: system status
GET  /api/metrics/today               -> Pulse: today's numbers
GET  /api/metrics/history?days=30     -> Costs: 30-day chart
GET  /api/metrics/breakdown?group_by= -> Costs: donuts
GET  /api/system/health               -> Pulse: FAISS/graph/vault
POST /api/chat                        -> Memory: "Ask TESSA"
POST /api/skills/execute              -> Skills: run a skill
GET  /api/memory/growth               -> Memory: timeline data
GET  /api/memory/search?q=            -> Memory: search results
GET  /api/memory/graph                -> Memory: vis.js data
GET  /api/memory/stats                -> Memory: index health
```

---

## Color Reference

### Dark Mode (Default)
```css
:root {
    --bg-primary: #0a0a0f;
    --bg-surface: #12121a;
    --bg-surface-hover: #1a1a28;
    --border: #1e1e2e;
    --border-accent: #2a2a3e;
    --text-primary: #e0e0e8;
    --text-secondary: #6b6b80;
    --text-muted: #4a4a5e;
    --accent: #00d4aa;
    --accent-glow: rgba(0, 212, 170, 0.2);
    --accent-warm: #ff6b35;
    --accent-cool: #7b68ee;
    --accent-memory: #7b68ee;
    --success: #00d4aa;
    --warning: #ffb347;
    --error: #ff4757;
    --tier-free: #00d4aa;
    --tier-light: #7b68ee;
    --tier-deep: #ff6b35;
    --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
    --font-mono: 'JetBrains Mono', 'Fira Code', monospace;
    --radius: 16px;
    --radius-sm: 8px;
    --shadow: 0 4px 24px rgba(0, 0, 0, 0.4);
    --shadow-glow: 0 0 20px var(--accent-glow);
}
```

### Light Mode
```css
[data-theme="light"] {
    --bg-primary: #f5f5f7;
    --bg-surface: #ffffff;
    --bg-surface-hover: #f0f0f5;
    --border: #e0e0e8;
    --border-accent: #d0d0de;
    --text-primary: #1a1a2e;
    --text-secondary: #6b6b80;
    --text-muted: #9b9bb0;
    --shadow: 0 4px 24px rgba(0, 0, 0, 0.08);
    --shadow-glow: 0 0 20px rgba(0, 212, 170, 0.15);
}
```

---

## The Privacy Message

Subtly woven throughout:
- Footer: "All data on your hardware. Zero cloud storage."
- Memory screen header: "Your brain. Your hardware. Your rules."
- Settings: "TESSA & COLE never phone home. All processing happens here."
- System health: "Encrypted at rest. Encrypted in transit."

Not preachy. Not a manifesto. Just confident, quiet sovereignty.
