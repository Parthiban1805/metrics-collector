# Platform Architecture: Unified Infrastructure, AI Usage and Security Monitoring

## 1. Overview
**Scope (current): internal platform for our own company.** We use it to manage our own servers, GPU/AI infrastructure, AI usage and security. The design stays *multi-tenant-ready* (every table keeps a `tenant_id`, with one tenant today), so selling it later does not require a rewrite. Items that only matter for external customers are marked **(SaaS later)**.

An agent installed on each VM gives one place to see:
- **Infrastructure health:** CPU, RAM, disk, network, GPU, load, swap and per-core metrics, collected every second as described in [Frequency of Data Collection](../data-collection-frequency/Frequency%20of%20Data%20Collection.md).
- **AI usage:** self-hosted LLMs (Ollama/vLLM on GPUs) and cloud AI APIs (OpenAI, Anthropic, Gemini). It covers tokens, cost, latency, errors and safety.
- **Security:** the Observe → Audit → Detect → Scan → Respond model from [Linux Server Monitoring and Security Visibility](../linux-server-monitoring-security/Linux_Server_Monitoring_and_Security_Visibility.md), using Netdata, auditd, Wazuh, ClamAV and Fail2Ban.
- **AI-assisted analysis:** incidents explained in plain English, triaged automatically, and answerable by natural-language questions.

**Key decisions**

| Topic | Decision |
|---|---|
| Collection | Hybrid. Netdata for metrics, plus our own lightweight agent for AI, security events and heartbeat |
| Backend | Python, FastAPI |
| Scale | 100+ servers; one tenant (our company) now, many tenants **(SaaS later)** |
| AI usage scope | Self-hosted and cloud AI |

## 2. Guiding Principle: Three Intelligence Tiers

| Tier | Engine | Runs on | Job |
|---|---|---|---|
| **Tier 0: Code** | Rules, statistics, classical ML | Every data point | Detection, thresholds, baselines, forecasts, cost math, correlation |
| **Tier 1: Decision model** | **Jev** (TypeSafe System One model); fallback Claude Haiku 5.5 with structured outputs | Every alert or event | Fast typed decisions with calibrated probabilities: triage, classification, grouping, routing, guardrails |
| **Tier 2: Text LLM** | Claude Sonnet 5.5 (default); Claude Opus 5.5 (critical); self-hosted (private tenants) | Per incident or per question | Explanations, root-cause narrative, natural-language Q&A, digests |

**Rules**
- No model ever reads the raw 1-second stream. Models see only compact fact bundles that Tier 0 has already computed.
- Models never act on their own. Automated response stays with Fail2Ban, Wazuh, or human-approved runbooks.
- Every model call is redacted, capped per tenant, cached, and stored with the incident it belongs to.

## 3. Complete Architecture

