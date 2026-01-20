# ISO 20022 Payments Middleware

> **📊 Implementation Status**: **~85% Complete** | See [docs/FEATURE_STATUS.md](docs/FEATURE_STATUS.md) for detailed tracking
> 
> **Current Version**: v2.0 (Multi-Project Architecture)
> 
> **Status Legend**: ✅ Implemented | ⚠️ Partial | 🔜 Planned

Production-ready middleware that ingests on-chain blockchain transactions, generates compliant ISO 20022 XML messages, creates cryptographic evidence bundles, and anchors them on EVM-compatible chains (Flare, etc.) for immutable audit trails.

## Key Features

### Core Capabilities ✅
- **15 ISO 20022 Message Types**: pain.001, pain.002, pain.007, pain.008, pacs.002, pacs.004, pacs.007, pacs.008, pacs.009, camt.029, camt.052, camt.053, camt.054, camt.056, remt.001
- **Multi-Chain Anchoring**: Support for multiple EVM chains with contract verification
- **Tenant-Mode Anchoring**: Users can anchor with their own wallets via MetaMask
- **Project Isolation**: Multi-project support with SIWE (Sign-In With Ethereum) authentication
- **Evidence Bundles**: Deterministic ZIP files with cryptographic signatures
- **Real-Time Updates**: Server-Sent Events (SSE) for live receipt tracking
- **TypeScript & Python SDKs**: Full-featured client libraries

### Advanced Features ⚠️
- **FX Support**: Infrastructure for fiat equivalents (EqvtAmt + XchgRateInf) - providers pending 🔜
- **IPFS Storage**: Full upload/download support with web3.storage ✅
- **Arweave Integration**: Complete implementation via Bundlr ✅
- **W3C Verifiable Credentials**: Planned 🔜
- **Travel Rule (IVMS 101)**: Configuration exists, enforcement planned 🔜

### x402 Payment Protocol & Autonomous Agents ✅ NEW
- **Micropayment API**: Pay-per-use endpoints with USDC on Base chain
- **6 Premium Endpoints**: Verify bundles, generate statements, FX lookup, bulk operations
- **XMTP Agent**: Autonomous AI agent with natural language command processing
- **Automatic Payments**: Agents handle USDC transfers transparently via x402 protocol
- **Agent Management**: Full CRUD API + UI for managing autonomous agents
- **Revenue Analytics**: Track payments, usage, and revenue by endpoint
- **Multi-Agent Support**: Run multiple agents per project with independent wallets

## Tech Stack

- **Backend**: Python 3.11, FastAPI, SQLAlchemy, PostgreSQL
- **Blockchain**: web3.py, Flare/EVM chains
- **ISO 20022**: lxml with XSD validation
- **Frontend**: Next.js 14, TypeScript, TailwindCSS
- **Auth**: API Keys, SIWE (Sign-In With Ethereum)
- **Storage**: Local files, IPFS (partial), Arweave (planned)

## Quick Links

- 📖 [Feature Status](docs/FEATURE_STATUS.md) - Comprehensive implementation tracking
- 📘 [API Documentation](API_Documentation.md) - Complete API reference with examples
- 🗺️ [Development Plan](DEVELOPMENT_PLAN.md) - Roadmap and phase tracking
- 🧑‍💻 [Developer Guide](DEVELOPER_GUIDE.md) - Setup and development
- 👤 [User Guide](USER_GUIDE.md) - End-user documentation
- 🎨 [UI Features](docs/UI_FEATURES.md) - web-alt UI capabilities
- 🛠️ [TypeScript SDK](packages/sdk/README.md) - Client library for TS/JS
- 🐍 [Python SDK](packages/sdk-python/README.md) - Client library for Python
- 🤖 [x402 Integration Guide](docs/X402_INTEGRATION.md) - Micropayment protocol setup
- 🤖 [Agents Guide](docs/AGENTS_GUIDE.md) - XMTP agent deployment
- 💾 [Storage Backends](docs/STORAGE.md) - IPFS and Arweave integration

## Architecture Overview

- **API (FastAPI)**: REST endpoints under `/v1/`, background workers, SSE streams, file serving
- **Evidence Bundles**: Deterministic ZIP with manifest, signatures, and ISO XML files
- **Blockchain Anchoring**: EvidenceAnchor smart contracts on EVM chains with event verification
- **UI (web-alt)**: Production Next.js dashboard with project management, verification, SDK generation
- **Smart Contracts**: Solidity contracts with Factory pattern for deployments
- **SDKs**: TypeScript and Python clients with contract ABIs

