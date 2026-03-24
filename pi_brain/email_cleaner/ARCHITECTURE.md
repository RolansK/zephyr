oss_life\pi_brain\email_cleaner\ARCHITECTURE.md
```
```markdown
# Email Cleaner — Architecture

> Automatically clean Gmail and Proton Mail using a local LLM. Runs on Raspberry Pi 5.

---

## 0) Goals

* Local LLM only (no cloud, no data exfiltration)
* Support Gmail (OAuth2) and Proton Mail (Bridge IMAP)
* Configurable YAML rules for delete/keep/trash
* Auto-exempt "valuable" emails from future checks
* Dry-run mode; idempotent batch processing
* Daily scheduled clean via systemd timer

---

## 1) Stack

| Component | Choice |
|-----------|--------|
| Language | Go ≥ 1.22 |
| LLM Serving | llama.cpp (native binary, no Ollama) |
| Model | Qwen3.5 2B Q4_0 (~2GB) |
| Database | SQLite (exemptions + audit log) |
| Email | Gmail API + imapflow (Proton Bridge) |
| Scheduler | systemd timer |
| Container | Docker (Proton Bridge only) |

**Memory (Pi 5 / 8GB)**

```
Qwen3.5 2B Q4_0: ~2GB model + 1GB KV cache
OS + Go app:    ~1.5GB
Headroom:       ~4.5GB
```

---

## 2) Directory Layout

```
/srv/email-cleaner/
├── app/
│   ├── main.go
│   ├── go.mod / go.sum
│   └── cmd/
│       └── clean/
│           └── main.go        # CLI entrypoint
├── config/
│   └── default.yaml
├── rules/
│   └── user-rules.yaml
├── data/
│   ├── state.db               # SQLite: exemptions, audit_log
│   └── audit.log
├── llm/
│   └── qwen3.5-2b-q4_0.gguf
├── proton-bridge/             # Docker volume
└── ops/
    ├── docker-compose.yml
    └── systemd/
        ├── email-clean.service
        └── email-clean.timer
```

---

## 3) Email Providers

### 3.1 Gmail API (OAuth2)

* Gmail API v1 with `google-auth-library` equivalent in Go (`golang.org/x/oauth2`)
* Scopes: `gmail.readonly`, `gmail.modify`
* First run: OAuth2 device flow → store refresh token

### 3.2 Proton Mail Bridge (IMAP)

* Proton Bridge runs in Docker (`ghcr.io/videocurio/proton-mail-bridge`)
* Exposes IMAP on `127.0.0.1:12143`
* App connects via `github.com/emersion/go-imap` or `imapflow`

### 3.3 Unified Interface

```go
type EmailPreview struct {
    ID       string
    Subject  string
    From     string
    Date     time.Time
    Snippet  string // ~200 chars
    Labels   []string // Gmail labels or Proton folders
}

type EmailProvider interface {
    FetchEmails(ctx context.Context, opts FetchOptions) ([]EmailPreview, error)
    TrashEmail(ctx context.Context, id string) error
    MoveEmail(ctx context.Context, id, folder string) error
}
```

---

## 4) LLM Integration

### 4.1 llama.cpp Server

```bash
./llama-server -m llm/qwen3.5-2b-q4_0.gguf \
  -c 2048 --host 127.0.0.1 --port 8080
```

App queries via HTTP POST to `http://127.0.0.1:8080/completion`.

### 4.2 Prompt

**System**

```
You are an email classification assistant. Classify each email as KEEP, DELETE, or TRASH.
- DELETE: promotional, spam, marketing, newsletters user doesn't want
- KEEP: transactional (orders, receipts, shipping), important, explicitly wanted
- TRASH: unwanted but not clearly spam

Return JSON array only: [{"id":"...","action":"KEEP|Delete|Trash","reason":"...","confidence":0.0}]
```

**User**

```
Rules:
${yamlRules}

Emails:
${emailPreviewsJSON}
```

**Params:** `temperature=0.1`, `num_predict=256`

---

## 5) Rules Engine

### 5.1 YAML Format

```yaml
delete:
  - patterns: ["promotional", "unsubscribe", "limited time"]
    keywords_in_subject: true
  - from_domains: ["*.marketing.com", "newsletter-ads.xyz"]
  - labels: ["PROMOTIONS", "SPAM"]

keep:
  - from_addresses: ["github.com", "support@stripe.com"]
  - subjects: ["order confirmation", "security alert"]
  - labels: ["IMPORTANT"]

trash:
  - patterns: ["you have won", "click here to claim"]
```

