# ISO 20022 Payments Middleware API Documentation

> **📊 Implementation Status**: See [docs/FEATURE_STATUS.md](docs/FEATURE_STATUS.md) for comprehensive feature tracking.
> 
> **Legend**: ✅ Implemented | ⚠️ Partial | 🔜 Planned

## Overview

The ISO 20022 Payments Middleware is a production-ready system that bridges blockchain transactions with traditional banking standards. It automatically converts blockchain tips into ISO 20022 XML documents, creates cryptographic evidence bundles, and anchors them on the Flare blockchain for immutable audit trails.

## Quick Start

### 1. Start the Server

```bash
uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
```

### 2. Access Interactive Documentation

- Swagger UI: http://127.0.0.1:8000/docs
- ReDoc: http://127.0.0.1:8000/redoc

### 3. Test the API

```bash
# Health check
curl http://127.0.0.1:8000/v1/health

# Record a tip
curl -X POST http://127.0.0.1:8000/v1/iso/record-tip \
  -H "Content-Type: application/json" \
  -d '{
    "tip_tx_hash": "0xabc123",
    "chain": "flare",
    "amount": "0.001",
    "currency": "FLR",
    "sender_wallet": "0xSender",
    "receiver_wallet": "0xReceiver",
    "reference": "demo:tip:1"
  }'
```

## Core Endpoints

### POST /v1/iso/record-tip ✅

Records a blockchain tip and automatically processes it into ISO 20022 format.

Request Body:
```json
{
  "tip_tx_hash": "0xabc123",
  "chain": "flare",
  "amount": "0.001",
  "currency": "FLR",
  "sender_wallet": "0xSender",
  "receiver_wallet": "0xReceiver",
  "reference": "demo:tip:1",
  "callback_url": "https://your-app.com/callback" // optional
}
```

Response:
```json
{
  "receipt_id": "1150292a-4699-46b6-8a0e-60ece78ce8e2",
  "status": "pending"
}
```

### GET /v1/iso/receipts/{id} ✅

Retrieves detailed information about a processed tip receipt.

Response:
```json
{
  "id": "1150292a-4699-46b6-8a0e-60ece78ce8e2",
  "status": "anchored",
  "bundle_hash": "0xcc4cdd738ada83b7d7c04fd8d96415dfd78dfe1f0011b3250fcb508f77632f4f",
  "flare_txid": "0x58f6e1b8b8175adb7d1ae164a289d3ff2b6370ea5977cbd65cad05a885a5857b",
  "xml_url": "/files/1150292a-4699-46b6-8a0e-60ece78ce8e2/pain001.xml",
  "bundle_url": "/files/1150292a-4699-46b6-8a0e-60ece78ce8e2/evidence.zip",
  "created_at": "2025-10-05T17:34:24.316407",
  "anchored_at": "2025-10-05T17:34:25.351755"
}
```

### POST /v1/iso/verify ✅

Verifies the integrity of an evidence bundle and/or checks on-chain anchoring.

Request Body (Option 1 - Bundle URL):
```json
{
  "bundle_url": "http://127.0.0.1:8000/files/1150292a-4699-46b6-8a0e-60ece78ce8e2/evidence.zip"
}
```

Request Body (Option 2 - Bundle Hash):
```json
{
  "bundle_hash": "0xcc4cdd738ada83b7d7c04fd8d96415dfd78dfe1f0011b3250fcb508f77632f4f"
}
```

Response:
```json
{
  "matches_onchain": true,
  "bundle_hash": "0xcc4cdd738ada83b7d7c04fd8d96415dfd78dfe1f0011b3250fcb508f77632f4f",
  "flare_txid": "0x58f6e1b8b8175adb7d1ae164a289d3ff2b6370ea5977cbd65cad05a885a5857b",
  "anchored_at": "2025-10-05T17:34:25.351755",
  "errors": []
}
```

## Additional Endpoints

### GET /v1/health ✅

Health check endpoint.

Response:
```json
{
  "status": "ok",
  "ts": "2025-10-05T18:30:00.000000"
}
```

### POST /v1/iso/confirm-anchor ✅

