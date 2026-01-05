# Email Cleaner — **Architecture**
> Purpose: Automatically clean Gmail and Proton Mail accounts using local LLM (Gemma 3 1B) to delete/move promotional and unwanted emails based on configurable rules. Runs on Raspberry Pi 5 (8GB).

---

## 0) Goals & Non-Goals

**Goals**

* Local LLM processing (no cloud API calls, no data exfiltration)
* Support both Gmail (OAuth2) and Proton Mail (Bridge IMAP)
* Configurable rules in text/YAML files
* "Valuable" emails exempt from future LLM checks
* Daily or scheduled batch processing
* Dry-run mode for testing rules before execution

**Non-Goals**

* No real-time filtering (batch only)
* No email sending/composing
* No modification of "valuable" marked emails without explicit command
* No multi-account sync between Gmail and Proton

---

## 1) High-Level Overview

**Components**

1. **Proton Bridge (Docker)** — local IMAP/SMTP for Proton Mail
2. **Gmail API Client** — OAuth2 authenticated Google API client
3. **LLM Service** — llama.cpp or Ollama serving quantized Gemma 3 1B
4. **Email Cleaner (Node.js)** — orchestrator: fetch → classify → action
5. **SQLite Database** — state, exemptions, audit log
6. **Scheduler** — systemd timer or node-cron

**Data Flow**

1. Scheduler triggers clean job at configured interval
2. Load config: accounts, rules, limits, date range
 each account:
  3. For - Fetch headers/previews (Gmail: API `messages.list` + `messages.get`; Proton: IMAP)
   - Filter out already-exempt emails (by ID)
   - Batch emails (e.g., 10-20 per batch for LLM context)
   - Build prompt with rules + email previews
   - Query local LLM
   - Parse LLM response → actions (KEEP, DELETE, TRASH)
4. Execute actions (Gmail: `messages.modify` (trash); Proton: IMAP MOVE)
5. Update database: mark valuable emails, log actions
6. Report summary

---

## 2) Runtime & OS Assumptions

* Raspberry Pi 5 (ARM64), Void Linux
* See [pi_setup.md](../pi_setup.md) for OS installation and configuration
* Node.js ≥ 20 LTS
* Docker for Proton Bridge container
* 8GB RAM: allocate ~3GB to LLM, rest to OS + app

---

## 3) Directory Layout

```
/srv/email-cleaner
├── app/
│   ├── src/
│   │   ├── cli.ts
│   │   ├── config/
│   │   │   └── loader.ts
│   │   ├── accounts/
│   │   │   ├── gmail.ts
│   │   │   └── proton.ts
│   │   ├── llm/
│   │   │   └── client.ts
│   │   ├── cleaner/
│   │   │   ├── fetcher.ts
│   │   │   ├── classifier.ts
│   │   │   └── executor.ts
│   │   ├── store/
│   │   │   ├── sqlite.ts
│   │   │   └── exemptions.ts
│   │   └── types.ts
│   ├── package.json
│   └── .env.example
├── config/
│   └── default.yaml
├── rules/
│   └── user-rules.yaml      # User's cleaning rules
├── data/
│   ├── state.sqlite         # Idempotency & exemptions
│   └── audit.log
└── ops/
    ├── docker-compose.yml   # Proton Bridge
    ├── systemd/
    │   ├── email-clean.service
    │   └── email-clean.timer
    └── ollama/
        └── start-llm.sh     # Gemma 3 1B startup script
```

---

## 4) Email Provider Integration

### 4.1 Gmail API (OAuth2)

**Setup**

* Create Google Cloud project → enable Gmail API
* Create OAuth2 credentials (desktop app)
* First run: authenticate via URL, paste token
* Store refresh token in `.env` or OS keyring

**Libraries**

* `@googleapis/gmail` — official Google client
* `google-auth-library` — OAuth handling

**Rate Limits**

* Gmail: 250 API calls/second/user (generous)
* Batch fetch headers to minimize calls

**Permissions required**

```
gmail.readonly    — read emails
gmail.modify     — trash/delete emails
```

### 4.2 Proton Mail (Bridge IMAP)

**Why Bridge**

Proton does not offer public third-party API. Bridge provides local IMAP/SMTP access after authentication.

**Compose**

```yaml
services:
  proton-bridge:
    image: ghcr.io/videocurio/proton-mail-bridge:latest
    container_name: protonmail_bridge
    restart: unless-stopped
    volumes:
      - /srv/email-cleaner/proton-bridge:/root
    ports:
      - "127.0.0.1:12143:143"   # IMAP STARTTLS
    environment:
      - TZ=Europe/Amsterdam
```

