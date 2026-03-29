# TESSA & COLE - Part 4 of 6
## Skills, Security & Operations

---

## Skill Architecture

Every skill in TESSA & COLE is a Python script with an argparse CLI. Skills are invoked by agents via subprocess calls, with no shared state, no import dependencies, and no framework coupling. This is deliberate: scripts are testable, debuggable, and replaceable independently.

### Skill Registry Pattern

```python
# Each skill has a _manifest.json for auto-discovery
# skill_registry.py scans all skill dirs for manifests

# workspace/skills/crm/_manifest.json
{
    "name": "crm",
    "description": "Mackay 66 relationship manager",
    "commands": ["add", "update", "find", "log", "followup"],
    "agents": ["main", "ct", "jordan"],
    "script": "scripts/crm.py",
    "dependencies": ["none"],
    "data_format": "markdown + _index.json"
}

# Discovery: skill_registry.py --scan
# Validation: skill_registry.py --validate
# Per-agent: skill_registry.py --agent main
```

### Skill Directory Convention

```
workspace/skills/
├── memory-manager/
│   ├── _manifest.json
│   └── scripts/
│       ├── recall.py           # Hybrid search (FAISS+BM25+Graph)
│       ├── remember.py         # Dedup-aware storage
│       ├── graph_builder.py    # Knowledge graph construction
│       ├── graph_query.py      # Graph traversal + HTML export
│       ├── decisions.py        # Decision journal
│       ├── conflict_resolver.py # Memory conflict detection
│       ├── trends.py           # Topic frequency analysis
│       ├── bookmarks.py        # Smart read-later queue
│       ├── proactive_recall.py # Forgotten connections
│       └── status.py           # Memory system health
├── crm/
│   ├── _manifest.json
│   └── scripts/
│       ├── crm.py              # Contact CRUD + Mackay 66
│       └── relationship_health.py # Health scoring
├── control-tower/
│   ├── _manifest.json
│   └── scripts/
│       └── ct.py               # Ventures, todos, meetings, resume
├── r2/
│   ├── _manifest.json
│   └── scripts/
│       └── r2.py               # Document intelligence + cross-extraction
├── doc/
│   ├── _manifest.json
│   └── scripts/
│       └── doc.py              # Health intelligence (JSONL time-series)
├── morning-report/
│   ├── _manifest.json
│   └── scripts/
│       └── compile.py          # 13-section aggregator
├── finance-tracker/
│   ├── _manifest.json
│   └── scripts/
│       └── finance.py          # Accounts, balances, transactions
├── google-calendar/
│   ├── _manifest.json
│   └── scripts/
│       └── cal.py              # OAuth + Calendar CRUD
├── brave-search/
│   ├── _manifest.json
│   └── scripts/
│       ├── search.js           # Web search
│       └── content.js          # Full page content extraction
├── doc-processor/
│   ├── _manifest.json
│   └── scripts/
│       └── process.py          # PDF/image OCR to markdown
├── email-monitor/
│   ├── _manifest.json
│   └── scripts/
│       └── check_inbox.py      # Gmail API triage
├── security-watchdog/
│   ├── _manifest.json
│   └── scripts/
│       └── health_check.py         # 20+ automated security checks
├── model-switch/
│   ├── _manifest.json
│   └── scripts/
│       └── switch_model.py     # Per-agent model switching
├── transcript-processor/
│   ├── _manifest.json
│   └── scripts/
│       └── transcript.py       # Route transcripts + extraction checklists
├── log-condenser/
│   ├── _manifest.json
│   └── scripts/
│       └── condense_logs.py    # Daily->weekly->monthly log condensation
├── jke-synthesizer/
│   ├── _manifest.json
│   └── scripts/
│       ├── synthesize.py       # Weekly knowledge synthesis
│       ├── synthesize_engagement.py  # Engagement pattern analysis
│       └── synthesize_expiry.py      # Archive old signals
├── magic-link/
│   ├── _manifest.json
│   └── scripts/
│       ├── magic_link.py       # Client-side link management
│       └── magic_link_server.py # Sidecar HTTP+WebSocket server
└── skill_registry.py           # Auto-discovers _manifest.json files
```

---

## Production Skill Deep-Dives

### CRM (Mackay 66 Relationship Intelligence)

