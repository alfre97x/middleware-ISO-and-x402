# UI Features (web-alt) – Projects, Receipts Scoping, AI assistant, SDK

This doc focuses on the **web-alt** UI (Next.js) flows and how they map to backend endpoints.

> Design goal: **No API keys in browser JS**.
> All privileged calls go through `web-alt/app/api/proxy/*` so the server can inject `X-API-Key` from an httpOnly cookie.

---

## 1) Run locally

Backend:
```bash
python -m uvicorn app.main:app --reload --port 8000
```

UI:
```bash
cd web-alt
npm install
npm run dev
```

Open:
- http://localhost:3000

Key env vars:
- API: `PUBLIC_BASE_URL` should match the public host used by the UI (for SIWE domain binding).
- UI: `NEXT_PUBLIC_API_BASE` (defaults to `http://127.0.0.1:8000`).

---

## 2) Projects + wallet auth (SIWE) + multi-project cookie store

### Register first project
UI component: `web-alt/components/ProjectsPanel.tsx`

Flow:
1. UI calls `GET /api/proxy/v1/auth/nonce`.
2. UI builds an EIP-4361 SIWE message and signs it using the wallet.
3. UI calls `POST /api/store/register` which calls backend `POST /v1/projects/register`.
4. Backend verifies SIWE and returns:
   - `project` info
   - `api_key` (returned **once**)
5. UI server stores `api_key` in an httpOnly cookie (`iso_projects`) via `web-alt/lib/server/auth.ts`.

### Switching projects
- The cookie stores a list of projects + an `active_project_id`.
- All `/api/proxy/*` requests will attach the `X-API-Key` for the **active project**.

### Global active-project indicator
- Header shows: `Project: <name>` (or `(none)`).
- AI Assistant panel also shows the active project.

---

## 3) Receipts scoping (mine vs all)

Backend endpoint:
- `GET /v1/receipts?scope=mine|all`

Rules:
- `scope=mine` is the default.
- `scope=all` requires an **admin key**. Non-admin keys will get HTTP 403.

UI:
- Dashboard has a `Scope` selector.
- If user selects `all` and receives 403, the UI falls back to `mine` and shows a message.

---

## 4) API Keys management (project-level)

Backend endpoints:
- `GET /v1/auth/api-keys` – list keys for the active project (admin keys see all projects)
- `POST /v1/auth/api-keys` – create key (secret returned once in response header `X-API-Key`)
- `DELETE /v1/auth/api-keys/{id}` – revoke

UI:
- Panel: `web-alt/components/APIKeysPanel.tsx`
- Notes:
  - Calls go through `/api/proxy` so the request is authenticated by the active project cookie key.
  - New secrets are returned **once** and must be copied immediately.

---

## 5) Tenant anchoring UI (awaiting_anchor → confirm)

Backend endpoint:
- `POST /v1/iso/confirm-anchor` with `{ receipt_id, flare_txid, chain? }`

UI:
- Dashboard highlights `awaiting_anchor` status.
- Receipts table includes an action button “Confirm anchor” for those rows.
- Submits through `/api/proxy/v1/iso/confirm-anchor`.

---

## 6) AI Assistant (project-safe)

Endpoint:
- `POST /api/proxy/v1/ai/assist`

Scope toggles in UI:
- `allow_read_receipts`
- `allowed_receipt_ids` (optional extra restriction)
- `allow_read_artifacts`

Important security behavior:
- The backend now enforces **principal scoping** in AI tools.
  - project keys cannot read other projects' receipts via the AI endpoint.
- Even with OpenAI enabled, the provider only receives **scope-filtered tool outputs**.

Useful prompts:
- `SDK help (ts)`
- `List receipts`
- `Receipt <id>`

---

## 5) SDKs

### Internal SDK for the UI
- web-alt uses a local workspace package: `packages/sdk` (`iso-middleware-sdk`).
- It points to `baseUrl=/api/proxy` so the Next server can inject `X-API-Key`.

### Generated SDK zips
Backend:
- `POST /v1/sdk/build` returns a zip for TS or Python.

UI:
- The Dashboard includes “SDK Builder + OpenAPI”.

---

## 7) Per-project anchoring contract deployment (Factory + MetaMask)

The web-alt UI now includes a **Project Config** panel (`web-alt/components/ProjectConfigPanel.tsx`) that can:
- connect to MetaMask,
- call `EvidenceAnchorFactory.deploy()`,
- parse the `AnchorDeployed` event to learn the new `EvidenceAnchor` address,
- save it into the project config via `PUT /v1/projects/{id}/config`.

Notes:
- The factory address is entered in the UI (or prefilled with `NEXT_PUBLIC_ANCHOR_FACTORY_ADDR`).
- After deployment the UI sets `anchoring.execution_mode=tenant` and upserts the configured chain contract.

### Dev deploy factory
Deploy the factory with:
```bash
node scripts/deploy_factory.js --rpc <RPC_URL> --pk <PRIVATE_KEY>
```

Then either paste the address into the UI, or set:
- `web-alt/.env.local`: `NEXT_PUBLIC_ANCHOR_FACTORY_ADDR=0x...`

---

## 8) Notes / best practices

- Do **not** add `NEXT_PUBLIC_API_KEY` in production.
- Prefer:
  - SIWE registration
  - httpOnly cookie storage
  - `/api/proxy` for all frontend calls
