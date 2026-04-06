# Pi 5 Invoice Collector — **Architecture** (Node.js + Proton Bridge on Raspberry Pi OS Lite)

> Purpose: Automatically collect invoice emails from a Proton Mail business inbox via Proton Bridge, extract key fields (date, supplier, invoice number, net, VAT, gross, currency), and deliver a monthly/quarterly “VAT pack” (CSV + PDFs). Designed to run end‑to‑end on a Raspberry Pi 5 (Raspberry Pi OS Lite, trixie‑based).

---

## 0) Goals & Non‑Goals

**Goals**

* Zero Proton-side filters (all filtering lives in code).
* Private, robust, idempotent ingestion.
* UBL‑first parsing, PDF text fallback, OCR as last resort.
* Monthly pack zip and optional quarterly pack.
* Simple ops: systemd timers, Docker for Bridge, Node.js worker.

**Non‑Goals**

* No third‑party parsing SaaS.
* No mutation of mailbox state (no auto-replies, labels, deletes).
* No accounting postings; output is normalized CSV + PDFs ready for your bookkeeping tool or accountant.

---

## 1) High‑Level Overview

**Components**

1. **Proton Bridge (Docker, ARM64)** — exposes local IMAP/SMTP for the Proton mailbox.
2. **Invoice Collector (Node.js)** — a small service with CLI commands: `ingest`, `pack`, `verify`.
3. **Storage** — PDFs/UBLs on disk, SQLite for state, CSV per month, ZIP bundles.
4. **Schedulers** — systemd timers for daily ingest and monthly pack.

**Data Flow**

1. Collector connects to Bridge IMAP on `127.0.0.1:12143` (STARTTLS).
2. Performs incremental search using last seen UID (or date range for backfill).
3. Fetches minimal headers + BODYSTRUCTURE; downloads attachments only if relevant.
4. For each mail: detect UBL XML → parse; else extract PDF text → parse via templates; else OCR → parse.
5. Normalize & write/append record to monthly CSV; rename/store PDFs.
6. On the 1st, create `VAT-YYYY-MM.zip` with CSV + PDFs.

---

## 2) Runtime & OS Assumptions

* Raspberry Pi 5 (ARM64), Raspberry Pi OS Lite (Debian trixie).
* Node.js **≥ 20 LTS** (22 LTS recommended).
* Docker / containerd available for the Bridge container.
* Timezone **Europe/Amsterdam** (system time configured accordingly).

---

## 3) Directory Layout

```
/srv/invoice-collector
├── app/                      # Node.js project
│   ├── src/
│   │   ├── cli.ts
│   │   ├── imap/
│   │   │   ├── client.ts
│   │   │   └── selectors.ts
│   │   ├── parsing/
│   │   │   ├── ubl.ts
│   │   │   ├── pdf.ts
│   │   │   └── templates.ts
│   │   ├── pipeline/
│   │   │   ├── ingest.ts
│   │   │   ├── normalize.ts
│   │   │   └── pack.ts
│   │   ├── store/
│   │   │   ├── sqlite.ts
│   │   │   └── fs.ts
│   │   ├── util/
│   │   │   ├── logging.ts
│   │   │   ├── config.ts
│   │   │   └── dates.ts
│   │   └── types.ts
│   ├── package.json
│   ├── tsconfig.json (optional)
│   └── .env.example
├── config/
│   └── default.yaml          # regex rules, vendor list, paths, concurrency, etc.
├── templates/
│   └── vendors/              # YAML vendor templates for PDF regex extraction
│       ├── adobe.yaml
│       ├── jetbrains.yaml
│       └── notion.yaml
├── data/
│   ├── state.sqlite          # idempotency & offsets
│   ├── storage/
│   │   ├── pdfs/2025-10/*.pdf
│   │   └── ubl/2025-10/*.xml
│   └── output/
│       ├── csv/invoices_2025-10.csv
│       └── zips/VAT-2025-10.zip
└── ops/
    ├── docker-compose.yml    # Bridge container
    ├── systemd/
    │   ├── invoices-ingest.service
    │   ├── invoices-ingest.timer
    │   ├── invoices-pack.service
    │   └── invoices-pack.timer
    └── README-ops.md
```

---

## 4) Proton Bridge (Docker) on Pi OS

**Compose (bind to loopback only):**