If you run in tenant anchoring mode (project config has `anchoring.execution_mode = tenant`), the worker will stop after generating evidence and set status `awaiting_anchor`. A tenant can then confirm anchoring by posting the txid.

Request:
```json
{ "receipt_id": "<uuid>", "flare_txid": "0x...", "chain": "flare" }
```

Response:
```json
{ "receipt_id": "<uuid>", "status": "anchored", "flare_txid": "0x...", "anchored_at": "..." }
```

### GET /v1/iso/events/{id} ✅

Server-Sent Events stream for real-time receipt updates.

Usage:
```javascript
const eventSource = new EventSource('/v1/iso/events/1150292a-4699-46b6-8a0e-60ece78ce8e2');
eventSource.onmessage = function(event) {
  const data = JSON.parse(event.data);
  console.log('Receipt update:', data);
};
```

### POST /v1/debug/anchor ✅

Debug endpoint to directly anchor a bundle hash.

Request Body:
```json
{
  "bundle_hash": "0xcc4cdd738ada83b7d7c04fd8d96415dfd78dfe1f0011b3250fcb508f77632f4f"
}
```

Response:
```json
{
  "flare_txid": "0x58f6e1b8b8175adb7d1ae164a289d3ff2b6370ea5977cbd65cad05a885a5857b",
  "block_number": 123456
}
```

## Configuration

### GET /v1/config ✅
Returns the current organization configuration that controls ISO generation, mapping, anchoring, evidence, FX, etc.

Response (example excerpt):
```json
{
  "org": {"name":"Capella","jurisdiction":"SEPA","default_message_families":["pain.001","pain.002","camt.054"]},
  "ledger": {"network":"flare","rpc_url":"https://flare-api.flare.network/ext/C/rpc","asset":{"symbol":"FLR","decimals":18}},
  "mapping": {"party_scheme":"WALLET","account_scheme":"WALLET_ACCOUNT","charge_bearer":"SLEV"},
  "anchoring": {"chains":[],"lookback_blocks":50000,"signature_alg":"ed25519"},
  "evidence": {"include":["pain001.xml","receipt.json","tip.json","manifest.json","public_key.pem"],"sign_over":"zip_without_sig","store":{"mode":"local"}},
  "fx_policy": {"mode":"none","rounding":"bankers"},
  "status": {"emit_pain002":true,"enable_cancellation":true,"enable_returns":true}
}
```

### PUT /v1/config ✅
Updates configuration. All fields are validated and persisted.
```json
{
  "org": {"name":"Capella","jurisdiction":"SEPA","default_message_families":["pain.001","pain.002","camt.054"]},
  "ledger": {"network":"flare","rpc_url":"https://flare-api.flare.network/ext/C/rpc","asset":{"symbol":"FLR","decimals":18}},
  "mapping": {"party_scheme":"WALLET","account_scheme":"WALLET_ACCOUNT","charge_bearer":"SLEV","purpose":"GDDS"},
  "anchoring": {"chains":[{"name":"flare","contract":"0x0690d8cFb1897c12B2C0b34660edBDE4E20ff4d8","rpc_url":null,"explorer_base_url":"https://flarescan.com"}],"lookback_blocks":50000,"signature_alg":"ed25519"},
  "evidence": {"include":["pain001.xml","receipt.json","tip.json","manifest.json","public_key.pem"],"sign_over":"zip_without_sig","store":{"mode":"local"}},
  "fx_policy": {"mode":"eqvt_amt","base_ccy":"EUR","provider":"coingecko","rounding":"bankers"},
  "status": {"emit_pain002":true,"enable_cancellation":true,"enable_returns":true}
}
```

## ISO Artifacts

In addition to pain.001:
- pain.002 (status report) is generated on state transitions (pending → anchored/failed)
- camt.054 (debit/credit notification) is generated after anchoring
- pacs.004 (payment return) is generated when a refund is initiated

### GET /v1/iso/messages/{id} ✅
List ISO artifacts for a receipt, optionally filtered by type.
Query params:
- type: pain.001 | pain.002 | camt.054 | pacs.004 | remt.001
Response:
```json
[
  {"type":"pain.001","url":"/files/<id>/pain001.xml","sha256":"0x...","created_at":"2025-10-05T17:34:24Z"},
  {"type":"pain.002","url":"/files/<id>/pain002.xml","sha256":"0x...","created_at":"2025-10-05T17:34:26Z"},
  {"type":"camt.054","url":"/files/<id>/camt054.xml","sha256":"0x...","created_at":"2025-10-05T17:34:26Z"}
]
```