### Project Structure

```
├── app/                    # FastAPI backend
│   ├── api/routes/        # API endpoints
│   ├── iso_messages/      # ISO 20022 generators (15 types)
│   ├── auth/              # Authentication (API keys, SIWE)
│   └── services/          # Business logic
├── web-alt/               # Next.js UI
│   ├── app/               # Pages and API routes
│   ├── components/        # React components
│   └── lib/               # Client utilities
├── packages/              # SDKs
│   ├── sdk/               # TypeScript SDK
│   └── sdk-python/        # Python SDK
├── contracts/             # Solidity contracts + ABIs
├── docs/                  # Documentation
└── tests/                 # Test suite
```

**TIP**: For production, deploy the API and web-alt UI as separate services.

## 1) TL;DR — Run the UI (web-alt)

1. Start API: `uvicorn app.main:app --reload --port 8000`
2. Start UI:
   ```
   cd web-alt
   npm install
   npm run dev
   ```
3. Open http://localhost:3000 (defaults to API http://127.0.0.1:8000)


## 2) Architecture at a glance

- API (FastAPI): routes under `/v1/...`, background worker for bundling + anchoring, SSE for live updates, files under artifacts dir
- Evidence bundle: deterministic zip with manifest; optional signature; downloadable via API
- Anchoring: bundle hash anchored via EvidenceAnchor; verification endpoint checks on-chain log
- UI:
  - web-alt (Next.js in `web-alt/`): Production-ready single page UI with dashboard, verify, SDK builder, statements, config, AI assistant
- (Removed) Capella integration
- Contracts: `contracts/` (Solidity, ABI, deployed.json)

Repository map
- `app/` (FastAPI, ISO generator, anchoring, SSE, DB models, schemas)
- `web-alt/` (Next.js UI)
- `contracts/` (EvidenceAnchor)
- `scripts/` (deploy, anchor, find, smoke tests)
- `ui/`, `embed/` (live receipt page + embeddable widget)
- `alembic/` (migrations)


## 3) Local development

Prereqs
- Python 3.11
- Node.js >= 18.17 (if using web-alt or scripts)
- Docker (optional)

Backend (API)
```bash
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
# Docs:   http://127.0.0.1:8000/docs
# Health: http://127.0.0.1:8000/v1/health
```

Create a test receipt
```bash
curl -X POST http://127.0.0.1:8000/v1/iso/record-tip \
  -H "Content-Type: application/json" \
  -d '{
    "tip_tx_hash":"0xabc",
    "chain":"flare",
    "amount":"0.000000000000000001",
    "currency":"FLR",
    "sender_wallet":"0xS",
    "receiver_wallet":"0xR",
    "reference":"demo:tip:1"
  }'
```
Response: `{ "receipt_id": "<uuid>", "status": "pending" }`

Zero‑polling live view and files
- Live page: `http://127.0.0.1:8000/receipt/<receipt_id>` → redirects to `/ui/receipt.html?rid=...`
- Embeddable widget: `http://127.0.0.1:8000/embed/receipt?rid=<receipt_id>&theme=light`
- SSE stream: `GET /v1/iso/events/<receipt_id>`
- Receipt detail: `GET /v1/iso/receipts/<receipt_id>` returns links to XML and evidence bundle
- Verify bundle: `POST /v1/iso/verify` with `{ "bundle_url": "http://127.0.0.1:8000/files/<rid>/evidence.zip" }`

web-alt (Next.js)
```bash
cd web-alt
npm install
npm run dev   # http://localhost:3000
# Optional for local if API not on default:
# PowerShell: $env:NEXT_PUBLIC_API_BASE="http://127.0.0.1:8000"
```


## 4) Environment variables

