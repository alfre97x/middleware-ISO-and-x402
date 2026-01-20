# User Guide

This guide explains how non-developers (operators/back‑office) and internal admins use the ISO 20022 Middleware UIs.

Who should use which UI?
- web-alt (Next.js): Production UI for operators/back‑office. Global, can read/verify any transaction processed by the middleware.

Prerequisites
- You have a deployed API (FastAPI) URL. Example: https://api.example.com
- If API key auth is enabled, an operator key is issued to you.

web-alt (Production UI)
1) Open the UI in your browser (e.g., https://ui.example.com or http://localhost:3000 for local).
2) Configure API base (optional if pre-configured):
   - Local: set NEXT_PUBLIC_API_BASE before starting, or use a reverse proxy.
   - If API keys are enforced, the admin may preconfigure the UI; otherwise you may be given a key for local testing only.
3) What you can do:
   - Dashboard / Recent Receipts: browse recent items; open details to download XML and evidence bundles.
   - Verify:
     - By Bundle URL: paste the evidence.zip URL from a receipt detail and verify on-chain.
     - By CID: paste an IPFS CID or Arweave TXID; the UI resolves and verifies it.
   - SDK Builder + OpenAPI: download OpenAPI JSON; optionally generate a client SDK archive (developer task).
   - Statements: request camt.053 (daily) and camt.052 (intraday) and download the files.
   - Config (read‑only by default): load and view non‑secret configuration. Admins can save changes when using privileged credentials.

Tips
- Public vs Private API URLs: Use the public API URL in browsers. Private service‑to‑service hostnames are not reachable by browsers.
- Secrets: Do not paste secret tokens or private keys into UI fields. Those belong in backend environment variables only.
- Evidence URLs: For verification, ensure the evidence.zip URL is publicly reachable where appropriate.

Troubleshooting
- Verify fails for a bundle:
  - Check that the bundle URL is accessible and correct
  - Check chain/RPC configuration on the API side
- Statements return 0 or errors:
  - Check dates and windows; consult the backend logs if issues persist