## Refunds

### POST /v1/iso/refund ✅

Initiates a payment return for an existing receipt, producing a pacs.004 artifact and scheduling processing (XML/bundle/anchor) for the refund receipt.

**Requirements:**
- Original receipt must exist and be in 'anchored' status
- Requires authentication (API key or SIWE)
- Principal must have access to the original receipt's project

**Request:**
```json
{
  "original_receipt_id": "<uuid>",
  "reason_code": "CUST"  // optional: CUST, DUPL, TECH, FRAD
}
```

**Response:**
```json
{
  "refund_receipt_id": "<uuid>",
  "status": "pending"
}
```

**Example:**
```bash
curl -X POST http://127.0.0.1:8000/v1/iso/refund \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "original_receipt_id": "1150292a-4699-46b6-8a0e-60ece78ce8e2",
    "reason_code": "CUST"
  }'
```

**Process:**
1. Creates new receipt with negative amount (sender/receiver reversed)
2. Generates pacs.004 payment return XML
3. Creates evidence bundle and anchors it
4. Emits status notifications via SSE

**UI Integration:** The web-alt dashboard includes a "Refund" button for all anchored receipts.

## UI Endpoints

### GET /receipt/{id}
Live receipt page with real-time updates.

### GET /embed/receipt?rid={id}
Embeddable widget for integration into other applications.

## Data Models

### TipRecordRequest
```json
{
  "tip_tx_hash": "string",
  "chain": "flare",
  "amount": "number",
  "currency": "string",
  "sender_wallet": "string",
  "receiver_wallet": "string",
  "reference": "string",
  "callback_url": "string" // optional
}
```

### ReceiptResponse
```json
{
  "id": "string (UUID)",
  "status": "pending" | "anchored" | "failed",
  "bundle_hash": "string (0x-prefixed hex)",
  "flare_txid": "string (0x-prefixed hex)",
  "xml_url": "string (relative path)",
  "bundle_url": "string (relative path)",
  "created_at": "string (ISO datetime)",
  "anchored_at": "string (ISO datetime)" // nullable
}
```

### VerifyRequest
```json
{
  "bundle_url": "string (URL)" // OR
  "bundle_hash": "string (0x-prefixed hex)"
}
```

### VerifyResponse
```json
{
  "matches_onchain": "boolean",
  "bundle_hash": "string (0x-prefixed hex)",
  "flare_txid": "string (0x-prefixed hex)", // nullable
  "anchored_at": "string (ISO datetime)", // nullable
  "errors": "array of strings"
}
```

## Error Handling

### HTTP Status Codes
- 200: Success
- 400: Bad Request (invalid input)
- 404: Not Found (receipt not found)
- 422: Validation Error (invalid data format)
- 500: Internal Server Error

### Error Response Format
```json
{
  "detail": "Error message",
  "errors": ["specific", "validation", "errors"]
}
```

## Authentication

API key auth is supported.
- If `API_KEYS` is set (comma-separated) OR API keys exist in DB, endpoints require `X-API-Key`.
- You can manage API keys via:
  - POST /v1/auth/api-keys
  - GET /v1/auth/api-keys
  - DELETE /v1/auth/api-keys/{id}

Additionally, some endpoints support SIWE (Sign-In With Ethereum):
- GET /v1/auth/nonce - Generate a nonce for SIWE authentication
- POST /v1/auth/siwe-verify - Verify SIWE signature and create session
- GET /v1/auth/me - Get current principal information (role, project_id, is_admin)
- POST /v1/auth/siwe-mint-key - Generate new API key using SIWE signature (for key rotation)

### GET /v1/auth/me
Returns information about the currently authenticated principal.

Response:
```json
{
  "role": "project_admin",
  "project_id": "uuid-here",
  "is_admin": false
}
```

