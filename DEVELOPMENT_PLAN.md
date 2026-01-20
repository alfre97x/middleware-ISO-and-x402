# Development Plan and Implementation Phases

> **📊 Current Status**: See [docs/FEATURE_STATUS.md](docs/FEATURE_STATUS.md) for detailed implementation tracking.
> 
> **Quick Summary (as of Jan 2026)**:
> - ✅ Phase 0: ~95% Complete
> - ⚠️ Phase 1: ~75% Complete (refund endpoint being added)
> - ⚠️ Phase 2: ~80% Complete (auth complete, queue status unclear)
> - ⚠️ Phase 3: ~40% Complete (multi-chain ✅, IPFS/Arweave being added)
> - ✅ Phase 4: ~90% Complete (web-alt UI with project management)
> - ✅ Phase 5: ~95% Complete (documentation cleanup in progress)
> - ⚠️ Phase 6: Status unknown (some tests exist)

This document sets the roadmap, scope, and acceptance criteria for evolving the ISO 20022 Payments Middleware from a PoC into a production-ready, configurable system. It is designed so any developer can resume an interrupted job with clear phase boundaries, checklists, and references.

Contents
- 0. Guiding Principles
- 1. Cross-Cutting Design Decisions (Auth, FX, Storage, Tenancy, Reliability)
- 2. Phase-by-Phase Plan (0–6)
- 3. Interruption & Resume Guide
- 4. Milestones & Deliverables Checklist

-------------------------------------------------------------------------------

0) Guiding Principles

- Deterministic evidence: Artifacts are reproducible and cryptographically anchored. Any extension (FX, compliance, multi-chain) must preserve determinism or clearly document non-deterministic inputs (e.g., time, external rates).
- Configurability first: Behavior is driven by an organization/tenant config (OrgConfig). No hard-coded finance logic where a policy flag exists.
- Message correctness: All ISO outputs validate against vendored XSDs (when provided).
- Observability and safety: Every async job has retries, dead-letter, logs, and metrics.
- Least privilege & incremental security: API keys now; JWT/OIDC and more granular scopes when multi-tenant or external partners expand.

-------------------------------------------------------------------------------

1) Cross-Cutting Design Decisions

1.1 Authentication (now and future)
- Now: API Keys
  - Require X-API-Key header for write endpoints (record-tip, refund, debug/anchor).
  - Store hashed keys; rotate keys per environment.
  - Rate-limit per key (slowapi) where relevant.
- Future: JWT/OIDC
  - Add when onboarding multiple third parties or requiring user-level scopes.
  - Strategy: FastAPI JWT dependency; map claims → tenant_id/scopes; optional OIDC provider.
  - Docs: Explain migration path and when to prefer JWT over pure API keys.

1.2 FX Policy and Fiat-Facing Options
- Baseline (crypto-native):
  - In pain.001, set InstdAmt/@Ccy to the on-chain asset ticker (e.g., FLR). Note: not ISO 4217; acceptable for PoC/sandbox.
- Fiat-facing options:
  - EqvtAmt + XchgRateInf (recommended): Keep crypto InstdAmt; add EqvtAmt (ISO 4217, e.g., EUR/USD) and XchgRateInf (rate + source + timestamp).
  - Fiat InstdAmt: Set InstdAmt to fiat and carry the crypto amount in Remittance/AdditionalData (or a proprietary extension).
- Provider abstraction:
  - PoC: HTTP price API (e.g., CoinGecko) cached 1–5 minutes.
  - Prod-ish: Chainlink price feeds when available; fallback to HTTP with auditing.
  - Persist the exact rate, timestamp, provider in DB and include in bundle manifest.

1.3 Storage Backends: Local, IPFS, Arweave
- Local (default): Serve at /files/{id} paths.
- IPFS (recommended optional backend):
  - Why: Low friction, public deduplication via CIDs, easy pinning services (web3.storage, Pinata).
  - How: Upload evidence.zip, store CID; optionally anchor the zip hash (preferred) and include CID in metadata.