Backend (API — FastAPI)
- PUBLIC_BASE_URL = https://<api-public-url>
- ARTIFACTS_DIR = /data/artifacts                 (mount a volume in prod)
- DATABASE_URL = postgresql+psycopg://...         (Railway Postgres connection)
- SQL_ECHO = 0
- API_KEYS = admin-...,partner-...                (comma separated; enables API key auth)
- FLARE_RPC_URL = https://.../rpc
- ANCHOR_CONTRACT_ADDR = <address>
- ANCHOR_ABI_PATH = contracts/EvidenceAnchor.abi.json
- ANCHOR_LOOKBACK_BLOCKS = 50000
- ANCHOR_PRIVATE_KEY = 0x...                      (secret; backend only)
- AI_PROVIDER = openai
- OPENAI_API_KEY = sk-...                         (secret; backend only)
- AI_MODEL = gpt-4o-mini
- AI_TEMPERATURE = 0.2
- AI_MAX_TOKENS = 512

UI (web-alt)
- NEXT_PUBLIC_API_BASE = https://<api-public-url>
- NEXT_PUBLIC_API_KEY = <key> (optional for local testing; avoid in production)

Capella (example)
```
ISO_MIDDLEWARE_URL=https://<api-public-url>
ISO_MW_TIMEOUT_MS=30000
```

IMPORTANT
- Browsers/Next.js must call the API via its PUBLIC URL, not the private `...railway.internal` host.
- Keep secrets on the API only; never put secrets in `NEXT_PUBLIC_*`.


## 5) Production deployment on Railway (Recommended)

Create a NEW project. Do not reuse older projects.

Services
- api-service (repo root)
  - Deploy: Dockerfile
  - Port: 8000
  - Volume: create (e.g., `artifacts-iso`, mount at `/data`), set `ARTIFACTS_DIR=/data/artifacts`
  - Postgres: provision and set `DATABASE_URL`
  - Env: set variables listed in Backend section (including FLARE / contract / OpenAI / API_KEYS)

- ui-service (web-alt)
  - Source: Root Directory = `web-alt`
  - Deploy: Dockerfile Path = `web-alt/Dockerfile`
  - Port: 3000
  - Env: `NEXT_PUBLIC_API_BASE=https://<api-public-url>`

Domains & CORS
- Attach public domains (optional): api.example.com → api-service; ui.example.com → ui-service
- Ensure API `PUBLIC_BASE_URL` matches the public API domain
- Ensure API CORS allows your UI origin (e.g., `ALLOW_ORIGINS=https://<ui-public-url>` if you enforce origins)