```python
# crm.py, a production CRM with 12 contact archetypes

# Base template (every contact gets this)
BASE_TEMPLATE = """# {name}

## Quick Reference
- **Company:** {company}
- **Role:** {role}
- **Tags:** {tags}
- **Ventures:** {ventures}
- **Email:** {email}
- **Phone:** {phone}
- **LinkedIn:** {linkedin}
- **Location:** {location}
- **Birthday:** {birthday}
- **Source:** {source}

## Interaction Log

## Notes
"""

# Archetype sections (injected after tagging)
ARCHETYPES = {
    "speaker":   "## Speaker Profile\n- Topics:\n- Events:\n- Fee:\n- Agent:",
    "cofounder": "## Cofounder Profile\n- Equity:\n- Vesting:\n- Responsibilities:",
    "client":    "## Client Profile\n- Contract:\n- Revenue:\n- SLA:",
    "investor":  "## Investor Profile\n- Thesis:\n- Check Size:\n- Fund Stage:",
    "doctor":    "## Medical Provider\n- Specialty:\n- Clinic:\n- Patients:",
    "family":    "## Family\n- Relation:\n- Birthday:\n- Allergies:",
    # ... 6 more archetypes
}

# Atomic writes prevent corruption
def atomic_write(filepath: str, content: str) -> None:
    """Write to temp file, set perms, rename (POSIX atomic)."""
    fd, tmp = tempfile.mkstemp(dir=os.path.dirname(filepath), suffix=".tmp")
    try:
        os.write(fd, content.encode())
        os.close(fd)
        os.chmod(tmp, 0o660)  # Production fix: mkstemp creates 600
        os.rename(tmp, filepath)
    except:
        os.unlink(tmp)
        raise

# _index.json for fast structured queries (no FAISS needed)
# Updated automatically on every crm.py operation
INDEX_SCHEMA = {
    "contacts": {
        "sarah-chen": {
            "name": "Sarah Chen",
            "company": "Acme Corp",
            "tags": ["partner", "technical"],
            "ventures": ["acme-project"],
            "file": "sarah-chen_a1b2.md",
            "last_interaction": "2026-03-15",
        }
    }
}
```

### Doc (Family Health Intelligence)

```python
# doc.py, structured health data with unit conversion

# Per-person data structure (JSONL time-series, append-only)
# workspace-doc/memory/{person}/vitals.jsonl
{"date": "2026-03-29", "type": "bp", "value": "120/80", "person": "alex",
 "notes": "Morning reading"}
{"date": "2026-03-29", "type": "weight", "value": "82.5", "unit": "kg",
 "person": "alex"}

# workspace-doc/memory/{person}/labs.jsonl
{"date": "2026-03-15", "type": "lab", "test": "Glucose", "value": "102.0",
 "unit": "mg/dL", "ref_high": "110", "ref_low": "70", "person": "alex",
 "converted": "5.66 mmol/L"}

# Unit conversion (Mexican -> Canadian lab values)
CONVERSIONS = {
    "glucosa":        {"factor": 1/18.018, "from": "mg/dL", "to": "mmol/L"},
    "glucose":        {"factor": 1/18.018, "from": "mg/dL", "to": "mmol/L"},
    "colesterol":     {"factor": 1/38.67,  "from": "mg/dL", "to": "mmol/L"},
    "cholesterol":    {"factor": 1/38.67,  "from": "mg/dL", "to": "mmol/L"},
    "trigliceridos":  {"factor": 1/88.57,  "from": "mg/dL", "to": "mmol/L"},
    "triglycerides":  {"factor": 1/88.57,  "from": "mg/dL", "to": "mmol/L"},
    "creatinina":     {"factor": 88.4,     "from": "mg/dL", "to": "umol/L"},
    "creatinine":     {"factor": 88.4,     "from": "mg/dL", "to": "umol/L"},
    "acido urico":    {"factor": 59.48,    "from": "mg/dL", "to": "umol/L"},
    "uric acid":      {"factor": 59.48,    "from": "mg/dL", "to": "umol/L"},
    "bun":            {"factor": 0.357,    "from": "mg/dL", "to": "mmol/L"},
    "urea":           {"factor": 0.166,    "from": "mg/dL", "to": "mmol/L"},
    "bilirrubina":    {"factor": 17.1,     "from": "mg/dL", "to": "umol/L"},
    "hemoglobina":    {"factor": 10.0,     "from": "g/dL",  "to": "g/L"},
    "albumina":       {"factor": 10.0,     "from": "g/dL",  "to": "g/L"},
}

def convert_units(test_name: str, value: float, unit: str) -> str | None:
    """Convert Mexican lab units to Canadian equivalents."""
    # Accent-insensitive matching
    test_lower = unicodedata.normalize('NFD', test_name.lower())
    test_lower = ''.join(c for c in test_lower if unicodedata.category(c) != 'Mn')

    for key, conv in CONVERSIONS.items():
        if key in test_lower and unit.lower() == conv["from"].lower():
            converted = value * conv["factor"]
            return f"{value} {unit} (= {converted:.2f} {conv['to']})"
    return None

# Per-agent access control
AGENT_ACCESS = {
    "main":     ["alex", "jordan", "child1", "child2"],  # Admin: all
    "jordan":   ["jordan", "child1", "child2"],            # Not Alex's data
    "doc":      ["alex", "jordan", "child1", "child2"],   # Health agent: all
}

def check_access(agent: str, person: str) -> bool:
    """Enforce per-agent health data access control."""
    allowed = AGENT_ACCESS.get(agent, [])
    return person.lower() in allowed
```