```
┌──────────────────────────── CUSTOMER ENVIRONMENT ────────────────────────────┐
│ Linux VM / bare metal / container host                                        │
│  ├─ Netdata ............ 1s metrics (system.*, cpu.*, nvidia_smi.*, apps.*)   │
│  ├─ auditd ............. syscall / file / user audit trail                    │
│  ├─ Wazuh agent ........ FIM, rootkit, SCA, vuln → Wazuh manager              │
│  ├─ ClamAV ............. scheduled + on-upload scans                          │
│  ├─ Fail2Ban ........... ban/unban events                                     │
│  ├─ Ollama / vLLM ...... local models (optional)                              │
│  └─ ligament-agent (ours)                                                     │
│       collectors: netdata · ollama · gpu · fail2ban · clamav · authlog ·      │
│                   auditd · processes · packages/changes · docker/k8s · ports  │
│       local buffer (disk queue) → batch every 10s → HTTPS/gzip, mTLS + key    │
│                                                                               │
│ Customer apps ──► LiteLLM AI Gateway (we host or tenant self-hosts)           │
│                    per-request tokens, cost, latency, user/team tags          │
└───────────────────────────────────┬───────────────────────────────────────────┘
                                    │
┌──────────────────────────── PLATFORM (internal, SaaS-ready) ─────────────────────────────────┐
│ Edge: API gateway / WAF · rate limit per tenant · agent auth (mTLS + API key) │
│                                                                               │
│ Ingestion                                                                     │
│  ├─ /ingest/metrics  /ingest/events  /ingest/ai-usage  (FastAPI, stateless)   │
│  ├─ OpenTelemetry / Prometheus remote-write endpoint (bring-your-own data)    │
│  └─ → Queue: Redis Streams (start) → NATS JetStream / Kafka (scale)           │
│                                                                               │
│ Connectors (pull workers)                                                     │
│  ├─ Wazuh manager API / indexer      ├─ OpenAI Usage & Cost API               │
│  ├─ Anthropic Admin usage/cost API   ├─ Gemini / Vertex billing export        │
│  ├─ Cloud providers (AWS/Azure/GCP): VM inventory, tags, billing              │
│  └─ LiteLLM spend logs                                                        │
│                                                                               │
│ Processing workers                                                            │
│  ├─ Normalizer → canonical metric/event schema, tenant tagging                │
│  ├─ Rollups (1s → 1m → 1h) via Timescale continuous aggregates                │
│  ├─ TIER 0  Rules engine (YAML rules) · Correlation engine · Baselines ·      │
│  │          Anomaly (EWMA/z-score, Isolation Forest) · Forecasts · Cost calc  │
│  ├─ Incident manager: dedup, grouping, lifecycle, maintenance windows         │
│  └─ Notification service: email, Slack, Teams, PagerDuty, webhooks, push      │
│                                                                               │
│ AI Service                                                                    │
│  ├─ Redactor (secrets, tokens, IPs/hostnames per policy)                      │
│  ├─ Bundle builder (metric stats, top processes, related events, baseline)    │
│  ├─ TIER 1  Decision router → Jev │ fallback Haiku 5.5 (same Pydantic schema)│
│  ├─ Confidence gate: p≥0.85 auto · 0.5–0.85 → Tier 2 · <0.5 → human queue    │
│  ├─ TIER 2  Text LLM → Sonnet 5.5 │ Opus 5.5 (critical) │ self-hosted         │
│  ├─ "Ask" agent: tool-calling over read-only platform API (no raw SQL)        │
│  └─ AI meter: our own model spend, per-tenant caps, result cache              │
│                                                                               │
│ Storage                                                                       │
│  ├─ TimescaleDB ...... metrics hypertables (raw 48h, 1m 30d, 1h 1y)           │
│  ├─ PostgreSQL ....... tenants, users, hosts, events, incidents, AI usage,    │
│  │                     rules, runbooks, AI outputs (Row-Level Security)       │
│  ├─ OpenSearch/Loki .. selected logs + full-text search (optional module)     │
│  ├─ Object storage ... reports, exports, cold archive (S3-compatible)         │
│  └─ Redis ............ cache, sessions, rate limits                           │
│                                                                               │
│ API & UI                                                                      │
│  ├─ Public REST API + webhooks (tenant-scoped tokens)                         │
│  ├─ Next.js dashboard (ECharts): Fleet · Host · AI Usage · Security ·         │
│  │   Incidents · Ask · Reports · Settings                                     │
│  └─ Auth: JWT, SSO (SAML/OIDC), RBAC (owner/admin/operator/viewer), audit log │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Data flows
1. **Metrics.**
   - Netdata produces 1s data.
   - The agent reads the local Netdata API, batches it every 10s and sends it to the ingest endpoint and queue.
   - The normalizer writes to TimescaleDB, and the rollups and the rules engine run on top.
2. **Security events.**
   - The agent tails Fail2Ban, ClamAV, auth.log and auditd and sends them to the events endpoint.
   - The Wazuh connector pulls alerts.
   - The correlation engine turns these into incidents.
3. **AI usage.**
   - The gateway sends per-request records.
   - Provider admin APIs are polled hourly and used as the billing truth; the two sources are reconciled.
   - The Ollama collector provides local model usage, and GPU metrics come from Netdata.
4. **Incident intelligence.**
   - A Tier 0 rule fires and opens an incident. The bundle builder and redactor prepare the facts.
   - Tier 1 adds labels with probabilities.
   - Tier 2 writes the summary when the incident is severe or Tier 1 confidence is low.
   - Notifications follow.
5. **Ask.**
   - The user asks a question. Tier 1 routes the intent.
   - Tier 2 calls read-only tools (get_metrics, list_events, top_processes, get_ai_usage) and answers with links and charts.

### 3.2 Core data model (PostgreSQL / Timescale)
| Table | Key fields |
|---|---|
| `tenants`, `users`, `memberships` | plan, limits, roles, SSO config |
| `assets` (was `hosts`) | tenant_id, type, hostname, labels, cloud ids, owner, criticality, agent_version, last_seen (see §10.1) |
| `metrics` (hypertable) | tenant_id, host_id, metric (`system.cpu.user`…), ts, value |
| `events` | tenant_id, host_id, source (wazuh/auditd/fail2ban/clamav/agent), type, severity, payload, ts |
| `incidents` | tenant_id, status, severity, rule_id, started/resolved, linked events, AI labels + probabilities, AI summary |
| `ai_usage` | tenant_id, source (gateway/provider/ollama), provider, model, team/app/user tag, tokens in/out, cached tokens, cost, latency, status, ts |
| `ai_decisions` | incident/event id, decision type, provider (jev/haiku), label, probabilities, latency, cost |
| `rules`, `runbooks`, `notification_channels`, `maintenance_windows`, `budgets` | tenant config |
| `audit_log` | every user/API action on the platform itself |

Every table carries `tenant_id` and is protected by Postgres Row-Level Security.

## 4. What Is Programmatic vs. AI

### Tier 0: Programmatic
| Area | Logic |
|---|---|
| Thresholds | CPU > 90% for 5m, RAM > 90%, swap > 0 sustained, load1 > cores×1.5, iowait > 20%, GPU temp |
| Trends / forecasts | Disk-full ETA (regression), memory-leak slope, capacity forecast |
| Baselines / anomaly | Per-host hour-of-week baseline, z-score > 3, Isolation Forest |
| Processes | New binary first-seen, denylist (`xmrig`), process-count explosion |
| Correlation | CPU spike + unknown process + outbound traffic spike → *possible cryptomining*. Failed SSH spike + Fail2Ban ban → *brute force*. Wazuh FIM on /etc + auditd user → *config tampering* |
| Security | Wazuh level ≥ 10, ClamAV hit, sudoers/passwd change, new listening port |
| AI cost | tokens × price table, chargeback per team/app, budgets, token-spike anomaly, cache-hit rate, error/rate-limit rate |
| GPU efficiency | VRAM allocated while GPU is idle, tokens/sec per model, queue latency |
| Hygiene | Agent offline, cert expiry, alert dedup, maintenance windows |

### Tier 1: Jev typed decisions
| Decision (enum + probability) | Input |
|---|---|
| Alert triage: `benign / investigate / urgent` | Alert + baseline deviation + host role |
| Process verdict: `expected / unusual / suspicious / miner` | Name, cmdline, parent, user, CPU/net, first-seen |
| Incident grouping: `same_incident(id) / new` | New event + open incidents |
| Root-cause category: `workload / leak / backup / attack / misconfig / hardware` | Incident bundle |
| Routing: runbook/team (up to 255 choices) | Security event |
| Ask intent router | User question |
| AI-traffic guardrail: `safe / jailbreak / pii_leak / policy_violation` | Prompts/responses sampled at the gateway |

### Tier 2: Text LLM
| Feature | Output |
|---|---|
| Incident summary + root-cause narrative | What happened, likely cause, confidence, next steps |
| Explain security records | Plain-English explanation of auditd / Wazuh rules |
| Ask (natural-language Q&A) | Answers with charts and links, e.g. "why was web-12 slow last night?" |
| Daily / weekly digest | Narrative report per tenant (Batch API, 50% cheaper) |
| Optimization advice | Right-sizing and AI-cost recommendations from computed numbers |

## 5. Model Comparison and Choice

| Model | Price in/out per MTok | Output | Latency | Role | Risk |
|---|---|---|---|---|---|
| **Jev** (TypeSafe) | $0.042 / free | Typed enum/struct + calibrated probabilities, no free text | 70–500 ms | **Tier 1 primary** | Early access; licensing, retention and self-hosting not published; vendor-run benchmarks |
| **Claude Haiku 5.5** `claude-haiku-5-5` | $0.10 / $0.50 | Text + structured outputs | Fast | **Tier 1 fallback**, short explanations | Low |
| **Claude Sonnet 5.5** `claude-sonnet-5-5` | $2 / $10 | Text, tool use, 1M context | Medium | **Tier 2 default** | Low |
| **Claude Opus 5.5** `claude-opus-5-5` | $4 / $20 | Deepest reasoning | Slower | **Tier 2 premium**: critical/security incidents | Cost, so it is gated |
| OpenAI / Gemini | Verify current pricing | Text + JSON | Varies | Optional bring-your-own-key per tenant via LiteLLM | Extra integration |
| Self-hosted open-weight (Qwen / Llama / DeepSeek on vLLM) | GPU cost only | Text + JSON | GPU-dependent | Private / on-prem tenants | Lower quality; we run the GPUs |

**Provider abstraction.**
- `app/ai/providers/base.py` defines `DecisionModel` and `TextModel`. Implementations are `jev.py`, `claude.py` (Anthropic SDK) and `selfhosted.py`, chosen per tenant by config.
- Decisions use one Pydantic schema across providers. TypeSafe's `system-one-adapter-python` keeps LLM fallbacks compatible with that format.
- Jev becomes the default only if it beats Haiku 5.5 on our labeled eval set for accuracy, calibration, latency and cost.

**Rough cost** for 1 tenant with 100 hosts, about 20k events/day and about 50 incidents/day:

| Usage | Approx. cost |
|---|---|
| Jev on all events (about 30M input tokens) | about $1.3/day |
| Sonnet 5.5 on incidents | about $1.3/day |
| Haiku-only triage instead of Jev | about $4–5/day, and slower |

These are estimates to measure in production.

## 6. Additional Features

### Infrastructure
- **Uptime and synthetic checks:** HTTP/TCP/ping, TLS certificate expiry, DNS, from multiple regions.
- **Containers and Kubernetes:** Docker container metrics, pod restarts, OOM kills, node pressure.
- **Change tracking:** a timeline of package updates, deploys, config changes and reboots. It is correlated with incidents to answer "what changed before it broke?"
- **Capacity planning:** forecasts of when CPU, RAM, disk or GPU run out, per host and per fleet.
- **Cloud FinOps:** VM inventory and cost from AWS/Azure/GCP, idle-VM detection, right-sizing.
- **SLOs and error budgets:** per service, with burn-rate alerts and a public status page.

### AI usage
- **Chargeback and showback:** AI cost per team, app, user and project, with monthly budgets and hard or soft limits.
- **Shadow-AI detection:** spot hosts calling AI provider endpoints (e.g. `api.openai.com`) directly from network/DNS data without going through the gateway.
- **Model comparison board:** cost vs. latency vs. error rate per model, plus "switch X to a cheaper model" suggestions.
- **Prompt efficiency:** cache-hit rate, average context length trends, retry waste.
- **GPU fleet view:** which local models are loaded where, VRAM waste, utilization heatmap, consolidation advice.
- **AI safety monitoring:** Jev guardrail labels on tenant prompts and responses (jailbreak, PII leak).

### Security and compliance
- **Vulnerability inventory:** CVEs per host from Wazuh, prioritized by exposure.
- **Compliance reports:** CIS benchmark (Wazuh SCA), with SOC 2 / ISO 27001 evidence exports.
- **Exposure monitoring:** new listening ports, new users, sudoers changes, SSH key inventory.
- **Safe remediation runbooks:** human-approved one-click actions (restart service, block IP, kill process, isolate host), with every action logged.

### Platform
*(Enterprise items below are **SaaS later**.)*
- **Alert routing:** escalation policies, on-call schedules, snooze, maintenance windows, mobile push.
- **Open integrations:** Prometheus remote-write and OpenTelemetry ingestion, a Grafana-compatible query API, webhooks, Slack and Teams bots ("Ask" from chat).
- **Reports:** scheduled PDF and email reports, CSV and Parquet exports.
- **Enterprise:** SSO (SAML/OIDC), SCIM, data-residency regions, a fully self-hosted/on-prem edition with local LLMs.

## 7. Technology Stack
| Layer | Choice |
|---|---|
| Agent | Python (psutil, httpx), packaged as .deb/.rpm with a one-line `install.sh`; disk-backed buffer, auto-update |
| Metrics source | Netdata (local API / Prometheus format) |
| Security sources | auditd, Wazuh, ClamAV, Fail2Ban |
| AI gateway | LiteLLM proxy |
| API | FastAPI, Pydantic, SQLAlchemy, Alembic |
| Queue | Redis Streams, moving to NATS JetStream / Kafka |
| Workers | Python async workers (Celery / Arq), scikit-learn, statsmodels |
| Time-series | TimescaleDB; ClickHouse evaluated past about 1000 hosts |
| Relational | PostgreSQL with RLS |
| Logs (optional) | OpenSearch or Loki |
| AI | Jev API, Anthropic SDK (Haiku 5.5, Sonnet 5.5, Opus 5.5), vLLM for self-hosted |
| Frontend | Next.js, React, ECharts |
| Deploy | Docker Compose (dev), Kubernetes + Helm (prod), Terraform |
| Observability of the platform | OpenTelemetry, Prometheus, Grafana; we monitor ourselves |

## 8. Multi-Tenancy, Security and Scaling
*Internal now, one tenant. Per-tenant quotas, dedicated databases/regions and tenant AI-policy variety are **SaaS later**; `tenant_id` + RLS stay from day one.*

- **Isolation:**
  - `tenant_id` is on every row, enforced by Postgres RLS.
  - Ingest is rate-limited and quota-limited per tenant.
  - Enterprise tenants can get a dedicated database or region.
- **Agent trust:**
  - The agent uses mTLS plus a per-host API key and runs least-privilege (non-root where possible).
  - Updates are signed, and the agent makes outbound-only connections.
- **Data protection:**
  - Data is encrypted in transit and at rest.
  - Secrets are redacted before any AI call.
  - Each tenant has an AI policy: off, cloud-allowed, or self-hosted-only.
- **Retention:** raw 1s data for 48h, 1-minute rollups for 30 days, 1-hour rollups for 1 year, then a cold archive in object storage. Retention is configurable per plan.
- **Scaling:**
  - Ingest and workers are stateless and scale horizontally.
  - The queue is partitioned by tenant.
  - Rough sizing: 100 hosts × about 60 series is about 6k points/s, which one Timescale node handles. Shard or move to ClickHouse as fleets grow.

## 9. Repository Layout
```
agent/              ligament-agent + collectors + packaging
backend/
  app/api/          auth, tenants, hosts, metrics, events, incidents, ai_usage, ask, webhooks
  app/ingest/       push endpoints, OTel/Prometheus receivers, queue producers
  app/connectors/   wazuh, openai_usage, anthropic_usage, gemini_billing, litellm, cloud_providers
  app/workers/      normalizer, rollups, notifications
  app/rules/        YAML rules, correlation engine
  app/analytics/    baselines, anomaly, forecast, finops
  app/ai/           redaction, bundle builder, confidence gate, ask tools
    decisions/      Pydantic decision schemas
    providers/      base.py, jev.py, claude.py, selfhosted.py
    prompts/        frozen, cache-friendly system prompts
    evals/          labeled datasets + scoring
  migrations/
