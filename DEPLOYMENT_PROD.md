# Production Deployment (Flare mainnet + OpenAI + Railway)

Scope
- Chain: Flare mainnet (chainId 14)
- Fresh EvidenceAnchor deployed (address below)
- Backend: FastAPI (uvicorn) with Postgres
- UI: Next.js (web-alt)
- AI: OpenAI (server-side, scope enforced)
- Hosting: Railway (create a NEW project; do not touch existing projects)
- Do NOT deploy Streamlit or mock UIs

Contract (already deployed)
- Address: 0x0690d8cFb1897c12B2C0b34660edBDE4E20ff4d8
- Tx: 0x76e2c88ab0cf69aa291744729a4d8caf7d3a82dcb9ce57212baaee2d80b9e0d2
- Network RPC: https://flare-api.flare.network/ext/C/rpc

Backend env (api-service)
Set these Railway variables (no secrets in UI):
- PUBLIC_BASE_URL=https://api.example.com                     # external API URL
- ARTIFACTS_DIR=/data/artifacts                              # mount volume at /data
- DATABASE_URL=postgresql+psycopg://user:pass@host:5432/db  # from Railway Postgres
- SQL_ECHO=0
- RUN_MIGRATIONS=1
- AUTO_CREATE_DB=0
- API_KEYS=prod-admin-...,partner-...                        # comma-separated; enables X-API-Key
- FLARE_RPC_URL=https://flare-api.flare.network/ext/C/rpc
- ANCHOR_CONTRACT_ADDR=0x0690d8cFb1897c12B2C0b34660edBDE4E20ff4d8
- ANCHOR_ABI_PATH=contracts/EvidenceAnchor.abi.json
- ANCHOR_LOOKBACK_BLOCKS=50000
- ANCHOR_PRIVATE_KEY=0x...                                   # secret; set ONLY in Railway env
- AI_PROVIDER=openai
- OPENAI_API_KEY=sk-...                                      # secret; backend only
- AI_MODEL=gpt-4o-mini
- AI_TEMPERATURE=0.2
- AI_MAX_TOKENS=512

UI env (ui-service; web-alt)
- NEXT_PUBLIC_API_BASE=https://api.example.com

Build artifacts
- Backend: Dockerfile (runs `alembic upgrade head` automatically on startup, then starts uvicorn)
- UI: web-alt/Dockerfile (Next.js on port 3000)

Railway deployment (new project)
1) Login and create new project (CLI or Dashboard)
- Do not reuse existing projects; create a new one.

2) Add services
- api-service:
  - Source: repo root
  - Deploy: Dockerfile
  - Expose: 8000
  - Volume: create a Railway Volume (e.g., artifacts-iso, 10GB) and mount at /data
  - Env: set variables above (including Postgres DATABASE_URL, AI/OpenAI, chain/env)
- ui-service (web-alt):
  - Source: web-alt/
  - Deploy: web-alt/Dockerfile
  - Expose: 3000
  - Env: NEXT_PUBLIC_API_BASE=https://api.example.com

3) Postgres
- Provision Railway Postgres (or attach an existing managed DB)
- Copy connection string to DATABASE_URL in api-service env

4) Domains
- Attach custom domains (optional):
  - api.example.com → api-service
  - ui.example.com → ui-service
- Ensure PUBLIC_BASE_URL matches the API domain

5) SSE & files
- SSE: Railway generally supports streaming; if stalls appear, verify ingress does not buffer responses.
- Files: artifacts under ARTIFACTS_DIR. Ensure /data volume persists across redeploys.

Security
- Keep ANCHOR_PRIVATE_KEY and OPENAI_API_KEY only in backend env (Railway secrets).
- Use API_KEYS to guard write endpoints.
- Leave allow_config_changes OFF in UI assistant scope for production.

Smoke tests (post-deploy)
- Health: GET https://api.example.com/v1/health
- Create a receipt:
  curl -X POST https://api.example.com/v1/iso/record-tip \
       -H "Content-Type: application/json" \
       -H "X-API-Key: prod-admin-..." \
       -d "{\"tip_tx_hash\":\"0xabc123...\",\"chain\":\"flare\",\"amount\":\"0.000000000000000001\",\"currency\":\"FLR\",\"sender_wallet\":\"0xS\",\"receiver_wallet\":\"0xR\",\"reference\":\"prod:tip:1\"}"
- List receipts: GET https://api.example.com/v1/receipts (with X-API-Key when enabled)
- UI dashboard: https://ui.example.com
- Live page: https://api.example.com/receipt/{receipt_id}
- Verify: POST https://api.example.com/v1/iso/verify with bundle_url from GET /v1/iso/receipts/{id}
- AI Assistant: open ui-service, toggle “Allow reading receipts” and ask “List receipts”, confirm scope adherence

Rollback considerations
- Revert to prior known config (keep previous deployed contract details if needed).
- If artifacts storage changes, ensure migration or clear documentation for data retention.

Follow-ups
- Add Alembic migrations for DB schema versioning.
- Add rate limiting and JWT/OIDC as per DEVELOPMENT_PLAN Phase 2.
- Evaluate storage backends: IPFS/Arweave (already supported best-effort).