### Magic Link System

```python
# Three link types, each with different capabilities

# 1. SCOPED CHAT, for third party chats with limited agent
# magic_link.py create-chat --scope "playdate coordinator" --context "..." --ttl 24
{
    "token": "a1b2c3...",  # 256-bit random
    "type": "chat",
    "scope": "playdate coordinator",
    "context": "Coordinating playdate between two kids",
    "model": "google/gemini-2.5-flash",
    "ttl_hours": 24,
    "max_turns": 20,
    "rate_limit": "3/min",
    "created": "2026-03-29T10:00:00Z",
    "expires": "2026-03-30T10:00:00Z",
}

# 2. VISUAL CONTENT, for charts, dashboards, reports as HTML
# magic_link.py create-visual --html "<h1>Report</h1>" --label "Weight Trend"
{
    "token": "d4e5f6...",
    "type": "visual",
    "label": "Weight Trend Chart",
    "ttl_hours": 48,
    "has_feedback": True,  # User can submit feedback
}

# 3. LIVE PAGE, for real-time WebSocket updates
# magic_link.py create-live --label "Dashboard" --ttl 2
{
    "token": "g7h8i9...",
    "type": "live",
    "label": "Live Dashboard",
    "ws_clients": [],  # Connected WebSocket clients
}

# Sidecar server architecture
# magic_link_server.py, an aiohttp async server on 127.0.0.1:YOUR_SIDECAR_PORT
# Fronted by Caddy at /m/* with TLS
# Routes (actual implementation):
#   GET  /m/{token}              -> Render chat/visual/live page
#   WS   /m/{token}/ws           -> WebSocket (chat + visual/live push + feedback)
#   POST /m/{token}/message      -> HTTP fallback for chat (rate-limited)
#   POST /internal/create        -> Create link (Bearer auth)
#   POST /internal/update/{tok}  -> Push HTML content (visual/live)
#   GET  /internal/feedback/{tok} -> Read pending feedback
#   DELETE /internal/{tok}       -> Revoke link + generate summary
#   POST /internal/summarize/{tok} -> Generate summary without revoking
#   GET  /internal/list          -> List active/recent links
# Cleanup: cleanup timer (daily 4AM UTC, 48h grace)
# Protection: fail2ban jail (10 hits/5min = 1h ban)
```

### Model Switch