Public vs Private URLs
- UI must call API using the public URL (https://<api>.up.railway.app or custom domain)
- Internal hostname `...railway.internal` is for service-to-service traffic only (not reachable from browsers)

Smoke tests (post-deploy)
- Health: `GET https://<api-public-url>/v1/health`
- Create receipt (with API key if configured):
  ```bash
  curl -X POST https://<api-public-url>/v1/iso/record-tip \
       -H "Content-Type: application/json" \
       -H "X-API-Key: <your-key>" \
       -d '{"tip_tx_hash":"0xabc","chain":"flare","amount":"0.000000000000000001","currency":"FLR","sender_wallet":"0xS","receiver_wallet":"0xR","reference":"prod:tip:1"}'
  ```
- List receipts: `GET https://<api-public-url>/v1/receipts`
- UI: open `https://<ui-public-url>` and verify dashboard loads receipts
- Verify: `POST /v1/iso/verify` with bundle_url from the receipt detail


## 6) Capella integration (quick)

Use `capella_integration/`:
- Copy `lib/isoClient.ts`
- Add routes under `app/api/iso/...` in Capella:
  - `record-tip/route.ts` → proxies `POST /v1/iso/record-tip`
  - `receipts/[id]/route.ts` → proxies `GET /v1/iso/receipts/{id}`
  - `verify/route.ts` → proxies `POST /v1/iso/verify`
  - `callback/route.ts` (optional) → receive callback updates

Capella env
```
ISO_MIDDLEWARE_URL=https://<api-public-url>
ISO_MW_TIMEOUT_MS=30000
```

Zero‑polling options
- Link to `/receipt/{receipt_id}` or embed `/embed/receipt?rid={receipt_id}`
- Or pass `callback_url` in `record-tip`; middleware will POST results to your backend


## 7) Flare anchoring details

- Contract address and chain are tracked in `contracts/EvidenceAnchor.deployed.json` (source of truth).
- ABI: `contracts/EvidenceAnchor.abi.json`
- RPC URL: set `FLARE_RPC_URL` in the API env
- Anchored fields: bundle/content hashes (and optionally policy digest if enabled later)


## 8) Troubleshooting

- Next.js (web-alt) builds on Railway
  - Ensure service uses `Root Directory=web-alt` and `Dockerfile Path=web-alt/Dockerfile`
  - Set `NEXT_PUBLIC_API_BASE` on the UI service
  - If you see Python requirements in logs, you’re building the root Dockerfile — fix the path
- CORS & public URLs
  - UI must call the API public URL; private `...railway.internal` is not accessible from browsers
  - Allow your UI origin in the API (CORS) if enforcing origins
- SSE behind proxies
  - Confirm your host does not buffer streaming responses
- Verify failures
  - Ensure `bundle_url` is reachable; confirm on-chain lookups via RPC


## 9) Security

- Do not commit secrets (`.env`)
- Keep private keys and provider tokens in backend environment only
- Use API keys / rate limits for write endpoints
- Avoid exposing secrets via `NEXT_PUBLIC_*` variables


## 10) x402 Payment Protocol & Autonomous Agents

### Overview

The x402 payment protocol enables autonomous AI agents to make micropayments for API access using USDC on Base chain. Agents can interact via natural language through XMTP messaging and automatically handle payment processing.

### Quick Start - Deploy an Agent

```bash
# Navigate to agent directory
cd agents/iso-x402-agent

# Install dependencies
npm install

# Configure environment
cp .env.example .env
# Edit .env with your wallet private key and configuration

# Build and run
npm run build
npm start
```

### Available Premium Endpoints

| Endpoint | Price (USDC) | Description |
|----------|--------------|-------------|
| `/v1/x402/premium/verify-bundle` | 0.001 | Verify evidence bundle |
| `/v1/x402/premium/generate-statement` | 0.005 | Generate camt.052/053 statement |
| `/v1/x402/premium/iso-message/{type}` | 0.002 | Get specific ISO message |
| `/v1/x402/premium/fx-lookup` | 0.001 | FX rate lookup |
| `/v1/x402/premium/bulk-verify` | 0.010 | Bulk bundle verification |
| `/v1/x402/premium/refund` | 0.003 | Initiate refund via agent |

### Agent Commands (via XMTP)

**Free Commands:**
```
list [limit]              # List recent receipts
get <receipt_id>          # Get receipt details
help                      # Show available commands
```

**Paid Commands (Auto-payment):**
```
verify <bundle_url>       # Verify bundle (0.001 USDC)
statement <date>          # Generate statement (0.005 USDC)
refund <receipt_id>       # Initiate refund (0.003 USDC)
```

### Example Agent Interaction

```
User: list 5

Agent: 📋 Recent Receipts (5):
       🧾 PAY-20260120-001
          ID: abc-123-def
          Amount: 100.00 USD
          Status: anchored
          ...

User: verify https://ipfs.io/ipfs/Qm...

Agent: ⏳ Verifying bundle (paying 0.001 USDC)...
       ✅ Verification Complete
       Valid: ✓ Yes
       Bundle Hash: 0x1234...
       💰 Payment: 0.001 USDC paid
```

### Agent Management UI

Access at `http://localhost:3000/agents`:
- Register new agents with wallet addresses
- Configure endpoint pricing
- View revenue analytics by endpoint
- Track agent usage and spending
- Monitor payment history

### Documentation

- **[x402 Integration Guide](docs/X402_INTEGRATION.md)** - Complete protocol documentation
- **[Agents Guide](docs/AGENTS_GUIDE.md)** - Agent setup and deployment
- **[Feature Status](docs/FEATURE_STATUS.md)** - Implementation status

### Key Features

✅ **Payment Protocol**: USDC micropayments on Base chain  
✅ **6 Premium Endpoints**: Pay-per-use API access  
✅ **XMTP Agent**: Natural language interface  
✅ **Automatic Payments**: Transparent USDC handling  
✅ **Agent Management**: Full CRUD API + UI  
✅ **Revenue Analytics**: Payment tracking & reporting  
✅ **Multi-Agent**: Support multiple agents per project  

---

## Appendix — ISO 20022 mapping (pain.001.001.09)

- Message: `Document/CstmrCdtTrfInitn` (namespace: `urn:iso:std:iso:20022:tech:xsd:pain.001.001.09`)
- Groups used: `GrpHdr`, `PmtInf`, `CdtTrfTxInf` (wallet mapping to Othr/Id)
- XSDs: place official schemas under `./schemas` (see `schemas/README.md`). If absent, generation proceeds without runtime XSD validation.
