---
name: ai-infra-provider-researcher
description: Audits the codebase layer-by-layer (Frontend, Backend, Database/Postgres, Queues, AI/GPU), calculates exact bandwidth and RAM needs, and searches live providers (e.g., Supabase vs Neon vs Railway vs Hetzner) to recommend the most cost-effective, non-overkill stack.
tools:
  - Read
  - Glob
  - Grep
  - Bash
  - WebSearch
  - WebFetch
---

# Skill: Modular Codebase Audit & Live Provider Search

## Objective
Audit the repository, calculate exact hardware/bandwidth needs, and run **live web searches** (`WebSearch` / `WebFetch`) to compare the best providers **for each specific architectural layer** detected in the project. Never recommend overkill infrastructure.

---

## Phase 1: Layer-by-Layer Codebase Audit
Scan the repository (`package.json`, `.env.example`, `docker-compose.yml`, ORM schemas, SQL migrations, API routes) and classify every active layer:

1. **Database & Vector Layer:**
   - Detect engine: PostgreSQL, MySQL, SQLite, MongoDB, Redis.
   - Detect extensions & usage: Is `pgvector` used? What vector dimension (e.g., 1536, 3072) and index type (`HNSW` vs `IVFFlat`)?
   - Detect BaaS features: Does the code rely on Supabase Auth, RLS, Storage, or Realtime, or is it standard SQL via Prisma/Drizzle/`pg`?
   - Detect query cadence: Are there background workers/cron jobs keeping the DB awake 24/7?
2. **Backend / API / Worker Layer:**
   - Detect framework: Fastify, Express, NestJS, Python FastAPI, Go, n8n, BullMQ.
   - Detect statefulness: Long-running WebSockets/SSE, cron jobs, or stateless HTTP?
3. **Frontend / Edge Layer:**
   - Detect framework: Next.js (SSR vs Static), Vite/SPA, Remix, Astro.
4. **AI / Model Inference Layer:**
   - External API calls (OpenAI, Anthropic, Gemini) vs. self-hosted weights/embeddings.

---

## Phase 1.5: The Ponytail Ladder (Collapse Before You Shop)
Stop at the first rung that holds for the calculated load. Do NOT search providers for a layer you can delete or merge:
1. **Does this layer need to exist separately? (YAGNI)**
   - Background queue (`BullMQ` / `Redis`) with low job volume? -> Collapse into Postgres (`pg-boss` or `FOR UPDATE SKIP LOCKED`).
   - Separate vector DB (`Pinecone` / `Qdrant`) under 1M vectors? -> Collapse into existing PostgreSQL via `pgvector`.
   - Separate Frontend + Backend hosts causing cross-cloud egress? -> Co-locate on the same PaaS private network or single VPS.
2. **Does `pgvector` even need an HNSW index yet?**
   - Under 20,000 vectors, sequential scan or a lightweight index avoids multi-GB RAM bloat entirely. State this if true.
3. **Single-Box Short-Circuit:**
   - If total calculated RAM is under 4 GB, RPS is under 100, and high availability is not explicitly required, compare **1 Co-located PaaS vs. 1 Lean VPS** instead of running a 12-provider matrix.

---

## Phase 2: Bandwidth, RAM & Vector Sizing Math
Calculate and explicitly state these metrics before searching providers:

1. **Peak RPS & Monthly Bandwidth (GB):**
   - `Monthly Egress (GB) = (Daily Requests * Avg Payload KB * 30) / 1,048,576`
2. **PostgreSQL & `pgvector` RAM Calculation (If Postgres is detected):**
   - Standard Postgres overhead + active connections: `~512MB - 1GB`
   - `pgvector` HNSW RAM formula: `~ (Number of Vectors * Dimensions * 4 bytes) * 1.5`
   - *Crucial Check:* Ensure the recommended database tier has enough RAM to hold the HNSW index in memory without disk thrashing.

---

## Phase 3: Component-Specific Live Provider Search (MANDATORY)
**CRITICAL RULE:** For EVERY layer detected in Phase 1, you must use `WebSearch` and `WebFetch` to compare at least **3 competing providers** using current-year pricing.

### 1. If PostgreSQL is Detected -> Search & Compare DB Providers:
You MUST evaluate and search current pricing/limits for:
- **Supabase:** Check Base Pro cost ($25/mo), included compute credits ($10), Micro/Small/Medium RAM sizes, IOPS limits, IPv4 add-on cost, and egress quota. *Best when Auth/Storage/Realtime are also used or traffic is steady 24/7.*
- **Neon (Serverless Postgres):** Check Compute Unit (CU) hourly pricing, scale-to-zero behavior, storage cost per GB, and branch limits. *Warning: Calculate if 24/7 cron/worker pings prevent scale-to-zero and inflate the monthly bill.*
- **Co-located PaaS Postgres (Railway / Render / Fly.io):** Check cost of running Postgres inside the same private network as the backend API (zero internal egress fees + sub-2ms private network latency).
- **Managed Cloud / Specialized (Aiven / DigitalOcean Managed DB / Tembo / Crunchy Bridge):** Check if dedicated vCPU/RAM is needed at scale.
- **Self-Hosted Docker Postgres (Hetzner / DigitalOcean VPS + Coolify/PgBackRest):** Search VPS RAM-to-price ratio. *Mandatory to evaluate if `pgvector` needs 4GB–16GB+ RAM on a tight budget.*