```python
# switch_model.py, change agent models via messaging command

MODEL_MAP = {
    "flash":    "google/gemini-2.5-flash",
    "pro":      "google/gemini-2.5-pro",
    "gpt":      "openai/gpt-4o-mini",
    "gpt5":     "openai/gpt-5-mini",
    "gpt5full": "openai/gpt-5",
    "haiku":    "anthropic/claude-haiku-4-5",
    "sonnet":   "anthropic/claude-sonnet-4",
    "opus":     "anthropic/claude-opus-4",
}

AGENT_ALIASES = {
    "assistant": "main", "r2": "r2", "ct": "ct", "doc": "doc",
    "jordan": "jordan", "agent3": "agent3",
    "genealogy": "genealogy", "ghost": "ghostwriter",
    "homeschool": "homeschool",
}

def switch_model(agent_alias: str, model_shortcut: str) -> str:
    """Switch an agent's default model in openclaw.json.

    Production gotcha: sessions.json may contain modelOverride fields
    that persist across /new and restarts. switch_model only changes
    openclaw.json, so it does NOT clear modelOverride.
    """
    agent_name = AGENT_ALIASES.get(agent_alias.lower())
    model_id = MODEL_MAP.get(model_shortcut.lower())
    if not agent_name or not model_id:
        return f"Unknown agent or model. Agents: {list(AGENT_ALIASES.keys())}"

    # Atomic backup + write
    config = load_config("openclaw.json")
    backup_config(config)  # Keep last 3 backups

    for agent in config["agents"]["list"]:
        if agent["id"] == agent_name:
            old_model = agent.get("model", "default")
            agent["model"] = model_id
            save_config(config)
            return f"Switched {agent_name} from {old_model} to {model_id}"

    return f"Agent {agent_name} not found in config"
```

---

## Security Architecture

### Defense in Depth (6 Layers)

```
Layer 1: Network (UFW firewall, only 22/80/443 exposed)
Layer 2: TLS (Caddy auto-cert, HSTS, security headers)
Layer 3: Authentication (SSH key-only, fail2ban, TOTP for web)
Layer 4: Encryption at Rest (LUKS encrypted vault, secrets encryption, encrypted overlays)
Layer 5: Access Control (per-user permissions, per-agent data access, ACLs)
Layer 6: Active Monitoring (system audit logging, security watchdog, vault protection)
```

### Layer 1: Network (UFW)

```bash
# Only these ports exposed to the internet
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    # SSH (key-only, fail2ban protected)
sudo ufw allow 80/tcp    # HTTP (Caddy auto-redirects to HTTPS)
sudo ufw allow 443/tcp   # HTTPS (Caddy with auto TLS)
sudo ufw enable

# All other services bind to 127.0.0.1 (loopback only):
# - Gateway, messaging daemon, sidecar services
# - FAISS indexer: no network (file watcher)
```

### Layer 2: TLS + Security Headers (Caddy)

```
# /etc/caddy/Caddyfile
yourdomain.com {
    # Reverse proxy to TESSA & COLE
    handle /api/* {
        reverse_proxy 127.0.0.1:8100
    }

    # Magic links
    handle /m/* {
        reverse_proxy 127.0.0.1:YOUR_SIDECAR_PORT
    }

    # Security headers (production-hardened)
    header {
        Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
        Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://unpkg.com; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' wss:"
        Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), accelerometer=(), gyroscope=(), magnetometer=()"
        Cross-Origin-Opener-Policy "same-origin"
        Cross-Origin-Resource-Policy "same-origin"
        -Server  # Strip server header
    }

    # Access logging (for fail2ban)
    log {
        output file /var/log/caddy/access.json
        format json
    }
}
```

### Layer 3: Authentication

```bash
# SSH hardening (/etc/ssh/sshd_config.d/99-hardening.conf)
PasswordAuthentication no
PermitRootLogin prohibit-password   # NEVER 'no' if using cloud console
AllowUsers admin root               # Whitelist only
MaxAuthTries 3
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
```

```ini
# fail2ban (/etc/fail2ban/jail.local)
[sshd]
enabled = true
bantime = 1h
maxretry = 3
bantime.increment = true
bantime.factor = 2        # Exponential backoff: 1h, 2h, 4h, 8h...

[caddy-auth]
enabled = true
bantime = 2h
maxretry = 5
findtime = 5m
bantime.increment = true

[caddy-magic-link]
enabled = true
port = http,https
filter = caddy-magic-link
logpath = /var/log/caddy/access.json
backend = auto
bantime = 1h
maxretry = 10
findtime = 5m
bantime.increment = true
bantime.factor = 2
```

### Layer 4: Encryption at Rest

```bash
# LUKS2 encrypted vault (2GB, AES-256-XTS)
# Contains ALL data: workspaces, agents, knowledge, config, credentials
dd if=/dev/zero of=/path/to/tessa_vault.img bs=1M count=2048
LOOP=$(sudo losetup --find)
sudo losetup $LOOP /path/to/tessa_vault.img
sudo cryptsetup luksFormat --type luks2 $LOOP
sudo cryptsetup luksOpen $LOOP tessa_crypt
sudo mkfs.ext4 /dev/mapper/tessa_crypt
sudo mount -o nosuid,nodev /dev/mapper/tessa_crypt /mnt/tessa_vault

# Additional encryption for secrets (keys, tokens, protocols)
# Use age, GPG, or similar tooling for encrypting individual files

# Encrypted directory overlays (defense in depth)
# Additional encryption layers on sensitive subdirectories within the vault
# MUST be mounted after every vault mount, or data will appear empty
```