### 5.2 Pre-Filtering

Apply regex/keyword rules **before** LLM:
- `keep` match → KEEP immediately
- `delete` match → DELETE immediately
- Otherwise → send to LLM

### 5.3 Confidence

```go
type Classification struct {
    ID        string
    Action    string // "keep", "delete", "trash"
    Reason    string
    Confidence float64 // 0.0–1.0
}
```

If LLM confidence < `0.7` → queue for manual review.

---

## 6) Exemption System

### 6.1 Database Schema

```sql
CREATE TABLE exemptions (
    account    TEXT,
    email_id   TEXT PRIMARY KEY,
    reason     TEXT,
    confidence REAL,
    created_at TEXT
);

CREATE TABLE audit_log (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id     TEXT,
    account    TEXT,
    email_id   TEXT,
    action     TEXT,
    reason     TEXT,
    confidence REAL,
    timestamp  TEXT
);
```

### 6.2 Flow

1. Fetch emails
2. Skip already-exempt IDs
3. Apply pre-filter rules
4. Remaining → LLM classification
5. `keep` + confidence > `0.9` → insert into `exemptions`
6. Log all actions

### 6.3 CLI

```bash
email-cleaner exempt --account gmail --id <msg-id>
email-cleaner unexempt --account gmail --id <msg-id>
email-cleaner exemptions --account proton
```

---

## 7) Configuration

```yaml
paths:
  data:  /srv/email-cleaner/data
  rules: /srv/email-cleaner/rules/user-rules.yaml

accounts:
  gmail:
    enabled:           true
    credentials_path:  /srv/email-cleaner/data/gmail-token.json
  proton:
    enabled:           true
    host:              127.0.0.1
    port:              12143
    user:              "bridge-user"
    pass:              "bridge-pass"
    mailbox:           "All Mail"

cleaning:
  batch_size:        10
  max_per_run:       100
  date_range_days:   7
  dry_run:           true
  confidence_threshold: 0.7
  auto_exempt_above: 0.9

llm:
  server_url:  http://127.0.0.1:8080
  model:       qwen3.5-2b-q4_0
  timeout_sec: 30

scheduling:
  enabled:   true
  time:      "08:30"
  timezone:  Europe/Amsterdam

logging:
  level:      info
  audit_path: /srv/email-cleaner/data/audit.log
```

---

## 8) CLI Commands

```bash
email-cleaner clean --account gmail --dry-run
email-cleaner clean --account proton --days 14 --max 50
email-cleaner clean --all --force
email-cleaner audit --since 2025-01-01
email-cleaner status
```

---

## 9) Scheduling

```ini
# /etc/systemd/system/email-clean.timer
[Timer]
OnCalendar=*-*-* 08:30:00 Europe/Amsterdam
Persistent=true
```

---

## 10) Idempotency & Safety

* **Dry-run default** — `--confirm` flag required for actual deletions
* **Confirmation prompt** — summary shown before execution
* **Rate limits** — configurable `max_per_run`
* **Audit log** — every action logged with `run_id`, timestamp
* **Skip duplicates** — already-processed IDs skipped within same run

---

## 11) Security

* All data stays local (no cloud calls)
* Proton Bridge bound to loopback only
* Gmail OAuth scopes minimal (`readonly` + `modify`)
* Run as dedicated user `emailclean`
* Tokens stored with `0600` permissions

---

## 12) Dependencies (Go)

```
github.com/emersion/go-imap
github.com/mattn/go-sqlite3
gopkg.in/yaml.v3
golang.org/x/oauth2
github.com/googleapis/gmail-api-go
```

---

## Appendix A — Example Rules

```yaml
delete:
  - patterns: ["promotional", "unsubscribe", "limited time offer"]
  - from_domains: ["*.marketing.com"]
  - labels: ["PROMOTIONS"]

keep:
  - from_addresses: ["github.com", "support@stripe.com"]
  - subjects: ["order shipped", "security alert"]

trash:
  - patterns: ["you have won", "dear customer"]
```

---

## Appendix B — Audit Entry

```json
{
  "run_id": "2025-01-15T08:30:00Z",
  "account": "gmail",
  "email_id": "msg-12345",
  "action": "DELETE",
  "reason": "Promotional email with 'limited time offer'",
  "confidence": 0.95,
  "timestamp": "2025-01-15T08:32:15Z"
}