### POST /v1/auth/siwe-mint-key
```json
{
  "receipt_id": "1150292a-4699-46b6-8a0e-60ece78ce8e2",
  "status": "anchored",
  "bundle_hash": "0xcc4cdd738ada83b7d7c04fd8d96415dfd78dfe1f0011b3250fcb508f77632f4f",
  "flare_txid": "0x58f6e1b8b8175adb7d1ae164a289d3ff2b6370ea5977cbd65cad05a885a5857b",
  "xml_url": "http://127.0.0.1:8000/files/1150292a-4699-46b6-8a0e-60ece78ce8e2/pain001.xml",
  "bundle_url": "http://127.0.0.1:8000/files/1150292a-4699-46b6-8a0e-60ece78ce8e2/evidence.zip",
  "created_at": "2025-10-05T17:34:24.316407",
  "anchored_at": "2025-10-05T17:34:25.351755"
}
```

## Integration Examples

### JavaScript/Node.js
```javascript
const response = await fetch('http://127.0.0.1:8000/v1/iso/record-tip', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    tip_tx_hash: '0xabc123',
    chain: 'flare',
    amount: '0.001',
    currency: 'FLR',
    sender_wallet: '0xSender',
    receiver_wallet: '0xReceiver',
    reference: 'demo:tip:1'
  })
});

const result = await response.json();
console.log('Receipt ID:', result.receipt_id);
```

### Python
```python
import requests

response = requests.post('http://127.0.0.1:8000/v1/iso/record-tip', json={
  'tip_tx_hash': '0xabc123',
  'chain': 'flare',
  'amount': '0.001',
  'currency': 'FLR',
  'sender_wallet': '0xSender',
  'receiver_wallet': '0xReceiver',
  'reference': 'demo:tip:1'
})

result = response.json()
print(f"Receipt ID: {result['receipt_id']}")
```

### cURL
```bash
curl -X POST http://127.0.0.1:8000/v1/iso/record-tip \
  -H "Content-Type: application/json" \
  -d '{
    "tip_tx_hash": "0xabc123",
    "chain": "flare",
    "amount": "0.001",
    "currency": "FLR",
    "sender_wallet": "0xSender",
    "receiver_wallet": "0xReceiver",
    "reference": "demo:tip:1"
  }'
```

## Web UI (web-alt)

Start the UI:
```bash
cd web-alt
npm install
npm run dev
```

Access at: http://localhost:3000

### Health Monitoring
```bash
# Check server health
curl http://127.0.0.1:8000/v1/health

# Check database connectivity
# Look for "Application startup complete" in server logs
```

## Production Deployment

### Environment Variables
```bash
# Required for anchoring
FLARE_RPC_URL=https://flare-api.flare.network/ext/C/rpc
ANCHOR_CONTRACT_ADDR=0x0690d8cFb1897c12B2C0b34660edBDE4E20ff4d8
ANCHOR_PRIVATE_KEY=0x<your_private_key_here>

# Optional
ANCHOR_LOOKBACK_BLOCKS=1000
ALLOW_ORIGINS=http://localhost:3000,http://127.0.0.1:3000
DATABASE_URL=postgresql+psycopg://user:pass@localhost/iso_mw
```

### Docker Deployment
```bash
docker compose up --build
```

Services:
- API: http://localhost:8000
- UI (web-alt, separate dev server): http://localhost:3000
- PostgreSQL: localhost:5432

## Security Considerations

### Data Privacy
- Only cryptographic hashes are stored on-chain
- Sensitive data remains in the database
- Evidence bundles are signed for integrity

### Best Practices
- Use HTTPS in production
- Implement proper authentication
- Monitor for unusual activity
- Regular security audits
- Keep dependencies updated

## Support

For technical support:
- Check the troubleshooting guide
- Review server logs for errors
- Test with debug endpoints
- Verify environment configuration

## Advanced Topics

### Authentication Roadmap (API Keys now, JWT later)
- Current default: API keys via X-API-Key for write endpoints (server-to-server), optional for read.
- When to adopt JWT/OIDC:
  - Multiple third-party integrators
  - Need user/service scopes, expirations, SSO
  - Multi-tenant separation based on claims