### Layer 5: Access Control

```bash
# Permission model
# Owner: tessa (service user)     -> rwx / rw-
# Group: tessa (admin user is member) -> rwx / rw- (enables SFTP access)
# Other: ---                      -> no access

# Default ACLs (new files inherit group permissions)
sudo setfacl -R -d -m u::rwx,g::rwx,o::--- /mnt/tessa_vault/.tessa/

# Per-agent data access (enforced in code, not filesystem)
# See doc.py AGENT_ACCESS example above
```

```python
# Production lesson: AI editors change file ownership
# After editing any vault file with Claude Code/Cline/etc:
# sudo chown tessa:tessa <file>
# Without this, the service user can't read the file (script exit code 2)
```

### Layer 6: Monitoring

```python
# health_check.py, 20+ automated security/health checks
# Run via cron or on-demand

CHECKS = [
    # Service health
    ("main-service",       check_systemd_service, "tessa"),
    ("messaging-daemon",   check_port_listening, "<port>"),
    ("gateway",            check_port_listening, "<port>"),
    ("magic-link",         check_port_listening, "<port>"),
    ("memory-indexer",     check_systemd_service, "tessa-memory"),

    # Security
    ("vault-mounted",      check_mount, "/mnt/tessa_vault"),
    ("ssh-password-auth",  check_ssh_config, "PasswordAuthentication", "no"),
    ("ssh-root-login",     check_ssh_config, "PermitRootLogin", "prohibit-password"),
    ("ssh-allow-users",    check_ssh_config, "AllowUsers", "contains root"),
    ("firewall-active",    check_ufw_status),
    ("exposed-ports",      check_exposed_ports, [22, 80, 443]),  # Only these
    ("env-permissions",    check_file_perms, ".env", 0o640),

    # Resources
    ("ram-usage",          check_ram_percent, 85),  # Warn above 85%
    ("disk-usage",         check_disk_percent, 80),
    ("swap-usage",         check_swap_percent, 50),

    # Integrity
    ("faiss-freshness",    check_file_age, "brain.index", hours=24),
    ("backup-freshness",   check_file_age, "backups/", hours=26),
]

def run_watchdog(quiet: bool = False) -> list[dict]:
    """Run all checks, return status report."""
    results = []
    for name, check_fn, *args in CHECKS:
        try:
            status, detail = check_fn(*args)
            if not quiet or status != "OK":
                results.append({"check": name, "status": status, "detail": detail})
        except Exception as e:
            results.append({"check": name, "status": "ERROR", "detail": str(e)})
    return results
```

---

## Systemd Services

### Main Service

```ini
# /etc/systemd/system/tessa.service
[Unit]
Description=TESSA & COLE AI Command Center
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=simple
User=tessa
Group=tessa
WorkingDirectory=/opt/tessa
ExecStart=/opt/tessa/venv/bin/uvicorn server.main:app --host 127.0.0.1 --port 8100
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

# Lightweight hardening (production-safe)
NoNewPrivileges=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
# WARNING: Do NOT add ProtectSystem=strict, PrivateTmp, ReadWritePaths
# These caused a server freeze (load 17, CPU spike) in production.
# Complex filesystem access patterns (messaging daemons, Docker, LUKS)
# cannot be safely whitelisted.

[Install]
WantedBy=multi-user.target
```

### FAISS Memory Indexer

```ini
# /etc/systemd/system/tessa-memory.service
[Unit]
Description=TESSA & COLE FAISS Memory Indexer
After=network-online.target

[Service]
Type=simple
User=root
ExecStart=/path/to/tessa_brain/bin/python /path/to/tessa_brain/memory_sync.py
Restart=always
RestartSec=5
Environment=OPENAI_API_KEY=sk-...

[Install]
WantedBy=multi-user.target
```

### Magic Link Sidecar