**First-run**

```bash
sudo docker compose up -d
docker exec -it protonmail_bridge bridge --cli
# login → 2FA → wait sync → `info` shows local IMAP creds
```

**Security**

* Ports bound to 127.0.0.1 only
* Local credentials used by app (not Proton credentials)

### 4.3 Unified Account Interface

```ts
interface EmailProvider {
  name: 'gmail' | 'proton';
  fetchEmails(options: { limit?: number; since?: Date }): Promise<EmailPreview[]>;
  trashEmail(messageId: string): Promise<void>;
  moveToFolder(messageId: string, folder: string): Promise<void>;
}

interface EmailPreview {
  id: string;
  subject: string;
  from: string;
  to: string;
  date: Date;
  snippet: string;  // First ~200 chars of body
  labels?: string[]; // Gmail: labels; Proton: folders
}
```

---

## 5) Local LLM Integration

### 5.1 Model Choice

**Gemma 3 1B (quantized 4-bit)**

* ~2.5GB memory footprint
* ~5-15 tokens/second on Pi 5
* Good enough for simple rule classification
* Alternative: Gemma 3 3B for better accuracy (slower)

**Why not cloud**

* Privacy: emails never leave device
* Cost: free after initial model download
* Latency: acceptable for batch processing

### 5.2 Serving Options

**Option A: Ollama (recommended)**

```bash
ollama run gemma3:1b-instruct-q4_0
# REST API on localhost:11434
```

**Option B: llama.cpp server**

```bash
./server -m gemma-3-1b-it-q4_0.gguf -c 2048 -host 127.0.0.1 -port 8080
```

**Memory Calculation**

```
Gemma 3 1B Q4_0: ~1.4GB model + 1GB KV cache = ~2.5GB
Pi 5 System: ~2GB
Node.js App: ~200MB
Headroom: ~3GB (comfortable)
```

### 5.3 Prompt Engineering

**System Prompt**

```
You are an email classification assistant. Given email previews and user rules,
classify each email as KEEP, DELETE, or TRASH.

Rules:
- Delete promotional, spam, marketing emails
- Keep newsletters user explicitly wants
- Keep transactional emails (orders, receipts, shipping)
- Trash emails that are unwanted but not spam

Return JSON array: [{"id": "...", "action": "KEEP|DELETE|TRASH", "reason": "..."}]
```

**User Prompt**

```
Rules:
${userRules}

Emails to classify:
${emailPreviewsJSON}

Respond ONLY with JSON array.
```

### 5.4 LLM Client

```ts
// src/llm/client.ts
import ollama from 'ollama';

export async function classifyEmails(
  emails: EmailPreview[],
  rules: string
): Promise<Classification[]> {
  const prompt = buildPrompt(emails, rules);
  
  const response = await ollama.chat({
    model: 'gemma3:1b-instruct-q4_0',
    messages: [
      { role: 'system', content: SYSTEM_PROMPT },
      { role: 'user', content: prompt }
    ],
    format: 'json',
    options: {
      temperature: 0.1,  // deterministic output
      num_predict: 512   // limit response size
    }
  });

  return parseResponse(response.message.content);
}
```

---

## 6) Rules Engine

### 6.1 Rule Format (YAML)

```yaml
# rules/user-rules.yaml
delete:
  - patterns:
      - "promotional"
      - "marketing"
      - "unsubscribe"
      - "special offer"
      - "limited time"
    keywords_in_subject: true
  - from_domains:
      - "*.marketing.com"
      - "newsletter-*.xyz"
  - labels: ["PROMOTIONS", "SPAM"]

keep:
  - from_addresses:
      - "newsletter@substack.com"
      - "updates@github.com"
  - subjects:
      - "Weekly digest"
      - "Order confirmation"
  - labels: ["IMPORTANT", "WORK"]

trash:
  - patterns:
      - "you won"
      - "claim your prize"
      - "dear customer"  # generic spam
```

### 6.2 Rule Pre-Filtering (Before LLM)

Apply simple regex first to reduce LLM load:

* If email matches `keep` rule → KEEP immediately
* If email matches `delete` rule → DELETE immediately
* Otherwise → send to LLM for nuance

### 6.3 Confidence Thresholds

```ts
interface Classification {
  id: string;
  action: 'KEEP' | 'DELETE' | 'TRASH';
  reason: string;
  confidence: number;  // 0-1
}

// If confidence < 0.7 → require manual review queue
```

---

## 7) "Valuable" Exemption System

### 7.1 Concept