```yaml
version: "3.9"
services:
  proton-bridge:
    image: ghcr.io/videocurio/proton-mail-bridge:latest
    container_name: protonmail_bridge
    restart: unless-stopped
    # Persist Bridge keychain/state
    volumes:
      - /srv/proton-bridge:/root
    # Expose locally only
    ports:
      - "127.0.0.1:12143:143"   # IMAP (STARTTLS)
      - "127.0.0.1:12025:25"    # SMTP (STARTTLS)
```

**First-run login:**

```bash
cd /srv/invoice-collector/ops
sudo docker compose up -d
sudo docker exec -it protonmail_bridge bash
bridge --cli
# login → 2FA → wait for initial sync → `info` to see local IMAP creds/ports
exit
```

**Security notes**

* Ports are loopback‑bound; nothing exposed on LAN.
* The `/srv/proton-bridge` volume holds Bridge state & keyring; back it up and protect permissions.

---

## 5) Node.js Service — Libraries & Rationale

* **IMAP**: [`imapflow`] — robust, modern, supports BODYSTRUCTURE streaming.
* **MIME**: [`mailparser`] for safe header/body/attachment handling when needed.
* **XML**: [`fast-xml-parser`] to parse UBL 2.1/2.3.
* **PDF text**: Prefer shelling out to `pdftotext` (Poppler) via `child_process`/`execa` for accuracy; fallback [`pdf-parse`] for simple cases.
* **OCR**: Call `ocrmypdf` only when no text layer is found.
* **State**: [`better-sqlite3`] (sync, simple, reliable on ARM).
* **CLI**: [`commander`] or [`yargs`].
* **Zip**: [`archiver`].
* **Logging**: [`pino`] (JSON logs; fast on ARM).
* **Concurrency control**: [`p-limit`].

> All heavy lifting (OCR) is optional and queued; most invoices parse via UBL or PDF text without OCR.

---

## 6) Filtering Logic (in code, not in Proton)

**Message selection**

* Use mailbox `All Mail` (or `INBOX`, configurable).
* Maintain **last seen UID** per mailbox in SQLite.
* **Incremental**: `UID SEARCH UID {last+1}:*`.
* **Backfill**: `SEARCH SINCE <DD-Mon-YYYY> BEFORE <DD-Mon-YYYY>`.

**Early discard rules (headers/BODYSTRUCTURE only)**

* No attachments → skip unless HTML contains explicit “invoice” keywords (configurable) and you want to capture link‑only notices as `PDF_MISSING`.
* Attachments that are neither `application/pdf` nor XML → skip (unless vendor‑whitelisted).

**Positive signals**

* Subject or body contains (case‑insensitive, configurable): `invoice|factuur|receipt|btw|vat|tax invoice|credit note`.
* From address domain in vendor allow‑list.
* PDF or UBL XML present.

**Negative signals**

* `proforma|quote|offerte|order confirmation` → tag as `NON_INVOICE` (optional).

---

## 7) Parsers (priority order)

### 7.1 UBL Extractor (preferred)

* Detect by XML root `Invoice` / `CreditNote` and UBL namespaces.
* Map fields to canonical schema:

  * `issue_date` (YYYY-MM-DD)
  * `supplier_name`, `supplier_vat_id`
  * `customer_name`, `customer_vat_id` (optional)
  * `invoice_number`
  * `currency`
  * `net_total`, `tax_total`, `gross_total`
  * `tax_breakdown[]` (rate, amount)
  * `is_credit_note`
* Validate numerics (sum lines ≈ totals; tolerance configurable).

### 7.2 PDF Text Extractor

* Run `pdftotext -layout input.pdf -` and parse plaintext.
* Apply **vendor templates** (YAML): patterns for invoice number, date, currency, totals; context via anchor phrases (e.g., company name/VAT).
* If no match, try **generic fallback** regex heuristics.

### 7.3 OCR Fallback (rare)

* If `pdftotext` yields empty/garbled text, run `ocrmypdf --skip-text --rotate-pages input.pdf output.pdf` then re‑run text extraction.

**Output Record (normalized)**

```
{
  message_id,
  received_at,
  supplier_name,
  supplier_vat_id,
  invoice_number,
  issue_date,
  currency,
  net_total, tax_total, gross_total,
  tax_breakdown: [{ rate, amount }],
  pdf_path,
  ubl_path,
  status: OK | PDF_MISSING | PARSE_FAILED | NON_INVOICE | DUPLICATE
}
```

---

## 8) Idempotency & State

**SQLite tables**