- Arweave (future optional backend):
  - Why: Long-term, immutable storage; ideal for compliance, audit permanence.
  - How: Fund wallet, submit bundle; persist transaction ID; consider anchoring both the zip hash and the Arweave TX hash.
  - Docs: Clarify cost model, permanence, and when to prefer Arweave over IPFS.

1.4 Tenancy (Single → Multi)
- Now: Single-tenant OrgConfig (singleton row).
- Later: Multi-tenant
  - Tenants table; api_keys(tenant_id); receipts(tenant_id); org_config(tenant_id).
  - All queries scoped by tenant; keys/auth map request → tenant context.

1.5 Reliability & Scale-Out
- Queue & Retry: Redis + RQ with exponential backoff and DLQ (dead-letter queue).
- SSE multi-instance: Replace in-memory hub with Redis pub/sub to broadcast events across replicas.
- Logs & Metrics: structlog JSON + Prometheus metrics (FastAPI instrumentator), Grafana dashboard JSON with key graphs (throughput, errors, latency, retry counts).

-------------------------------------------------------------------------------

2) Phase-by-Phase Plan

Phase 0 — Baseline Architecture & Config
- Objectives:
  - Introduce OrgConfig (Pydantic + DB JSON) with all parameters required for future phases.
  - Baseline Alembic migrations; refactor existing pain.001 generator behind an iso_messages module.
- Scope:
  - DB (Alembic baseline):
    - receipts (existing)
    - New tables:
      - iso_artifacts(id, receipt_id FK, type, path, sha256, created_at)
      - chain_anchors(id, receipt_id FK, chain, txid, anchored_at)
      - org_config(id, payload JSON, created_at, updated_at)
  - Code:
    - app/config.py (load/save OrgConfig)
    - app/iso_messages/pain001.py (move existing generation; add hooks for FX policy but disabled)
    - Repoint main flow to record iso_artifacts for pain.001
  - Endpoints:
    - GET /v1/config, PUT /v1/config
    - GET /v1/iso/messages/{rid}?type=pain.001  (metadata/URL)
  - Docs:
    - DEVELOPMENT_PLAN.md (this file)
    - Update API_Documentation.md sections for config endpoints (read-only note if write not yet open)
- Acceptance:
  - Alembic migration up/down works.
  - pain.001 still generated identically; iso_artifacts row created.

Phase 1 — ISO Suite & Fiat-Facing Options
- Objectives:
  - Add pain.002 (status), camt.054 (DCN), pacs.004 (returns), remt.001 (optional).
  - Implement FX policy (EqvtAmt + XchgRateInf; optional Fiat InstdAmt mode).
- Scope:
  - Code:
    - app/iso_messages/pain002.py: status report on state transitions.
    - app/iso_messages/camt054.py: single-receipt DCN after anchoring.
    - app/iso_messages/pacs004.py: on refund endpoint, generate return.
    - app/iso_messages/remt001.py: structured remittance (optional, config-driven).
  - Endpoints:
    - GET /v1/iso/messages/{rid}?type=pain.002|camt.054|pacs.004|remt.001
    - POST /v1/iso/refund {receipt_id, reason?} → creates pacs.004 and anchors bundle.
  - DB:
    - iso_artifacts rows for each ISO message; chain_anchors rows for new anchors if produced.
    - receipts.refund_of (nullable FK) if returns reference a prior payment.
  - Docs:
    - API_Documentation.md: new message types, refund flow, FX parameters and examples.
    - ISO_MAPPING.md: XPath mapping details for each message, FX examples.
- Acceptance:
  - Schema-valid ISO outputs with vendored XSDs.
  - FX policy results reproducible; recorded in bundle manifest.

Phase 2 — Reliability, Auth, Observability
- Objectives:
  - Move background work to Redis + RQ; add retries & DLQ. Replace SSE hub with Redis pub/sub.
  - Implement API key auth for write endpoints; rate limiting.
  - Add Prometheus metrics & structlog JSON logs.
- Scope:
  - Services:
    - docker-compose: add Redis; add rq worker service.
  - Code:
    - jobs: generate_xml, build_bundle, anchor_chain, emit_status, callback_post
    - SSE hub implementation: Redis pub/sub; existing interface preserved.
    - Auth dependency to validate X-API-Key (hashed secrets in DB or env).
    - Prometheus metrics; instrument key code paths; structlog logging with request IDs.
  - Docs:
    - OPERATIONS.md: running workers, queue management, DLQ handling.
    - SECURITY.md: API key management, rotation, deployment notes.