frontend/           Next.js dashboard
deploy/             docker-compose, helm, terraform
docs/               existing docs + this architecture + metric schema
```

## 10. Internal Compliance Module (our own SOC 2 / ISO 27001)
**Scope:** makes *our own company* audit-ready (SOC 2 / ISO 27001). It runs on this same internal platform. If the platform is later sold as SaaS, customers will ask for SOC 2 too, so this work carries over.

```
Asset sources (agent, Wazuh, osquery, AWS, GitHub, Google Workspace)
        │
        ▼
Asset Inventory (single source of truth)
        │
        ├─► Security baselines  (Wazuh SCA / CIS, Prowler for AWS)
        ├─► Vulnerability mgmt  (Wazuh CVEs + remediation SLAs)
        └─► Identity & access   (Google Workspace / GitHub / AWS IAM)
                    │
                    ▼
        Control checks (Tier 0 code, deterministic)
                    │
           ┌────────┴────────┐
           ▼                 ▼
         PASS               FAIL
     evidence saved     ticket + owner
           │                 │
           └────────┬────────┘
                    ▼
     Evidence store (S3 Object Lock, hashed, timestamped)
                    │
                    ▼
        Control matrix + posture dashboard
```

### 10.1 Asset inventory
- The `hosts` table becomes a general **`assets`** table covering servers, GPU/edge devices, laptops, cloud resources, databases, SaaS apps and repositories.
- Fields: owner, department, environment, criticality, data classification, OS/version, last seen, last scan, compliance status, EOL status, and links to related assets.
- **Criticality** has four levels: Critical, High, Medium and Low. It sets how often each asset is checked, and it is also passed to Jev triage as an input.

### 10.2 Controls and checks
| Area | Source | Example checks |
|---|---|---|
| Servers | Wazuh SCA, our agent | SSH root login off, password auth off, firewall on, auditd on, NTP, patches |
| Laptops | Wazuh agent + osquery/Fleet (no MDM built by us) | Disk encryption, screen lock, OS version, EDR running, local admins |
| AWS | Prowler (ships SOC 2/ISO mappings), CloudTrail | Public S3, 0.0.0.0/0 security groups, root MFA, unused keys, encryption, logging, backups |
| SaaS / identity | Google Workspace, GitHub, AWS IAM APIs | MFA coverage, admin list, inactive accounts, offboarded users still active |
| Vulnerabilities | Wazuh | Remediation SLA: Critical 7d, High 14d, Medium 30d, Low 90d (set by our policy) |

**Rules for checks**
- All checks are Tier 0 code, so the same input always gives the same result, which is what auditors need.
- AI only assists:
  - Suggesting criticality.
  - Drafting policies.
  - Summarizing evidence.
  - Flagging unused privileged access.

### 10.3 Exceptions
Every exception has:
- Asset and control
- Reason and risk
- Approver
- Compensating control
- Mandatory expiry date

An expired exception automatically turns the control back to failing.

### 10.4 Access lifecycle
- **Joiner, mover and leaver:** each step is a checklist item that records evidence.
- **Leaver:** disable the Google account, revoke GitHub, AWS and VPN access, and recover the laptop.
- **Access reviews:** monthly for Critical systems and quarterly for others. The result is stored as evidence.

### 10.5 Evidence and control matrix
- Common controls are mapped once to **SOC 2 (Security criteria first)** and **ISO 27001**, using the Secure Controls Framework as the crosswalk.
- Evidence consists of API exports, scan reports, access reviews, incident records and policy acknowledgements.
  - Every item is timestamped, hashed and stored write-once.
  - Screenshots are used only when there is no other option.
- The posture dashboard shows:
  - Assets: compliant, non-compliant and unknown
  - Coverage: MFA, encryption, EDR and patching
  - Open and expired exceptions
  - Readiness for each framework

### 10.6 Build vs. buy
| Part | Approach |
|---|---|
| Technical checks (servers, AWS, GitHub, Workspace) | **Build.** They reuse this platform's agent, Wazuh, connectors and rules engine |
| Policies, security training, vendor reviews, auditor portal | **Use a simple tracker or a GRC tool.** Building these internally is not worth it |
| Policies themselves (access control, vulnerability, incident response, backup, change management, acceptable use) | Written by the organization; Claude can draft them for review |

## 11. AI Data Sandbox & Access Control
**Principle:** the AI never holds credentials or database access. Every piece of data it sees passes through a checkpoint that enforces the *asking user's* permissions. This is enforced in the architecture, not by prompt instructions.

**Internal-first scope:** the main risks today are (a) employees seeing data outside their role, (b) secrets in logs reaching an external model, and (c) injected text in logs steering the AI. Cross-customer isolation is a **(SaaS later)** concern, although `tenant_id` + RLS is kept because it is cheap.

```
User (JWT: role, asset scopes)
   │ question
   ▼