```ini
# /etc/systemd/system/tessa-magic-link.service
[Unit]
Description=TESSA & COLE Magic Link Sidecar
After=tessa.service

[Service]
Type=simple
User=tessa
WorkingDirectory=/opt/tessa
EnvironmentFile=/opt/tessa/.env
ExecStart=/opt/tessa/venv/bin/python workspace/skills/magic-link/scripts/magic_link_server.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Magic Link Cleanup Timer

```ini
# /etc/systemd/system/tessa-magic-link-cleanup.timer
[Unit]
Description=Daily Magic Link cleanup

[Timer]
OnCalendar=*-*-* 04:00:00
RandomizedDelaySec=300
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/tessa-magic-link-cleanup.service
[Unit]
Description=Magic Link Cleanup, purge expired tokens and files
After=tessa-magic-link.service

[Service]
Type=oneshot
User=tessa
WorkingDirectory=/opt/tessa
ExecStart=/opt/tessa/venv/bin/python workspace/skills/magic-link/scripts/cleanup.py --grace-hours 48
```

### Timers

```ini
# /etc/systemd/system/tessa-backup.timer
[Unit]
Description=Daily TESSA & COLE Backup

[Timer]
OnCalendar=*-*-* 05:00:00 UTC
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/jke-synthesize.timer
[Unit]
Description=Weekly JKE Knowledge Synthesis

[Timer]
OnCalendar=Sun *-*-* 08:33:00 UTC
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

---

## Backup & Recovery

### Backup Script

```python
# tessa_backup.py, daily encrypted backup
# Timer: 5:00 AM UTC (midnight EST), 14-day rotation
# Encryption: AES-256 symmetric GPG
# Size: ~50MB/day compressed

import subprocess
import tarfile
import os
from datetime import datetime, timedelta

BACKUP_DIR = "/path/to/backups"
VAULT_BASE = "/mnt/tessa_vault/.tessa"
PASSPHRASE_FILE = "/path/to/backup.key"
RETENTION_DAYS = 14

INCLUDE_DIRS = [
    "workspace", "workspace-jordan", "workspace-agent3",
    "workspace-agent4", "workspace-ghost", "workspace-family",
    "workspace-ct", "workspace-r2", "workspace-doc",
    "knowledge", "agents", "credentials", "identity",
    "config", "cron", "media",
]

EXCLUDE_PATTERNS = [
    "*/venv/*", "*/.git/*", "*/node_modules/*",
    "*/__pycache__/*", "*/.venv/*",
]

def create_backup():
    """Create encrypted compressed backup."""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    tar_path = f"/tmp/tessa_backup_{timestamp}.tar.gz"
    gpg_path = os.path.join(BACKUP_DIR, f"tessa_backup_{timestamp}.tar.gz.gpg")

    # Create tar.gz
    with tarfile.open(tar_path, "w:gz") as tar:
        for dirname in INCLUDE_DIRS:
            full_path = os.path.join(VAULT_BASE, dirname)
            if os.path.exists(full_path):
                tar.add(full_path, arcname=dirname,
                        filter=lambda ti: None if any(
                            ti.name.match(p) for p in EXCLUDE_PATTERNS) else ti)

    # Add FAISS index
    for f in ["brain.index", "metadata.json", "bm25_corpus.json"]:
        path = f"/path/to/tessa_brain/{f}"
        if os.path.exists(path):
            with tarfile.open(tar_path, "a:gz") as tar:
                tar.add(path, arcname=f"faiss/{f}")

    # Encrypt with GPG
    subprocess.run([
        "gpg", "--batch", "--yes", "--symmetric",
        "--cipher-algo", "AES256",
        "--passphrase-file", PASSPHRASE_FILE,
        "--output", gpg_path, tar_path
    ], check=True)

    os.unlink(tar_path)  # Delete unencrypted tar
    prune_old_backups()
    return gpg_path


def prune_old_backups():
    """Delete backups older than RETENTION_DAYS."""
    cutoff = datetime.now() - timedelta(days=RETENTION_DAYS)
    for f in os.listdir(BACKUP_DIR):
        fpath = os.path.join(BACKUP_DIR, f)
        if os.path.getmtime(fpath) < cutoff.timestamp():
            os.unlink(fpath)
```

### Restore Procedure