```sql
CREATE TABLE IF NOT EXISTS checkpoints (
  mailbox TEXT PRIMARY KEY,
  last_uid INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS attachments (
  message_id TEXT,
  part_id TEXT,
  sha256 TEXT,
  stored_path TEXT,
  PRIMARY KEY (message_id, part_id)
);

CREATE TABLE IF NOT EXISTS records (
  message_id TEXT PRIMARY KEY,
  issue_date TEXT,
  invoice_number TEXT,
  supplier_name TEXT,
  currency TEXT,
  net_total REAL, tax_total REAL, gross_total REAL,
  status TEXT,
  pdf_path TEXT, ubl_path TEXT
);
```

* Before saving, compute **sha256** of each attachment; skip if already seen.
* Records are **upserted** so re‑runs are safe.

---

## 9) Files, Naming & CSV

* PDF rename: `YYYYMMDD_supplier_invoice-<number>.pdf` (ASCII‑only; spaces → dashes).
* UBL rename: same stem, `.xml`.
* CSV per month: `data/output/csv/invoices_YYYY-MM.csv` with columns:
  `issue_date,supplier,invoice_no,currency,net,vat,gross,pdf_path,ubl_path,status`
* ZIP pack per month: `data/output/zips/VAT-YYYY-MM.zip` containing the CSV + all PDFs for that month.

---

## 10) CLI Commands

```bash
# Backfill a quarter (inclusive start, exclusive end)
node app/dist/cli.js ingest --start 2025-07-01 --end 2025-10-01

# Regular daily incremental ingest
node app/dist/cli.js ingest

# Build a monthly pack
node app/dist/cli.js pack --month 2025-10

# Build a quarterly pack
node app/dist/cli.js pack --quarter 2025Q3

# Verify recent records, list parse failures
node app/dist/cli.js verify --since 2025-10-01
```

**Flags (selected)**

* `--mailbox All Mail|INBOX` (default: All Mail)
* `--concurrency 4` (I/O bound; 2–4 is fine on Pi 5)
* `--no-ocr` (disable OCR pass)
* `--vendors templates/vendors` (override templates dir)

---

## 11) Configuration (YAML example)

```yaml
# config/default.yaml
paths:
  data: /srv/invoice-collector/data
  storage: /srv/invoice-collector/data/storage
  output: /srv/invoice-collector/data/output
  templates: /srv/invoice-collector/templates/vendors

mail:
  host: 127.0.0.1
  port: 12143    # IMAP via Bridge
  secure: false  # STARTTLS (upgrade) via imapflow
  user: "local-bridge-username"
  pass: "local-bridge-password"
  mailbox: "All Mail"

rules:
  positive_subject: ["invoice", "factuur", "receipt", "vat", "btw", "tax invoice", "credit note"]
  negative_subject: ["proforma", "quote", "offerte", "order confirmation"]
  vendor_domains: ["adobe.com", "jetbrains.com", "notion.so"]

parsing:
  enable_ocr: true
  ocr_cmd: "ocrmypdf"
  pdftotext_cmd: "pdftotext"
  tolerance: 0.02  # allowed total rounding diff

packaging:
  when: monthly  # monthly packs by default
  zip_name: "VAT-{{YYYY-MM}}.zip"

logging:
  level: info
  pretty: false  # JSON logs
```

**Secrets**

* Keep Bridge creds in `.env` or OS keyring if you prefer; config loader merges `.env` → YAML → CLI flags.

---

## 12) Vendor Templates (PDF regex, YAML)

```yaml
# templates/vendors/adobe.yaml
vendor: Adobe
match:
  any_of:
    - header: "Adobe Inc"
    - vat_id: "^EU\d{9}$"
extract:
  invoice_number: "Invoice\s*No\.?\s*([A-Z0-9-]+)"
  issue_date: "Invoice Date\s*:\s*(\d{4}-\d{2}-\d{2}|\d{2}/\d{2}/\d{4})"
  currency: "Currency\s*:\s*([A-Z]{3})"
  net_total: "Subtotal\s*([0-9.,]+)"
  tax_total: "Tax\s*([0-9.,]+)"
  gross_total: "Total\s*([0-9.,]+)"
normalize:
  date_format_hint: ["YYYY-MM-DD", "DD/MM/YYYY"]
```

> Add one small YAML per frequent vendor; accuracy becomes near‑perfect after 1–2 runs.

---

## 13) Scheduling (systemd)

**Daily ingest (08:10 CET/CEST)**