- Acceptance:
  - Background tasks resilient; DLQ visible; SSE works across multiple app instances.
  - /metrics exposes basic graphs; structured logs present.

Phase 3 — Evidence Extensions (Multi-Chain, IPFS, VC, Travel Rule)
- Objectives:
  - Multi-chain anchoring support; IPFS storage backend; optional Arweave docs; W3C VC issuance; IVMS 101.
- Scope:
  - Code:
    - Multi-chain: config lists chains; anchor each; record in chain_anchors; verify aggregates matches.
    - Storage backends: local|ipfs|arweave pluggable. Implement IPFS first; document Arweave usage and include stubs.
    - VC issuance: did:key signing; include credential.json in bundle; expose verification path.
    - IVMS 101: add ivms101.json to bundle when compliance threshold triggers.
  - Endpoints:
    - GET /v1/anchors/{rid} → all chain anchors
    - Verify enhancements: accept CID-based verify if IPFS enabled.
  - Docs:
    - STORAGE.md: IPFS (how/why), Arweave (how/why, tradeoffs).
    - COMPLIANCE.md: IVMS, PII handling, public/private artifacts.
- Acceptance:
  - Bundle/verify works with IPFS; multi-chain anchors recorded; VC generated; IVMS policy enforced.

Phase 4 — UI (Streamlit) Integration
- Objectives:
  - Add tabs: Receipts, Live, Messages, Config Wizard, Monitoring, SDK Generator.
- Scope:
  - UI Features:
    - Receipts: search, detail view, ISO artifacts downloads, verify, refund action, anchor list.
    - Live: SSE view for updates and anchors across chains.
    - Config Wizard: forms for OrgConfig; save to API; preview JSON.
    - Monitoring: charts from Prometheus; recent error logs.
    - SDK Generator: select languages and features; trigger codegen (OpenAPI Generator) and download.
  - Docs:
    - UI_GUIDE.md with screenshots and flows.
- Acceptance:
  - All tabs functional; SDKs downloadable (TS/JS, Python at minimum).

Phase 5 — API Surface & Documentation Completeness
- Objectives:
  - Finalize endpoints, add auth/limits to docs, detail multi-tenant future, Arweave usage, JWT roadmap.
- Scope:
  - Endpoints audit and OpenAPI correctness.
  - Docs:
    - Update API_Documentation.md: 
      - JWT future use: when to adopt, example flow.
      - Arweave: why/how to use (cost/durability), configuration steps.
      - Multi-tenant: how it would change keys, scoping, DB; sample tenant-aware requests.
      - Fiat-facing FX options and examples.
  - README: architecture diagram; “Production Hardening” section summary.
- Acceptance:
  - Docs are current; all new endpoints present with examples; OpenAPI schema accurate.

Phase 6 — Testing & QA
- Objectives:
  - Thorough unit/integration/E2E; perf sanity; CI pipeline.
- Scope:
  - Tests:
    - Unit: ISO builders (snapshot XML), FX calculation, config parsing.
    - Integration: queue flows, Redis pub/sub, file storage, IPFS adapter.
    - E2E: smoke tests for full receipt lifecycle, refund flow, verify across chains.
  - CI:
    - Lint, type-check, test matrix; build Docker images; run migrations in CI.
- Acceptance:
  - Green CI; coverage targets met; reproducible builds; images published (if applicable).

-------------------------------------------------------------------------------

3) Interruption & Resume Guide

3.1 Where am I?
- DB migration status:
  - Run: alembic current  → note head(s)
  - If heads diverge: resolve with alembic merge as per team guidelines.
- Services:
  - docker compose ps → API, worker, Redis should be up for phases 2+.
- Config:
  - GET /v1/config → confirm OrgConfig loaded; validate critical fields (ledger, anchoring, evidence).
- Queues:
  - rq info (or Redis UI) → verify queues not stuck; check DLQ list if present.