```bash
# Decrypt
gpg --batch --passphrase-file /path/to/backup.key \
    --decrypt backup.tar.gz.gpg > backup.tar.gz

# Extract
tar -xzf backup.tar.gz -C /mnt/tessa_vault/.tessa/

# Rebuild FAISS index
sudo systemctl restart tessa-memory

# Fix permissions
sudo chown -R tessa:tessa /mnt/tessa_vault/.tessa/
sudo find /mnt/tessa_vault/.tessa/ -type d -exec chmod 770 {} \;
sudo find /mnt/tessa_vault/.tessa/ -type f -exec chmod 660 {} \;
```

### Safe Restart Script

```bash
#!/bin/bash
# /opt/restart-tessa.sh, safe restart that handles zombie processes

# Stop service
systemctl stop tessa

# Kill zombie processes on the gateway port
PIDS=$(lsof -ti :8100 2>/dev/null)
if [ -n "$PIDS" ]; then
    kill $PIDS 2>/dev/null
    sleep 2
    kill -9 $PIDS 2>/dev/null
fi

# Verify port is free
sleep 1
if lsof -ti :8100 >/dev/null 2>&1; then
    echo "ERROR: Port 8100 still in use"
    exit 1
fi

# Start service
systemctl start tessa

# Verify running
sleep 2
if systemctl is-active --quiet tessa; then
    echo "TESSA restarted successfully"
else
    echo "ERROR: TESSA failed to start"
    journalctl -u tessa -n 20
    exit 1
fi
```

### Reboot Recovery

```bash
#!/bin/bash
# Post-reboot recovery (vault does NOT auto-mount)

# 1. Mount encrypted vault
LOOP=$(sudo losetup --find)
sudo losetup $LOOP /path/to/tessa_vault.img
sudo cryptsetup luksOpen $LOOP tessa_crypt  # Prompts for passphrase
sudo mount -o nosuid,nodev /dev/mapper/tessa_crypt /mnt/tessa_vault

# 2. Mount encrypted directory overlays (CRITICAL, don't skip!)
sudo /path/to/mount-vault-overlays.sh

# 3. Kill orphan messaging daemon processes
sudo pkill -f messaging daemon 2>/dev/null ; sleep 2

# 4. Start services
sudo /opt/restart-tessa.sh
sudo systemctl start tessa-memory
sudo systemctl start tessa-magic-link

# 5. Verify
sudo systemctl status tessa --no-pager
sudo ss -tlnp | grep -E "8100|YOUR_SIDECAR_PORT"
```

---

## Production Gotchas (Lessons Learned)

### Critical: Things That Will Break Your System

| Gotcha | What Happens | Prevention |
|--------|-------------|------------|
| `systemctl restart tessa` | Zombie port, crash loop | Use `/opt/restart-tessa.sh` |
| Adding unknown keys to config.json | Gateway crashes (exit 1) | Only use documented keys |
| Running TESSA as root | Permission cascade failures | Always `sudo -u tessa` |
| `ProtectSystem=strict` in systemd | Server freeze (load 17) | Use lightweight hardening only |
| Editing vault files as admin user | Service can't read (perm denied) | `chown tessa:tessa` after every edit |
| Skipping encrypted dir mounts | Data dirs empty, scripts report "no data" | ALWAYS mount after vault mount |
| `losetup -D` | Detaches ALL loops including vault | Use `losetup -d /dev/loopN` specific |
| Manual .md file creation | Bypasses indexing, invisible to search | Always use scripts (crm.py, ct.py, etc.) |
| Dollar signs in double quotes | Shell interpolation (`$282.50` becomes `82.50`) | Use single quotes for amounts |
| sessions.json modelOverride | Persists across restarts, overrides config | Check sessions.json if model seems wrong |
| Deleting dirs without checking symlinks | Breaks linked systems | `find / -lname "*dir*"` first |

### Anti-Patterns to Avoid

1. **Don't ask the LLM to orchestrate.** Use scripts. Models narrate instead of executing.
2. **Don't ask the LLM to summarize cron output.** They always paraphrase. Say "VERBATIM."
3. **Don't put rules in ASSISTANT_RULES.md.** OpenClaw doesn't load it. Use TOOLS.md.
4. **Don't trust memory_search for structured data.** It only searches flat .md files.
5. **Don't use mkstemp without chmod 660.** Default 600 blocks group access.
6. **Don't duplicate SOUL.md instructions.** Double-loading wastes tokens.
7. **Don't delete without `ls -la` first.** You might miss files inside.

---

*Continue to Part 5: Build Plan & Code Reference*
