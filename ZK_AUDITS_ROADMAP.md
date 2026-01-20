# ZK Proofs & Audits Roadmap (Halo2 + Recursive Proofs)

Status: Roadmap / Design Doc (future development)
Scope: Add zero-knowledge attestations for integrity and policy compliance over ISO 20022 evidence, including recursive aggregation for audit-grade summaries (exchanges/banks).

Contents
- 1) Vision and Audit Use Cases
- 2) Architecture Overview
- 3) Circuits (Halo2)
- 4) Witness & Binding Strategy
- 5) Artifacts and API Surface
- 6) Prover/Verifier Runtime
- 7) UI Integration (Alternative Next.js UI)
- 8) Phased Delivery & Milestones
- 9) Security, Ops, and Compliance
- 10) Open Questions
- 11) References

---

1) Vision and Audit Use Cases

Goals
- Per‑receipt integrity proof: prove knowledge of evidence.zip whose SHA‑256 equals anchored bundle_hash, without revealing the zip.
- Policy/predicate proof: prove selected ISO fields satisfy audit predicates (e.g., currency allowlist, amount ranges) without disclosing full XML.
- Aggregated/recursive proof for audits:
  - Exchange/bank monthly audit: aggregate N per‑receipt proofs into a single proof exposing (sumAmount, count, date window) and policy compliance over the set.
  - User‑scoped audits: bind proof to a user identity (wallet/org account) and a given time window.

Why this matters
- Auditors can verify integrity and policy adherence without handling raw artifacts or PII.
- Clients (exchanges/banks) can publish a single small proof for entire monthly reports with exact, verifiable aggregates.

---

2) Architecture Overview

Commit‑and‑prove model
- Public commitments:
  - bundle_hash: SHA‑256(evidence.zip) (already anchored on-chain)
  - H_fields: canonical transcript hash of extracted ISO fields (derived off-chain)
  - userIdHash: hash of user/wallet identity
  - time window: [startDate, endDate] as public inputs
- Private witnesses:
  - Evidence preimage bits/chunks used to recompute bundle_hash inside the circuit (see “Witness & Binding”).
  - Selected ISO field values for policy‑proof circuits.
- Constraints/proofs:
  - Integrity: SHA‑256 preimage circuit ensures knowledge of evidence matching bundle_hash.
  - Policy: field values satisfy predicates (ranges, membership, allowlists).
  - Aggregation/recursion: combine k proofs (binary tree) into one final proof exposing set‑wide aggregates (sumAmount, count, window, userIdHash).

Recursion
- Use Halo2 recursion patterns (via snark‑verifier) to fold multiple per‑receipt proofs.
- Final public inputs carry audit‑level aggregates for simplified verification.

---

3) Circuits (Halo2)

Curves/SRS
- Curve: bn254 (KZG commitments), compatible with EVM verifier generation if desired later.
- SRS: universal setup (document SRS provenance/version in vk.json).

Hash gadgets
- SHA‑256 inside circuit for bundle preimage (matches anchored hash exactly).
- Optional: Poseidon commitments for auxiliary commitments; keep primary audit target SHA‑256 to match current anchor semantics.

Circuit profiles
- Integrity circuit (per‑receipt):
  - Public: bundle_hash
  - Private: evidence preimage (chunk inputs), recompute SHA‑256 → must equal bundle_hash.
- Policy circuit (per‑receipt):
  - Public: bundle_hash, H_fields (canonical hash of selected fields)
  - Private: field values; prove (a) H_fields correctness and (b) predicates: e.g., amount ≤ threshold, currency ∈ allowlist, country ∈ allowlist.
- Aggregate/recursive circuit:
  - Inputs: child proof public inputs
  - Public: sumAmount, count, startDate, endDate, userIdHash (and optionally Merkle root of included receipt_ids)
  - Constraint: outputs equal combination of children’s outputs; maintain window/user binding.

Performance guidance
- Integrity with SHA‑256 is heavier—start with small block chunking and optimize later.
- Policy circuits use fewer constraints; keep predicate library modular.

---

4) Witness & Binding Strategy

Bundle binding
- Preimage directly in circuit is heavy; we pass fixed‑size chunks (e.g., 1–4 KB) over a constrained interface.
- Circuit recomputes SHA‑256 over the chunk stream; ensure deterministic chunking and padding.
- Alternative (later): Merkle over chunk hashes; in‑circuit verify Merkle root = SHA‑256(evidence.zip).

Field binding (canonicalization)
- Off‑chain canonicalization: parse ISO XML (e.g., pain001.xml), build a canonical JSON of selected fields with a strict schema and order; compute H_fields = SHA‑256(canonical_json).
- Circuit proves (a) provided private fields hash to H_fields and (b) fields satisfy predicates.
- Store H_fields in public.json for transparent verification of selective disclosures.

User/time binding
- Public inputs: userIdHash = H(userId) and [startDate, endDate].
- Aggregate circuit ensures only receipts belonging to user/time window are folded (enforced by child proofs).

---

5) Artifacts and API Surface

Artifacts
- Per‑receipt proofs:
  - artifacts/<rid>/proofs/<profile>/proof.json
  - artifacts/<rid>/proofs/<profile>/public.json (bundle_hash, H_fields, predicates, etc.)
  - artifacts/<rid>/proofs/<profile>/vk.json (verifying key; include SRS info)
