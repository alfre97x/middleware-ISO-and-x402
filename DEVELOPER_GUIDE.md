# Developer Guide: ISO 20022 Middleware Platform and SDKs

This guide explains how to run the platform, configure it via API or UI, use the generated SDKs, integrate IPFS/Arweave evidence storage, and operate compliance/FX providers. It also covers authentication, database, and production tips.

Contents
1. Overview
2. Architecture
3. Quickstart
4. Authentication (API Keys, SIWE)
5. Configuration (API and UI)
6. Evidence Storage: Local, IPFS, Arweave
7. Database
8. SDKs (TypeScript and Python)
9. Core Flows
10. Compliance & FX Providers
11. OpenAPI
12. Environment Variables
13. Security Guidance (UI vs Secrets)
14. Troubleshooting
15. Production Notes

---

1) Overview

The platform converts ledger “tips” (transactions) into ISO 20022 message artifacts (pain.*, pacs.*, camt.*), bundles cryptographic evidence, anchors it on-chain, and exposes verification and statements.

Key features:
- ISO messages: pain.001, pain.002, pain.007, pain.008 (direct debit), pacs.004/007/008/009, camt.052/053/054, camt.056, camt.029, remt.001.
- Evidence: deterministic zip, manifest/signature, optional W3C VC, optional upload to IPFS/Arweave; multi-chain anchoring.
- Compliance & FX: sanctions/travel-rule hooks; Coingecko and Chainlink rate sources.
- Auth: API Keys; SIWE (Sign-In With Ethereum) for wallet linking.
- SDKs: Self-serve generation (TypeScript, Python). OpenAPI and Swagger UI available.

---

2) Architecture