### 2. If Backend API / Workers are Detected -> Search & Compare Compute Providers:
- **Usage-Based PaaS:** Railway, Fly.io, Google Cloud Run (check CPU/RAM per second + bandwidth egress cost).
- **Fixed-Instance PaaS:** Render, DigitalOcean App Platform (check RAM tiers and cold-start rules).
- **Lean VPS:** Hetzner Cloud, DigitalOcean, Vultr, OVH (check Singapore/APAC regional availability and included TB of bandwidth).

### 3. If Frontend (Next.js / Web) is Detected -> Search & Compare Edge/Web Hosts:
- **Vercel vs. Cloudflare Pages/Workers vs. Self-Hosted Node/Docker** (check per-seat team pricing traps and Fast Origin Transfer / Edge bandwidth costs).

### 4. If Self-Hosted AI / GPU is Detected -> Search & Compare Inference Providers:
- **Serverless GPU:** Modal, RunPod Serverless, Replicate, Baseten.
- **Token API Providers:** Groq, Together AI, Fireworks AI, DeepInfra.

---

## Phase 4: Search Query Templates to Execute
Run `WebSearch` using queries tailored to the detected stack:
- `"Supabase vs Neon pricing [Current Year] compute RAM limits"`
- `"Railway vs Render PostgreSQL pricing RAM bandwidth [Current Year]"`
- `"[Provider Name] bandwidth egress cost per GB [Current Year]"`
- `"[Provider Name] server regions Singapore APAC latency"`

---

## Phase 5: Required Output Format

### 1. Workload & Sizing Breakdown
- **Detected Architecture:** (List each component found in the repo)
- **Calculated Specs:**
  - App Compute: `[X] vCPU / [Y] GB RAM`
  - Database & Vector Storage: `[X] GB Disk / [Y] GB RAM required for indexes`
  - Monthly Bandwidth Egress: `[X] GB/month`
  - Target Region: `[Region]`

### 2. Layer-by-Layer Provider Comparison (Live Search Verified)

#### A. Database Layer (e.g., PostgreSQL + pgvector)
| Provider & Tier | Billing Model | RAM / vCPU / Storage | Connection Pooling & Vector Support | Egress / Network Notes | Est. Monthly Cost | Verdict |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Supabase** | Fixed Instance | ... | Supavisor + pgvector | ... | $... | Best if... |
| **Neon** | Compute Hours | ... | PgBouncer + pgvector | ... | $... | Best if... |
| **Railway / Render DB** | Usage / Fixed | ... | Private Network | Free internal traffic | $... | Best if... |
| **VPS (Self-Hosted)** | Fixed VPS | ... | Full root control | 1–20 TB included | $... | Best if... |

#### B. Backend & Worker Layer
| Provider & Tier | Specs (vCPU / RAM) | Included Bandwidth & Overage | Private DB Networking? |Closest Region | Est. Monthly Cost | Verdict |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Provider 1** | ... | ... | ... | ... | $... | ... |
| **Provider 2** | ... | ... | ... | ... | $... | ... |
| **Provider 3** | ... | ... | ... | ... | $... | ... |

*(Add Tables C & D for Frontend or GPU layers if present in the repo)*

### 3. Cloud Bill Traps & Overkill Rejected
- List hidden fees uncovered during search (e.g., Neon 24/7 awake cost vs. Supabase Micro RAM cap vs. cross-cloud DB latency/egress).
- Explain why higher tiers were rejected.

### 4. Final Recommended Stack ("The Sweet Spot")
- **Selected Providers per Layer:** (e.g., Frontend on X + Backend on Y + Postgres on Z)
- **Total Combined Monthly Cost:** `$XX.XX / month`
- **Why this combination wins for your exact bandwidth & latency.**

### 5. Ponytail Debt & Scale Ledger (Ceilings & Upgrade Triggers)
For every lean tier or architectural simplification chosen, output a strict ceiling and upgrade trigger:

| Layer / Shortcut Chosen | What Was Skipped | Hard Ceiling (When It Breaks) | Upgrade Trigger (When to Move) |
| :--- | :--- | :--- | :--- |
| e.g., Single Hetzner CPX21 (4GB RAM) for API + Postgres | Managed HA failover & separate DB host | ~350k `1536d` vectors in HNSW (~3.1GB RAM) or single-node disk failure | RAM sustained above 80%, `pgvector` index exceeds 2.5GB, or revenue requires 99.95% SLA |
| e.g., Neon Free/Launch Scale-to-Zero | Dedicated 24/7 compute instance | Cold-start latency (~300–500ms) or 24/7 background worker keeping CU active | Adding cron jobs running more than once every 5 mins (switches bill from ~$5 to ~$40+/mo) |

## Safety Boundary (Lazy, Never Negligent)
Never recommend a cheaper/self-hosted tier by cutting data durability or trust boundaries:
- **Self-Hosted DB Rule:** Only recommend VPS Docker Postgres if the cost/setup calculation includes automated daily offsite backups (e.g., `pgBackRest` or `wal-g` to Cloudflare R2 / S3) and firewall-locked private networking.
- **Connection Safety:** Serverless compute + Postgres MUST include a connection pooler (`PgBouncer` / `Supavisor`) in the spec.