```ini
# ops/systemd/invoices-ingest.service
[Unit]
Description=Invoice Collector – Daily Ingest
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/srv/invoice-collector
Environment=NODE_ENV=production
ExecStart=/usr/bin/node app/dist/cli.js ingest
User=invoices
Group=invoices

# ops/systemd/invoices-ingest.timer
[Unit]
Description=Run invoice ingest daily

[Timer]
OnCalendar=*-*-* 08:10:00 Europe/Amsterdam
Persistent=true

[Install]
WantedBy=timers.target
```

**Monthly pack (1st at 08:20)**

```ini
# ops/systemd/invoices-pack.service
[Unit]
Description=Invoice Collector – Monthly Pack
After=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/srv/invoice-collector
Environment=NODE_ENV=production
ExecStart=/usr/bin/node app/dist/cli.js pack --month %Y-%m
User=invoices
Group=invoices

# ops/systemd/invoices-pack.timer
[Unit]
Description=Run monthly pack on the 1st

[Timer]
OnCalendar=*-*-01 08:20:00 Europe/Amsterdam
Persistent=true

[Install]
WantedBy=timers.target
```

**Enable timers**

```bash
sudo cp -a ops/systemd/*.service ops/systemd/*.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now invoices-ingest.timer invoices-pack.timer
```

---

## 14) Observability

* **Logs**: JSON to stdout/journald (pino). Include `message_id`, `uid`, `status`, `duration_ms`.
* **Metrics (optional)**: expose a simple `/metrics` (Prometheus) with counters for parsed OK/failed/OCR‑used.
* **Health**: write `data/health.json` with last ingest timestamp and counts.

---

## 15) Security & Privacy

* Bridge ports bound to **127.0.0.1** only.
* Dedicated system user `invoices` with ownership of `/srv/invoice-collector`.
* Regular OS updates; lock down `/srv/proton-bridge` perms.
* Consider encrypted filesystem for `data/` if the Pi is portable.
* No exfiltration: all processing local; optional email of pack uses Bridge SMTP to self.

---

## 16) Performance Notes (Pi 5)

* Use small concurrency (2–4). Most time is network/IO; CPU spikes only with OCR.
* Limit OCR to PDFs with empty text layer; queue one at a time.
* `pdftotext` is fast on ARM; prefer it over pure‑JS parsing for quality.

---

## 17) Testing Strategy

* Fixture set:

  * 3× UBL invoices (EUR, USD, credit note).
  * 3× text‑PDF invoices (different vendors/fonts).
  * 1× scanned PDF requiring OCR.
  * 1× non‑invoice (proforma/quote) → should be flagged.
* Golden CSV snapshots per month for regression.
* Unit tests for UBL mapper and template extraction.

---

## 18) Migration Path (Go/Rust later)

* Keep vendor templates and normalization schema language‑agnostic (YAML + JSON schema).
* SQLite schema remains; only the worker binary changes.
* Shared `config/default.yaml` contract; CLI flags mirrored.

---

## 19) Maintenance & Updates

* Pin Bridge container tag once you confirm a working version.
* Update templates as vendors change layouts (cheap, isolated edits).
* Quarterly review of negative/positive keyword lists.
* Backup `data/` and `/srv/proton-bridge/` weekly.

---

## Appendix A — Minimal `package.json`

```json
{
  "name": "invoice-collector",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "build": "tsc -p .",
    "start": "node app/dist/cli.js ingest"
  },
  "dependencies": {
    "archiver": "^6",
    "better-sqlite3": "^9",
    "commander": "^12",
    "execa": "^9",
    "fast-xml-parser": "^4",
    "imapflow": "^1",
    "mailparser": "^3",
    "p-limit": "^5",
    "pino": "^9"
  },
  "devDependencies": {
    "typescript": "^5"
  }
}
```

## Appendix B — Example IMAP minimal client (sketch)

```ts
// src/imap/client.ts
import { ImapFlow } from "imapflow";

export async function withImap<T>(cfg, fn: (client: ImapFlow) => Promise<T>): Promise<T> {
  const client = new ImapFlow({
    host: cfg.mail.host,
    port: cfg.mail.port,
    secure: false,
    auth: { user: cfg.mail.user, pass: cfg.mail.pass },
    tls: { rejectUnauthorized: true }
  });
  await client.connect();
  try {
    await client.mailboxOpen(cfg.mail.mailbox);
    return await fn(client);
  } finally {
    await client.logout();
  }
}
```

## Appendix C — Example Backfill Command

```bash
# Q3 2025
node app/dist/cli.js ingest --start 2025-07-01 --end 2025-10-01
node app/dist/cli.js pack --quarter 2025Q3
```

---