- FastAPI service: app/main.py (endpoints). Pydantic schemas in app/schemas.py.
- ISO generators: app/iso_messages/*.py and app/iso.py (helper).
- Evidence: app/bundle.py (zip + signatures), app/vc.py (VC).
- Anchoring: app/anchor.py (Python web3), app/anchor_node.py (Node fallback).
- Providers: app/compliance.py (sanctions/travel-rule), app/fx_providers.py (Coingecko/Chainlink).
- Database: app/models.py (SQLAlchemy), app/db.py (engine/session).
- UI: web-alt (Next.js) in web-alt/.
- Artifacts: artifacts/<receipt_id>/ (XML, evidence.zip, optional vc.json, cid/arweave_txid markers).

---

3) Quickstart

Prereqs:
- Python 3.11+ recommended (tested).
- Node.js optional (for Node-based anchor fallback).
- pip.

Install:
```
python -m pip install -r requirements.txt
```

Run API:
```
python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Run UI (web-alt):
```
cd web-alt
npm install
npm run dev
```

Open UI:
- http://localhost:3000
- Defaults to API base http://127.0.0.1:8000 (override with NEXT_PUBLIC_API_BASE)

Health check:
- GET http://127.0.0.1:8000/v1/health

---

4) Authentication

A) API Keys
- Create: POST /v1/auth/api-keys with JSON {"label": "dev-key"}.
  - Response includes key metadata; the actual secret is returned once via the X-API-Key response header.
- Use: add X-API-Key header for subsequent requests.

B) SIWE (optional)
- GET /v1/auth/nonce → returns { nonce, domain }
- Client signs EIP-4361 message; verify:
  - POST /v1/auth/siwe-verify with { "message": "...", "signature": "0x..." }
- Domain binding: validated against PUBLIC_BASE_URL host
- ChainId allowlist: derived from config.ledger.network and config.anchoring.chains
- On success: wallet address stored in LinkedWallet

---

5) Configuration (API and UI)

UI: web-alt provides a Config section (non-secrets) and operator tools.

Config UI provides editors for:
- Security: anchor_mode (managed|self), key_ref
- Anchoring chains: add/edit name, contract, rpc_url
- Organization & Mapping: org.name, org.lei, jurisdiction, charge_bearer, purpose, category_purpose; optional IBAN/BIC/LEI flags and defaults; structured remittance toggle
- FX Policy: mode (none|eqvt_amt|instd_amt_fiat), base_ccy, provider (coingecko|chainlink), chainlink_feed, chainlink_rpc_url
- Evidence Store: store.mode (local|ipfs|arweave), files_base
- Compliance: provider endpoints (http+json:URL or mock:*), threshold, enforcement flags
- ID Strategy & Execution Timing: msg_id/e2e_id/pmt_inf_id strategies (uuid|reference|composite), ReqdExctnDt (immediate|date) with offset days, timezone
- Save pushes to /v1/config

API:
- GET /v1/config
- PUT /v1/config with OrgConfigModel JSON

Minimal example (redacted):
```
{
  "org": { "name": "Capella", "jurisdiction": "SEPA", "lei": null, "default_message_families": ["pain.001","pain.002","camt.054"] },
  "ledger": { "network": "flare", "rpc_url": "https://flare-api.flare.network/ext/C/rpc", "asset": { "symbol": "FLR", "decimals": 18 } },
  "anchoring": { "chains": [{ "name": "flare", "contract": "0x0690d8cFb1897c12B2C0b34660edBDE4E20ff4d8", "rpc_url": null, "explorer_base_url": "https://flarescan.com" }], "lookback_blocks": 50000, "signature_alg": "ed25519" },
  "evidence": { "include": ["pain001.xml","receipt.json","tip.json","manifest.json","public_key.pem"], "sign_over": "zip_without_sig", "store": { "mode": "local" } },
  "fx_policy": { "mode": "none" },
  "id_strategy": { "msg_id_strategy": "uuid", "e2e_id_strategy": "reference", "pmt_inf_id_strategy": "uuid", "reqd_exctn_mode": "immediate", "reqd_exctn_offset_days": 0, "timezone": "UTC" },
  "mapping": { "party_scheme": "WALLET", "account_scheme": "WALLET_ACCOUNT", "charge_bearer": "SLEV", "structured_remittance": false },
  "compliance": { "travel_rule_enforce": false, "sanctions_enforce": false },
  "status": { "emit_pain002": true },
  "integration": { "openapi": true, "webhook_retry": "exponential", "webhook_timeout_ms": 15000 },
  "security": { "auth": "api_key", "rate_limit_per_minute": 100, "anchor_mode": "managed" }
}
```

---

6) Evidence Storage: Local, IPFS, Arweave

Local:
- Default; artifacts written under artifacts/<receipt_id>/.

IPFS (web3.storage):
- Env:
  - IPFS_TOKEN (required to upload)
  - IPFS_GATEWAY (optional; default: https://ipfs.io/ipfs/)
- Flow:
  - After bundling evidence.zip, server POSTs raw bytes to web3.storage.
  - On success: artifacts/<rid>/cid.txt is written.
- Verify:
  - POST /v1/iso/verify-cid with { cid, store?: "ipfs" } returns matches_onchain, vc info, and checksums.

Arweave (Bundlr-like):
- Env:
  - ARWEAVE_POST_URL (required)
  - BUNDLR_AUTH (required)
- Flow:
  - After bundling evidence.zip, server POSTs to ARWEAVE_POST_URL with Authorization: Bearer ...; expects JSON containing { id | txid }.
  - On success: artifacts/<rid>/arweave_txid.txt
- Verify:
  - POST /v1/iso/verify-cid with { cid, store?: "arweave" } returns matches_onchain and enrichments.

UI vs Secrets:
- UI should allow you to select evidence.store.mode (local|ipfs|arweave) and non-secret URLs.
- Do not store tokens in DB/UI. Keep IPFS_TOKEN, BUNDLR_AUTH in server environment variables.

---

7) Database

- SQLAlchemy models in app/models.py:
  - OrgConfig (single row)
  - Receipt (core payment record)
  - ISOArtifact (per artifact metadata)
  - ChainAnchor (anchoring per chain)
  - APIKey, LinkedWallet
- Alembic is included; for dev, the schema auto-creates; for production, manage migrations via Alembic.

---

8) SDKs

A) Generate (UI)
- “SDK Builder + OpenAPI” tab:
  - Choose language: ts | python
  - Packaging: none | npm | pypi
  - “Build SDK” → download zip
  - “Download OpenAPI JSON” and link to /docs are available

B) Generate (API)
- POST /v1/sdk/build with:
```
{ "lang": "ts" | "python", "base_url": "http://127.0.0.1:8000", "packaging": "npm" | "pypi" | "none" }
```

C) TypeScript usage (ISOClient)
```
import { ISOClient } from './src/client';
const api = new ISOClient('http://127.0.0.1:8000', process.env.API_KEY);
const page = await api.listReceipts({ page: 1, page_size: 10 });
console.log(page.items);
```

D) TypeScript usage (ISOApi typed endpoints)
```
import { ISOApi } from './src/api';
const api = new ISOApi('http://127.0.0.1:8000', process.env.API_KEY);

const rec = await api.getReceipt('<rid>');
await api.verify({ bundle_url: `http://127.0.0.1:8000${rec.bundle_url}` });

await api.recordTip({
  tip_tx_hash: '0x123...',
  chain: 'flare',
  amount: '0.1',
  currency: 'FLR',
  sender_wallet: '0xSENDER',
  receiver_wallet: '0xRECEIVER',
  reference: 'client:tip:123',
});
```

E) Python usage (ISOClient)
```
from iso_client import ISOClient
api = ISOClient(base_url='http://127.0.0.1:8000', api_key='MY_KEY')
print(api.list_receipts())
```

F) Python usage (ISOApi typed endpoints)
```
from api import ISOApi
client = ISOApi('http://127.0.0.1:8000', 'MY_KEY')
resp = client.verify(bundle_url='http://127.0.0.1:8000/files/<rid>/evidence.zip')
print(resp.get('matches_onchain'))
```

---

9) Core Flows

A) Record a Tip (Create a Receipt)
- POST /v1/iso/record-tip with:
```
{
  "tip_tx_hash": "0x...",
  "chain": "flare",
  "amount": "0.0000001",
  "currency": "FLR",
  "sender_wallet": "0xSENDER",
  "receiver_wallet": "0xRECEIVER",
  "reference": "client:tip:abc",
  "callback_url": "https://your.app/callback"   // optional
}
```
- Background task:
  - Generates pain.001 (and, optionally, remt.001)
  - Bundles evidence.zip; optional VC issuance
  - Optional IPFS/Arweave upload
  - Anchors on-chain (multi-chain if configured)
  - Emits pain.002, optional pacs.002 and camt.054
  - Stores artifacts and publishes SSE
- SSE: GET /v1/iso/events/{rid}

B) Refund / Cancel
- Refund: POST /v1/iso/refund → emits pacs.004 on new refund receipt and processes like a fresh receipt
- Cancel (pain.007): POST /v1/iso/cancel → emits pain.007 (on original), camt.056, camt.029; creates refund record
- Direct debit: POST /v1/iso/pain008/{rid} → generates pain.008 based on the receipt

C) Verify
- By URL: POST /v1/iso/verify with { "bundle_url": "http://.../files/<rid>/evidence.zip" }
- By CID: POST /v1/iso/verify-cid with { "cid": "<ipfs-cid|arweave-txid>", "store": "ipfs" | "arweave" }
  - Returns matches_onchain, flare_txid/anchored_at, vc_present/vc_url, arweave_txid (if known), issuer (from VC), and checksums (content_sha256, bundle_sha256, zip_size_bytes, vc_sha256?).

---

10) Compliance & FX

FX providers:
- Coingecko:
  - Configure fx_policy.provider="coingecko", fx_policy.base_ccy="EUR"
  - TTL cache used (FX_CACHE_TTL)
- Chainlink:
  - fx_policy.provider="chainlink"
  - fx_policy.chainlink_feed="0x..." and fx_policy.chainlink_rpc_url (optional; ledger.rpc_url fallback)
- fx.json artifact records { rate, source, ts }.

Compliance providers:
- travel_rule_provider and sanctions_provider accept:
  - "http+json:https://..." → server POSTs payload; expects { decision: "allow"|"flag"|"deny", reason }
  - "mock:deny_if_amount_gt:N", "mock:deny_all", "mock:flag_all"
- Enforcement:
  - travel_rule_enforce/sanctions_enforce cause "deny" to short-circuit processing and mark receipt "failed".

---

11) OpenAPI

- Swagger UI: http://127.0.0.1:8000/docs
- OpenAPI JSON: http://127.0.0.1:8000/openapi.json
- The UI (SDK tab) provides a “Download OpenAPI JSON” button and link to Swagger.

---

12) Environment Variables (selected)

General:
- PUBLIC_BASE_URL: used for SIWE domain binding and for external URL prefix in callbacks
- ALLOW_ORIGINS: comma-separated CORS allowed origins for browser clients (default http://localhost:3000,http://127.0.0.1:3000)
- ARTIFACTS_DIR: default "artifacts"

Auth:
- API_KEYS (optional comma-separated list for env-based keys)

Anchoring:
- ANCHOR_PRIVATE_KEY (when security.anchor_mode="self")
- FLARE_RPC_URL, ANCHOR_CONTRACT_ADDR (may be overridden per chain by config)

Evidence:
- IPFS_TOKEN (upload)
- IPFS_GATEWAY (reads; default https://ipfs.io/ipfs/)
- ARWEAVE_POST_URL, BUNDLR_AUTH (for Arweave upload)

FX:
- COINGECKO_ID_FLR (override asset id; default "flare-networks")
- FX_CACHE_TTL (seconds; optional)

---

13) Security Guidance (UI vs Secrets)

- Safe in UI / DB:
  - Non-secret configuration (mapping, id strategies, fx policy parameters, evidence.mode, chainlink feed address, compliance endpoints if public)
- Keep strictly in server env (not in DB/UI):
  - IPFS_TOKEN, BUNDLR_AUTH, ARWEAVE_POST_URL
  - ANCHOR_PRIVATE_KEY
  - Any provider authentication headers or secrets

---

14) Troubleshooting

- “anchor_lookup_unavailable” in verify:
  - Ensure ledger.rpc_url is set and reachable; verify anchoring contract address matches the chain
- SIWE “domain_mismatch”:
  - Set PUBLIC_BASE_URL to your host (e.g., https://mydomain.com)
- Missing VC details in verify-by-cid:
  - vc.json wasn’t issued or not present; verify VC issuance logic
- IPFS/Arweave uploads not visible:
  - Ensure IPFS_TOKEN / ARWEAVE_POST_URL + BUNDLR_AUTH are set; uploads are best-effort by design (won’t fail the main flow)
- Schema validation issues:
  - Check schemas/ presence for XSD validation; ensure xmlschema is installed

---

15) Production Notes

- Reverse proxy (TLS termination) in front of the API; lock down CORS to known origins.
- Externalize DB (Postgres recommended) and configure SQLAlchemy connection in app/db.py.
- Maintain schema with Alembic migrations.
- Harden auth (API keys, rate limits) and monitor with /metrics (Prometheus integration).
- Manage secrets with a proper secrets manager; never store secrets in OrgConfig payloads.

---

11) AI Assistant

Overview:
- The UI includes an AI Assistant side box (visible on every page) that can help:
  - Explain SDK generation and how to integrate the TS/Python clients
  - Inspect receipts and artifacts (vc.json) if explicitly authorized
  - Verify bundles (by URL or CID) and summarize results
- Safety: The assistant operates under explicit, per-session scope toggles. By default, all data-access is disabled.

Backend endpoint:
- POST /v1/ai/assist
  - Request:
    {
      "messages": [{"role":"user"|"assistant"|"system", "content":"..."}],
      "scope": {
        "allow_read_receipts": false,
        "allowed_receipt_ids": [],
        "allow_read_artifacts": false,
        "allow_config_changes": false
      },
      "session_id": "ui-<uuid>",
      "params": { "filters": { "status"?: "...", "chain"?: "..." } }
    }
  - Response:
    { "reply": "...", "used_tools": [{ "tool": "...", "ok": true } ...] }
- Provider configuration is server-side (optional):
  - AI_PROVIDER (openai|azure|ollama|custom)
  - AI_API_KEY (not exposed to UI)
  - AI_BASE_URL (for Azure/OpenRouter/Ollama)
  - AI_MODEL (e.g., gpt-4o-mini, llama3.1:8b)
  - If unset, the assistant replies using local tool results and guide snippets.

Tool registry (scope-enforced):
- get_config() → read-only
- list_receipts(params) → only if allow_read_receipts; respects filters and allowed_receipt_ids
- get_receipt(rid) → respects allow_read_receipts and allowed_receipt_ids
- list_artifacts(rid) → respects allow_read_receipts and allowed_receipt_ids
- read_vc(rid) → only if allow_read_artifacts
- verify({ bundle_url? , bundle_hash? }) and verify_cid({ cid, store? })
- sdk_help({ lang, packaging, base_url? }) → returns usage tips/snippets (no zip creation)

Session logging and auditing:
- Logs are stored under artifacts/ai_sessions/<session_id>.log
- UI provides a “Download session log” link for transparency and audits

Security guidance:
- Secrets must not be stored in UI/DB; provider keys remain in server env
- By default, all toggles are off; the operator must explicitly enable any data access
- Optional redaction can be added if required (e.g., partial wallet masking in outputs)

UI usage:
- Safety & Scope toggles:
  - Allow reading receipts
  - Restrict to selected receipt IDs or filters (status/chain)
  - Allow reading artifacts (vc.json)
  - Allow config changes (dangerous; keep off unless needed)
- Chat prompts:
  - “List receipts”
  - “Receipt <id>”
  - “Verify <bundle_url>”
  - “SDK help (ts)” / “SDK help (python)”
- Session controls:
  - Clear conversation
  - Download session log

16) UI (Next.js)

Location and purpose:
- Path: web-alt/
- Standalone Next.js UI: gradient background, sticky “glass” header, 12-column grid with a persistent right-side AI assistant and left quick links.

Requirements:
- Node.js >= 18.17
- Backend running (FastAPI):
  - python -m uvicorn app.main:app --host 127.0.0.1 --port 8000

Install and run:
- cd web-alt
- npm install
- npm run dev
- Open http://localhost:3000
- Env vars:
  - NEXT_PUBLIC_API_BASE: defaults to http://127.0.0.1:8000
  - NEXT_PUBLIC_API_KEY: optional (for local testing if API key auth is enabled)

Single-page layout and features (center column):
- Dashboard + Recent Receipts table (latest items)
- Verify:
  - By bundle URL → POST /v1/iso/verify
  - By CID → POST /v1/iso/verify-cid with store (auto/ipfs/arweave) and optional receipt_id
  - Displays the full JSON verification response (matches_onchain, txid, checksums, vc hints)
- SDK Builder + OpenAPI:
  - Language: ts | python
  - Packaging: none | npm | pypi
  - Base URL override (defaults to NEXT_PUBLIC_API_BASE)
  - Build SDK → POST /v1/sdk/build and download zip (iso-client-ts.zip | iso-client-py.zip)
  - Download OpenAPI JSON and open Swagger UI
  - Usage snippets for ISOApi/ISOClient (TS/Python)
- Statements:
  - camt.053 daily: GET /v1/iso/statements/camt053 (date)
  - camt.052 intraday: GET /v1/iso/statements/camt052 (date + HH:MM-HH:MM window)
  - Displays count and download link when available
- Config & Auth (non-secrets):
  - Load config → GET /v1/config
  - Save config → PUT /v1/config
  - Quick editors (common fields): security.anchor_mode, security.key_ref, evidence.store.mode, evidence.store.files_base
  - Raw JSON editor (non-secrets only). Secrets remain in server env.

AI assistant (right column):
- Scope toggles: allow_read_receipts, allowed_receipt_ids, allow_read_artifacts (vc.json), allow_config_changes (default off)
- Endpoint: POST /v1/ai/assist
- Session logs saved under artifacts/ai_sessions/<session>.log

Security:
- Do not put secrets in the UI/DB. Provider tokens and private keys remain server-side env variables.
- If API key auth is enabled, export NEXT_PUBLIC_API_KEY only for local development. Prefer server-side protection in production.

Production build:
- cd web-alt && npm run build && npm start
- Ensure NEXT_PUBLIC_API_BASE points to your production API URL.

Troubleshooting:
- Verify failing: check that evidence.zip URL is reachable; confirm backend base URL (NEXT_PUBLIC_API_BASE)
- SDK build failing: check server logs; ensure headers (X-API-Key) are set if required
- CORS: if hosting the UI on another origin, allow http://localhost:3000 or your domain in backend CORS settings or proxy via a reverse proxy
- Node errors: use Node >= 18.17
- Styling: Tailwind classes control the gradient, header glass effect, and card styling; no CSS frameworks required

See also:
- web-alt/README-web-alt.md for a focused quickstart tailored to this UI.
- ZK_AUDITS_ROADMAP.md for the Halo2 recursive-proofs audit roadmap and design.

17) Capabilities and Coverage

ISO 20022 messages implemented
- Customer-to-Bank (pain.*)
  - pain.001 Credit Transfer Initiation
  - pain.002 Customer Payment Status Report
  - pain.007 Payment Cancellation Request
  - pain.008 Direct Debit Initiation
- FI-to-FI (pacs.*)
  - pacs.002 FI to FI Payment Status
  - pacs.004 Payment Return
  - pacs.007 FI to FI Payment Reversal
  - pacs.008 FI to FI Customer Credit Transfer
  - pacs.009 FI to FI Financial Institution Credit Transfer
- Cash Management (camt.*)
  - camt.029 Resolution of Investigation
  - camt.052 Bank To Customer Account Report (intraday)
  - camt.053 Bank To Customer Statement (daily)
  - camt.054 Bank To Customer Debit/Credit Notification
  - camt.056 FI To FI Payment Cancellation Request
- Remittance
  - remt.001 Remittance Information (structured)

Generator modules and artifact names
- pain.001 → app/iso_messages/pain001.py → artifacts/<rid>/pain001.xml
- remt.001 → app/iso_messages/remt001.py → artifacts/<rid>/remt001.xml
- pain.002 → app/iso_messages/pain002.py → artifacts/<rid>/pain002.xml
- pain.007 → app/iso_messages/pain007.py → artifacts/<rid>/pain007.xml
- pain.008 → app/iso_messages/pain008.py → artifacts/<rid>/pain008.xml
- pacs.002 → app/iso_messages/pacs002.py → artifacts/<rid>/pacs002.xml
- pacs.004 → app/iso_messages/pacs004.py → artifacts/<rid>/pacs004.xml
- pacs.007 → app/iso_messages/pacs007.py → artifacts/<rid>/pacs007.xml
- pacs.008 → app/iso_messages/pacs008.py → artifacts/<rid>/pacs008.xml
- pacs.009 → app/iso_messages/pacs009.py → artifacts/<rid>/pacs009.xml
- camt.029 → app/iso_messages/camt029.py → artifacts/<rid>/camt029.xml
- camt.052 → app/iso_messages/camt052.py → artifacts/statements/<date>/camt052-<HHMMHHMM>.xml
- camt.053 → app/iso_messages/camt053.py → artifacts/statements/<date>/camt053.xml
- camt.054 → app/iso_messages/camt054.py → artifacts/<rid>/camt054.xml
- camt.056 → app/iso_messages/camt056.py → artifacts/<rid>/camt056.xml

Attestations and evidence artifacts
- evidence.zip (deterministic bundle) containing receipt/tip/manifest/XML/keys per configuration
- Digital signature over configured scope (e.g., sign_over = zip_without_sig)
- Verifiable Credential vc.json over bundle_hash (issuer captured and returned by verification endpoints when present)
- FX details fx.json (when FX policy enabled)
- Compliance decisions compliance.json (sanctions/travel-rule outputs, decisions, reasons)
- Storage markers:
  - IPFS: artifacts/<rid>/cid.txt (when uploaded via web3.storage)
  - Arweave: artifacts/<rid>/arweave_txid.txt (when uploaded via Bundlr-like endpoint)
- Anchors: on-chain transactions recorded in ChainAnchor table; GET /v1/anchors/{rid}

Verification capabilities
- Verify by bundle URL: POST /v1/iso/verify
  - Integrity/signature checks; on-chain lookup via Python web3 and Node fallback
- Verify by CID: POST /v1/iso/verify-cid (store auto/ipfs/arweave; optional receipt_id)
  - Computes bundle hash from content; on-chain match; enriched output includes:
    - checksums (content_sha256, bundle_sha256, zip_size_bytes, vc_sha256 if available)
    - vc_present, vc_url, arweave_txid (if known), issuer (from VC)
    - matches_onchain, flare_txid, anchored_at, errors[]

Anchoring and storage
- Anchoring:
  - Python web3 primary path; Node fallback (anchor_node)
  - Multi-chain support via anchoring.chains[] (name/contract/rpc_url)
  - Security.anchor_mode with key_ref or self-managed private key via env
- Storage:
  - local (default), ipfs (web3.storage API), arweave (HTTP endpoint with bearer auth)
  - Uploads are best-effort and non-blocking

Compliance and FX integrations
- FX:
  - Coingecko HTTP with TTL caching
  - Chainlink on-chain feed (configurable feed and RPC)
  - fx.json artifact records { base_ccy, quote_ccy, rate, source, ts }
- Compliance providers:
  - http+json:https://… (server posts payload, expects decision: allow|flag|deny and reason)
  - mock strategies: deny_if_amount_gt:N, deny_all, flag_all
  - Enforcement toggles: travel_rule_enforce, sanctions_enforce to block on “deny”

Core processing flows (high level)
- Record Tip → Generate XML (pain.001 [+ remt.001]), bundle and sign, optional VC issuance, optional IPFS/Arweave upload, anchor on-chain, emit status (pain.002, optional pacs.002, camt.054), persist artifacts, publish SSE
- Refund/Return → pacs.004 on a new refund receipt, then processed like new
- Cancellation → pain.007 on original + camt.056/camt.029; creates refund receipt
- Direct Debit → pain.008 generated from a receipt context
- Statements → camt.053 (daily), camt.052 (intraday window) with persisted files under artifacts/statements/<date>/

Security and authentication
- API keys: env and DB-backed; create/list/revoke endpoints
- SIWE (EIP‑4361) verification: nonce, domain binding (PUBLIC_BASE_URL), chainId allowlist from config; EIP‑191 recovery
- Config: GET/PUT /v1/config for non-secrets; all secrets (keys/tokens) remain in server env

SDKs and developer tooling
- SDK generator: POST /v1/sdk/build
  - TypeScript: client + typed API wrappers, optional npm template
  - Python: client + TypedDict models, optional PyPI template
- OpenAPI/Swagger: /openapi.json and /docs; downloadable from UIs

User interface
- Next.js UI (http://localhost:3000, web-alt/):
  - Single-page sections: Recent Receipts, Verify, SDK Builder, Statements, Config (non-secrets)
  - Persistent right-side AI assistant, left quick links

Observability and ops
- /metrics (Prometheus instrumentation if available)
- Static artifacts under /files; embed and redirect routes for receipts (/receipt/{rid}, /embed/receipt)
- CORS configurable via ALLOW_ORIGINS (and reverse proxy in production)

Appendix: Endpoint Summary (Selected)

- POST /v1/iso/record-tip
- GET /v1/iso/receipts/{rid}
- GET /v1/iso/messages/{rid}?type=...
- POST /v1/iso/refund
- POST /v1/iso/cancel
- POST /v1/iso/pain008/{rid}
- POST /v1/iso/pacs007/{rid} | /pacs008/{rid} | /pacs009/{rid}
- GET /v1/iso/statements/camt052?date=YYYY-MM-DD&window=HH:MM-HH:MM
- GET /v1/iso/statements/camt053?date=YYYY-MM-DD
- GET /v1/iso/events/{rid}
- POST /v1/iso/verify
- POST /v1/iso/verify-cid
- GET /v1/config | PUT /v1/config
- GET /v1/auth/nonce | POST /v1/auth/siwe-verify
- POST /v1/auth/api-keys | GET /v1/auth/api-keys | DELETE /v1/auth/api-keys/{id}
- GET /openapi.json | GET /docs