AI Service ──► LLM (Claude / Jev)        ← no DB creds, no network tools, no write tools
   │  model requests a tool call
   ▼
Read-only tool API  (user_id injected server-side, never from the model;
   │                 role check in code, row/time caps)
   ▼
Redactor (secrets, tokens) → back to the LLM
   │
   └─► AI audit log: who asked, which tools, which rows, what was returned
```

### 11.1 v1 must-haves
| Control | How it works |
|---|---|
| Permission inheritance | The Ask agent acts as the logged-in user and can never see more than that user can. User ID is injected server-side; Postgres RLS (`tenant_id`) is kept as a backstop |
| Typed, limited tools | Read-only allowlisted tools only, with Pydantic-typed parameters and caps on rows and time range. No SQL, no shell, no write actions |
| Secret redaction | Passwords, tokens, keys and auditd command lines are stripped before any model call |
| Untrusted-data wrapping | Logs and events are passed as clearly marked data, never as instructions. With no write tools, a successful injection can only produce a wrong answer, not an action |
| AI access log | Each answer and incident records exactly which data the AI accessed ("What did the AI see?"). Also SOC 2 evidence (§10.5) |
| AI policy | `off` / `cloud` (zero-data-retention providers only) / `self-hosted only`, set per tenant (one today) |
| Small AI red-team test set in CI | Cross-role and prompt-injection prompts; the build fails on any leak |

### 11.2 Later, when there is a reason
| Control | Add when |
|---|---|
| Data classification + tokenization (`HOST_7`, `USER_3`) | Prompts go to cloud models for sensitive assets, or a customer asks. Until then simple redaction is enough |
| Break-glass access, two-person approval | Remediation runbooks exist for Critical assets |
| Canary records (honeytokens) | After launch; a cheap alarm if fake secrets ever show up in a prompt |
| Policy engine (OPA/Cedar) | Rules start varying per customer **(SaaS later)**; plain role checks are enough for a few internal roles |
| Container isolation (gVisor/Firecracker) | Only if we add an AI feature that *runs code*. The current design calls fixed tools, so there is nothing to isolate |
| Explainable denials | Nice UX; add alongside the Ask UI |

### 11.3 Code location
```
backend/app/ai/sandbox/
  tools/           read-only typed tools; server-side user injection
  redact.py        secret/token stripping
  audit.py         per-call AI access log
  redteam/         injection + cross-role attack prompts for CI
  (later) classify.py, tokenizer.py, canary.py, policy/
```