Emails previously classified as KEEP with high confidence are marked "valuable" and skipped in future runs.

### 7.2 Database Schema

```sql
CREATE TABLE exemptions (
  account TEXT,
  email_id TEXT PRIMARY KEY,
  classified_at TEXT,
  reason TEXT,
  manual_override BOOLEAN DEFAULT FALSE
);

CREATE TABLE audit_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT,
  account TEXT,
  email_id TEXT,
  action TEXT,
  llm_reason TEXT,
  confidence REAL,
  timestamp TEXT
);
```

### 7.3 Flow

```
1. Fetch emails
2. Filter: WHERE email_id NOT IN (SELECT email_id FROM exemptions)
3. Classify remaining
4. For KEEP with confidence > 0.9 → INSERT INTO exemptions
5. For manual review queue → skip, flag for user
```

### 7.4 Manual Commands

```bash
# Mark email as valuable (exempt from future checks)
node app/dist/cli.js exempt --account gmail --email-id <id>

# Remove exemption (re-evaluate in next run)
node app/dist/cli.js unexempt --account gmail --email-id <id>

# List exemptions
node app/dist/cli.js exemptions --account proton
```

---

## 8) Configuration (YAML)

```yaml
# config/default.yaml
paths:
  data: /srv/email-cleaner/data
  rules: /srv/email-cleaner/rules/user-rules.yaml

accounts:
  gmail:
    enabled: true
    credentials_path: /srv/email-cleaner/data/gmail-token.json
    scopes:
      - https://www.googleapis.com/auth/gmail.readonly
      - https://www.googleapis.com/auth/gmail.modify

  proton:
    enabled: true
    host: 127.0.0.1
    port: 12143
    user: "local-bridge-user"
    pass: "local-bridge-pass"
    mailbox: "All Mail"

cleaning:
  batch_size: 15           # emails per LLM call
  max_emails_per_run: 100  # limit processing
  date_range_days: 7       # look back X days
  dry_run: false           # don't actually delete
  confidence_threshold: 0.7
  auto_exempt_above: 0.9   # auto-exempt KEEP with confidence > 0.9

llm:
  provider: ollama          # ollama | llama-cpp
  host: 127.0.0.1:11434
  model: gemma3:1b-instruct-q4_0
  timeout_ms: 30000

exemptions:
  auto_mark: true
  manual_override_file: /srv/email-cleaner/data/manual-exempt.json

scheduling:
  enabled: true
  interval: daily          # daily | hourly | custom
  time: "08:00"           # HH:MM in system timezone
  timezone: Europe/Amsterdam

logging:
  level: info
  audit_path: /srv/email-cleaner/data/audit.log
```

**Secrets**

* Store OAuth tokens and Bridge credentials in `.env` or OS keyring
* Config loader: `.env` → YAML → CLI flags

---

## 9) CLI Commands

```bash
# Dry-run with Gmail (no deletions)
node app/dist/cli.js clean --account gmail --dry-run

# Clean Proton, last 14 days, max 50 emails
node app/dist/cli.js clean --account proton --days 14 --max 50

# Force re-evaluate all emails (ignore exemptions)
node app/dist/cli.js clean --all --force

# Mark email as valuable
node app/dist/cli.js exempt --account gmail --id <msg-id>

# View audit log
node app/dist/cli.js audit --since 2025-01-01

# Status check
node app/dist/cli.js status
```

**Flags**

* `--account gmail|proton|all`
* `--dry-run` — preview only
* `--days N` — look back N days (default: 7)
* `--max N` — limit emails processed
* `--force` — ignore exemptions, re-evaluate all

---

## 10) Scheduling (systemd)

**Daily clean at 08:30**

```ini
# ops/systemd/email-clean.service
[Unit]
Description=Email Cleaner — Daily Clean
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/srv/email-cleaner
Environment=NODE_ENV=production
ExecStart=/usr/bin/node app/dist/cli.js clean
User=emailclean
Group=emailclean

# ops/systemd/email-clean.timer
[Unit]
Description=Run email cleaner daily

[Timer]
OnCalendar=*-*-* 08:30:00 Europe/Amsterdam
Persistent=true

[Install]
WantedBy=timers.target
```

**Enable**

```bash
sudo cp -a ops/systemd/*.service ops/systemd/*.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now email-clean.timer
```

---

## 11) Idempotency & Safety

**Safety Checks**

1. **Dry-run default** — require `--confirm` flag for actual deletions
2. **Confirmation prompt** — show summary before execution
3. **Rate limiting** — max N emails per run
4. **Audit log** — every action logged with timestamp
5. **Rollback** — store message IDs in temp table; only commit after success