- How (future):
  - Validate JWT using provider JWKs (OIDC)
  - Map claims → tenant/scopes; enforce exp/nbf
  - Example FastAPI dependency (pseudocode):
    - Decode token, verify signature and expiry
    - Extract tenant_id/scope claims and attach to request context
  - Keep API keys for service automation and rotation as needed

### Fiat-Facing FX Options ✅ Infrastructure, 🔜 Provider Integration

- **Baseline (crypto-native)** ✅: pain.001 InstdAmt/@Ccy = crypto ticker (e.g., FLR). Acceptable for PoC/sandbox; not ISO 4217.
- **Recommended (fiat-facing)** ✅: EqvtAmt + XchgRateInf
  - Keep crypto InstdAmt
  - Add EqvtAmt in fiat ISO 4217 (USD/EUR) and XchgRateInf (rate, source, timestamp)
  - Persist rate/timestamp/provider in DB and include in bundle manifest for auditability
  - Rounding: bankers rounding to 2 decimals for fiat EqvtAmt; preserve full-precision crypto amount
- **Alternative (fiat InstdAmt)** ✅: Set InstdAmt to fiat and move the crypto amount into Remittance/AdditionalData
- **FX source** ⚠️:
  - **PoC** 🔜: HTTP provider (e.g., CoinGecko) cached 1–5 minutes - *planned*
  - **Prod-ish** 🔜: On-chain oracle (e.g., Chainlink), fallback to HTTP with signed/audited responses - *planned*
  - **Current**: FX policy configuration exists, but no actual provider integrations yet implemented

> **Note**: The FX infrastructure and message format support is complete, but actual price feed integrations (CoinGecko, Chainlink) are planned for future implementation.

### Storage Backends: IPFS and Arweave ✅

- **Local (default)** ✅: artifacts served under /files/{id}
- **IPFS (recommended optional)** ✅:
  - **Status**: Fully implemented via app/storage.py module
  - **Why**: developer-friendly, CIDs for content-addressed integrity, easy pinning (web3.storage, Pinata)
  - **How**: upload evidence.zip, store CID; optionally anchor the zip hash and include CID in metadata; enable verify-by-CID
  - **Setup**: Set `IPFS_TOKEN` env var with web3.storage API token
- **Arweave (permanent archive)** ✅:
  - **Status**: Fully implemented via app/storage.py module
  - **Why**: long-term persistence (200+ years) and public audit trace
  - **How**: fund wallet; upload bundle; store TX id; optionally anchor both the zip hash and the Arweave TX hash
  - **Setup**: Set `ARWEAVE_POST_URL` and `BUNDLR_AUTH` env vars
  - **Considerations**: cost/latency vs IPFS; when compliance/audit permanence is a priority

> **Full Documentation**: See [docs/STORAGE.md](docs/STORAGE.md) for comprehensive setup guide, cost comparison, and troubleshooting.

**Configuration Example:**
```json
{
  "evidence": {
    "store": {
      "mode": "ipfs",  // or "arweave" or "local"
      "files_base": "https://cdn.example.com"  // optional
    }
  }
}
```

### Multi-Tenant Readiness ⚠️ Project-Level Isolation, 🔜 Full Multi-Tenancy

- **Today** ⚠️: Project-based isolation
  - Each project has its own configuration, API keys, and receipts
  - Projects are isolated but share the same database tables
  - Users can register multiple projects via SIWE
- **Future model** 🔜: Full multi-tenancy
  - Tenants table; API keys bound to tenant; org_config and receipts scoped by tenant_id
  - All queries filtered by tenant; artifacts/storage segregated per tenant
  - SDKs generated per-tenant with their configuration baked into clients
- **API implications**:
  - Tenant context from JWT claims or API key mapping
  - Per-tenant rate limits, webhooks, and message families

> **Note**: The current project-based isolation provides good separation for most use cases. Full multi-tenancy is planned for enterprise deployments with strict tenant isolation requirements.

For an end-to-end roadmap and phase-by-phase tasks, see [DEVELOPMENT_PLAN.md](DEVELOPMENT_PLAN.md) and [docs/FEATURE_STATUS.md](docs/FEATURE_STATUS.md).

## License

MIT License - see LICENSE file for details.