3.2 How to resume a phase
- Identify the phase by checking:
  - Code modules present (e.g., iso_messages/* for Phase 1).
  - Endpoints registered (OpenAPI docs).
  - Docs updated per phase checklists (README/API_Documentation.md sections mentioned above).
- Use the acceptance criteria for the current phase to verify completion.
- If migrations changed:
  - Make sure alembic upgrade head runs on your env; don’t modify models without migration.

3.3 Rollback/Recovery
- If a migration fails:
  - alembic downgrade -1 → repair; edit migration; upgrade again.
- If queues fail repeatedly:
  - Pause worker; inspect DLQ; requeue after fix.
- If anchors fail:
  - Use /v1/debug/anchor guarded by API key (dev only); never run in prod without governance.

3.4 Environment Variables (Quick Reference)
- Security/Auth: API_KEYS, JWT_*, OIDC_* (future)
- Ledger/Anchor: FLARE_RPC_URL, ANCHOR_CONTRACT_ADDR, ANCHOR_PRIVATE_KEY, ANCHOR_LOOKBACK_BLOCKS
- Evidence/Keys: SERVICE_PRIVATE_KEY, SERVICE_PUBLIC_KEY, ARTIFACTS_DIR
- Queue/SSE: REDIS_URL
- Metrics: PROMETHEUS_MULTIPROC_DIR (if needed)
- Storage: IPFS_* credentials, ARWEAVE_* (future)
- Misc: PUBLIC_BASE_URL, ALLOW_ORIGINS, REDIS_URL, DATABASE_URL, SQL_ECHO

-------------------------------------------------------------------------------

4) Milestones & Deliverables Checklist

M0 (Phase 0) - ✅ ~95% Complete
- [x] Alembic baseline created; new tables iso_artifacts, chain_anchors, org_config.
- [x] Refactor pain.001 under iso_messages; iso_artifacts rows written.
- [x] GET/PUT config endpoints; basic docs updated.
- [x] Added project-level isolation (projects table, per-project configs)
- [x] SIWE authentication for project registration

M1 (Phase 1) - ⚠️ ~75% Complete
- [x] pain.002, camt.054, pacs.004 (and optional remt.001) builders implemented.
- [x] FX policy (EqvtAmt + XchgRateInf) infrastructure; message format support added.
- [x] GET /v1/iso/messages endpoint for listing ISO artifacts.
- [ ] POST /v1/iso/refund endpoint (🔜 being implemented)
- [ ] FX provider integrations (CoinGecko, Chainlink) - infrastructure exists, providers pending
- [x] Additional message types: pain.007, pain.008, pacs.002, pacs.007, pacs.008, pacs.009, camt.029, camt.056

M2 (Phase 2) - ⚠️ ~80% Complete
- [x] API key auth fully implemented; SIWE auth added
- [x] SSE via Redis for multi-instance support
- [x] Observability module with structlog and Prometheus support
- [ ] Redis + RQ jobs status unclear (jobs.py exists, unclear if Redis/RQ active or in-memory fallback)
- [ ] DLQ implementation unclear
- [ ] Rate limiting partially implemented
- [x] Auth documentation (API_Documentation.md, UI_FEATURES.md)

M3 (Phase 3) - ⚠️ ~40% Complete
- [x] Multi-chain anchoring; GET /v1/anchors/{rid}; verify aggregates per chain.
- [x] Tenant-mode anchoring (awaiting_anchor status, confirm-anchor endpoint)
- [x] Factory-based contract deployment via MetaMask (UI integrated)
- [x] CID verification for IPFS
- [ ] IPFS upload functionality (🔜 being implemented)
- [ ] Arweave integration (🔜 planned, documentation exists)
- [ ] W3C VC issuance (🔜 planned)
- [ ] IVMS 101 / Travel Rule (🔜 planned, config exists)

M4 (Phase 4) - ✅ ~90% Complete (Note: web-alt, not Streamlit)
- [x] Next.js UI (web-alt) with comprehensive features
- [x] Receipts panel with scope selector (mine/all)
- [x] Project registration and management (SIWE)
- [x] API Keys management panel
- [x] Project Config panel with MetaMask contract deployment
- [x] SDK Builder (TypeScript and Python generation)
- [x] Configuration editor (JSON + quick settings)
- [x] Statement generation UI (camt.052, camt.053)
- [x] AI Assistant panel (project-scoped)
- [x] Bundle verification (URL and CID)
- [ ] Refund UI (🔜 being implemented with refund endpoint)
- [ ] Individual ISO message downloads (currently only full bundle)

M5 (Phase 5) - ✅ ~95% Complete
- [x] API_Documentation.md comprehensive with status badges
- [x] docs/FEATURE_STATUS.md created (comprehensive tracking)
- [x] docs/UI_FEATURES.md documenting web-alt features
- [x] DEVELOPER_GUIDE.md available
- [x] USER_GUIDE.md available
- [x] JWT future guidance in API_Documentation.md
- [x] FX options documented (with implementation status)
- [x] Storage backends documented (IPFS/Arweave with status)
- [x] Multi-tenant guidance (project isolation vs full multi-tenancy)
- [ ] ISO_MAPPING.md (🔜 could be added for XPath details)
- [ ] SECURITY.md (🔜 could be expanded beyond current docs)

M6 (Phase 6) - ⚠️ Status Unknown
- [x] Some tests exist (test_auth_principal.py, test_confirm_anchor_validation.py, etc.)
- [ ] Comprehensive unit test coverage
- [ ] Integration tests for queue flows
- [ ] E2E smoke tests for full lifecycle
- [ ] CI pipeline setup
- [ ] Coverage targets defined and met

**Currently Being Implemented (Priority Tasks)**:
1. POST /v1/iso/refund endpoint + refund UI
2. IPFS upload functionality
3. Arweave integration (stubs + docs)
4. Storage backend selection in UI
5. Individual ISO message downloads in UI

-------------------------------------------------------------------------------

Appendix A: OrgConfig (indicative JSON schema)
{
  "org": { "name": "Capella", "jurisdiction": "SEPA", "defaultMessageFamilies": ["pain.001","pain.002","camt.054"] },
  "ledger": { "network": "flare", "rpcUrl": "https://flare-api.flare.network/ext/C/rpc", "asset": { "symbol":"FLR","decimals":18 } },
  "mapping": { "partyScheme":"WALLET", "accountScheme":"WALLET_ACCOUNT", "chargeBearer":"SLEV", "purpose":"GDDS", "categoryPurpose":"TRAD" },
  "anchoring": { "chains": [{"name":"flare","contract":"0x0690d8cFb1897c12B2C0b34660edBDE4E20ff4d8","rpc_url":null,"explorer_base_url":"https://flarescan.com"}], "lookbackBlocks": 50000, "signatureAlg":"ed25519" },
  "evidence": { "include":["pain001.xml","receipt.json","tip.json","manifest.json","public_key.pem"], "signOver":"zip_without_sig", "store":{"mode":"local|ipfs|arweave"} },
  "fxPolicy": { "mode":"eqvt_amt", "baseCcy":"EUR", "provider":"coingecko", "rounding":"bankers" },
  "status": { "emitPain002": true, "enableCancellation": true, "enableReturns": true },
  "integration": { "openapi": true, "webhook": { "retry":"exponential", "timeoutMs":15000 } },
  "security": { "auth":"api_key", "rateLimit":{"perMinute":100} },
  "compliance": { "travelRule":{"threshold":1000,"provider":"..."}, "sanctions":{"provider":"..."} }
}

Appendix B: JWT (future) – When and how
- When: onboarding multiple external integrators; need per-user or per-service scopes; SSO with OIDC.
- How: verify JWT via provider JWKs; map claims to tenant scope; cache introspection; rotate keys; enforce exp/nbf.

Appendix C: Arweave (future) – Why and how
- Why: durability and strong public persistence for audit-heavy environments.
- How: provision wallet; fund; upload zip; store TX id; provide verify-by-txid path; note cost/latency vs IPFS.

Appendix D: Multi-Tenant (future) – Model
- Tables include tenant_id; API keys bound to tenant; all queries filtered; artifacts/storage segregated by tenant; SDKs generated per-tenant config.