- Aggregate proofs:
  - artifacts/audits/<scope>/<period>/agg-proof.json
  - artifacts/audits/<scope>/<period>/agg-public.json (sumAmount, count, start/end dates, userIdHash, optional merkle root)

Proposed endpoints (API)
- Generate per‑receipt proof:
  - POST /v1/iso/zk/prove-receipt
    - Request: { receipt_id: string, profile: "integrity" | "policy", predicates?: {...} }
    - Response: { proof_url, public_url, vk_url }
- Generate aggregate proof:
  - POST /v1/iso/zk/prove-aggregate
    - Request: { receipt_ids: string[], predicates?: {...}, window?: {start, end}, user_id?: string }
    - Response: { agg_proof_url, agg_public_url, vk_url }
- Verify a proof (off‑chain):
  - POST /v1/iso/zk/verify
    - Request: { proof_url?: string, public_url?: string, vk_url?: string, inline_proof?: {...}, inline_public?: {...}, inline_vk?: {...} }
    - Response: { valid: boolean, public: {...}, errors?: string[] }
- List proofs for a receipt:
  - GET /v1/iso/zk/proofs/{rid}
- Optional: Anchor aggregate proof hash on‑chain:
  - POST /v1/iso/zk/anchor-aggregate { agg_public_url | hash }

Versioning
- Include circuit id/version, SRS digest, vk hash in public.json for audit traceability.

---

6) Prover/Verifier Runtime

Runtime design
- Rust workspace (zkp/):
  - circuits/: halo2 circuit implementations (integrity, policy, aggregate)
  - prover/: CLI to read inputs (zip path, fields, predicates) and write proof/public/vk JSON
  - verifier/: CLI or lib to verify proof against vk/public

FastAPI integration
- Server calls prover/verifier via subprocess (Phase 1), later replace with pyo3 bindings for performance.
- Proof generation can be queue‑based (background tasks) due to compute time.

Performance notes
- Proving may take seconds→minutes depending on circuit size; batch jobs recommended for aggregates.
- Memory/CPU requirements documented; consider Dockerized prover if needed.

---

7) UI Integration (Alternative Next.js UI)

Non‑invasive additions (Verify section)
- “Proofs (ZK)” panel:
  - Generate proof (per receipt/profile) → triggers POST /v1/iso/zk/prove-receipt
  - Generate aggregate proof for selected receipts/date range/user → POST /v1/iso/zk/prove-aggregate
  - Verify proof files → POST /v1/iso/zk/verify
  - Show public inputs summary and download links
- Streamlit tab can get a minimal port later if needed.

---

8) Phased Delivery & Milestones

Phase 0: Spec & Docs (this document)
- Define canonicalization schema and public inputs
- Document profiles and API surface

Phase 1: Per‑receipt Integrity Proof (Halo2)
- Circuit: SHA‑256 preimage = bundle_hash
- Prover/verifier CLIs; FastAPI endpoints: prove‑receipt (integrity) + verify
- Artifacts in artifacts/<rid>/proofs/integrity/

Phase 2: Policy Predicates
- Extend circuit with H_fields + predicate constraints (amount, currency, country code sets)
- Add profile=policy support in prove‑receipt

Phase 3: Recursive/Aggregate Proof
- Aggregator circuit (binary tree folding). Public outputs: {sumAmount, count, window, userIdHash}
- Endpoints: prove‑aggregate, verify; optional on‑chain anchor for agg proof hash
- Audit packaging for monthly reports

Phase 4: UI Integration
- Next.js “Proofs (ZK)” panel: generate/verify/list
- Optional Streamlit parity for key flows

Phase 5 (Optional): EVM Verifier
- Use snark‑verifier to produce an on‑chain verifier for bn254
- Publish verifier contract; add endpoint to submit proofs on‑chain

Milestones
- M1: Integrity proof end‑to‑end demo on a sample receipt
- M2: Policy proof with at least 2 predicates
- M3: Aggregate proof over ≥ 100 receipts within practical proving time
- M4: UI integration and auditor UX demo
- M5: Optional EVM on‑chain verification

---

9) Security, Ops, and Compliance

- SRS provenance: document source and digest; store in vk.json for each circuit build.
- Secrets:
  - None in UI/DB; proof generation is a server or privileged client task.
  - Only aggregate proof anchoring might require a private key (existing anchor_mode/key_ref pattern).
- Logging/auditability:
  - Log proof requests: who, when, scope, window, receipt_ids (or merkle root), predicates summary.
- Compliance:
  - Policy predicate library reviewed; ensure consistent canonicalization with schema checks.
- Performance/DoS:
  - Rate limit proof generation endpoints; queue long‑running tasks.

---

10) Open Questions

- Canonicalization spec: final selection of ISO fields and schema for H_fields.
- Chunking strategy for evidence preimage: flat SHA‑256 vs Merkle over chunks.
- SRS ceremony and distribution for different environments.
- Cost/latency targets for M3 aggregation scale; hardware sizing and parallel proving strategies.
- Minimal on‑chain verifier or keep verification off‑chain with audit signatures + anchor.

---

11) References

- Halo2 / snark‑verifier examples and recursion patterns
- ISO 20022 schemas and selected fields for canonicalization
- KZG and universal SRS considerations for production
- Selective disclosure (SD‑JWT/BBS+) as complementary approach (not strictly ZK)

---

Implementation status
- Not implemented yet. This document captures the intended design and a phased plan for future development.