**Duplicate Prevention**

* Track processed `message_id` per run in SQLite
* Skip already-processed emails if re-run

---

## 12) Performance Notes (Pi 5)

* Batch size 10-15 emails balances context vs. speed
* Q4_0 quantization for optimal performance
* Pre-warm model on startup

**Network I/O**

* Gmail: batch `messages.list` → parallel `messages.get` (max 10 concurrent)
* Proton: IMAP IDLE or UID SEARCH + FETCH

**Memory**

* Monitor with `free -h`
* Restart LLM service if memory > 7GB
* Consider swap file for edge cases

---

## 13) Security & Privacy

* **No data exfiltration** — all processing local
* **Loopback only** — Proton Bridge ports bound to 127.0.0.1
* **Minimal permissions** — Gmail scopes limited to read/modify
* **Dedicated user** — run as `emailclean` with restricted filesystem access
* **Encrypted storage** — consider LUKS for `/srv/email-cleaner/data`
* **Token security** — OAuth tokens in `.env` with 0600 permissions

---

## 14) Observability

* **Logs**: JSON to stdout/journald (pino)
* **Metrics per run**: emails processed, kept, deleted, trashed, errors
* **Health file**: `data/health.json` with last run timestamp
* **Alerting**: optional webhook on high error rate

---

## 15) Testing Strategy

**Unit tests**

* Rule parser (YAML → AST)
* LLM prompt builder
* Response parser (JSON → classifications)
* Database operations

**Integration tests (mock providers)**

* Gmail mock: simulate API responses
* Proton mock: IMAP simulation
* LLM mock: predefined responses

**Fixture set**

* 20 emails: 10 promotional (should delete), 5 important (keep), 5 edge cases (trash)
* Run dry-run, compare expected vs. actual

---

## 16) Dependencies (package.json)

```json
{
  "name": "email-cleaner",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "build": "tsc -p .",
    "clean": "node app/dist/cli.js clean"
  },
  "dependencies": {
    "@googleapis/gmail": "^9",
    "better-sqlite3": "^9",
    "commander": "^12",
    "google-auth-library": "^9",
    "imapflow": "^1",
    "ollama": "^0.1",
    "pino": "^9",
    "yaml": "^2"
  },
  "devDependencies": {
    "typescript": "^5"
  }
}
```

**System dependencies**

* `ollama` (binary) or `llama.cpp` compiled for ARM64
* Docker (for Proton Bridge)

---

## 17) First-Time Setup

See [pi_setup.md](../pi_setup.md) for OS, Docker, and Ollama setup.

```bash
# 1. Clone and build
cd /srv/email-cleaner
npm install
npm run build

# 2. Configure Gmail OAuth
# ... create credentials in Google Cloud Console
cp .env.example .env
# ... edit .env with tokens

# 3. Test dry-run
node app/dist/cli.js clean --account gmail --dry-run

# 4. Enable scheduler
sudo cp -a ops/systemd/*.service ops/systemd/*.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now email-clean.timer
```

---

## 18) Troubleshooting

| Issue | Solution |
|-------|----------|
| LLM out of memory | Use Q4_0 quantization, reduce batch_size |
| Proton Bridge sync slow | Wait initial sync; increase Bridge container resources |
| Gmail rate limit errors | Reduce `max_emails_per_run`, add delay between API calls |
| False positives (important email deleted) | Lower confidence threshold, add to `keep` rules |
| LLM timeout | Increase `timeout_ms`, reduce batch size |

---

## Appendix A — Example User Rules

```yaml
delete:
  - patterns:
      - "promotional"
      - "unsubscribe"
      - "limited time offer"
    keywords_in_subject: true
  - from_domains:
      - "*.marketing.com"
      - "newsletter-ads.com"
  - labels: ["PROMOTIONS"]

keep:
  - from_addresses:
      - "github.com noreply"
      - "aws-amazon.com"
      - "support@stripe.com"
  - subjects:
      - "security alert"
      - "order shipped"
      - "invoice"
  - labels: ["IMPORTANT", "INBOX"]

trash:
  - patterns:
      - "you have won"
      - "dear customer"
      - "click here to claim"
```

---

## Appendix B — Example Audit Log Entry

```json
{
  "run_id": "2025-01-15T08:30:00Z",
  "account": "gmail",
  "email_id": "msg-12345",
  "action": "DELETE",
  "llm_reason": "Promotional email with 'limited time offer' in subject",
  "confidence": 0.95,
  "timestamp": "2025-01-15T08:32:15Z"
}
```

---