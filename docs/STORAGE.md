# Storage Backends Documentation

This document covers the evidence bundle storage backends supported by the ISO 20022 Middleware.

## Overview

The middleware supports three storage modes for evidence bundles:

1. **Local** (default) - Files served directly from the API server
2. **IPFS** (recommended for public/decentralized) - Content-addressed storage via IPFS
3. **Arweave** (best for permanence) - Immutable permanent storage

You can configure the storage mode in the organization configuration (`OrgConfig`) or per-project settings.

## Storage Modes Comparison

| Feature | Local | IPFS | Arweave |
|---------|-------|------|---------|
| **Cost** | Server storage | Free + optional pinning | ~$5-10 per GB (one-time) |
| **Durability** | Depends on backups | High (if pinned) | Permanent (200+ years) |
| **Accessibility** | Requires API server | Public gateways | Public gateways |
| **Content Addressing** | No | Yes (CID) | Yes (TX ID) |
| **Setup Complexity** | Low | Medium | Medium-High |
| **Best For** | Development, private | Public audit, sharing | Compliance, legal records |

---

## 1. Local Storage (Default)

### How It Works

Files are stored in the `ARTIFACTS_DIR` directory and served via `/files/{receipt_id}/` endpoints.

### Configuration

```bash
# Environment variable
ARTIFACTS_DIR=/data/artifacts
```

```json
// OrgConfig
{
  "evidence": {
    "store": {
      "mode": "local",
      "files_base": "https://cdn.example.com"  // Optional CDN prefix
    }
  }
}
```

### Pros
- ✅ Simple setup
- ✅ No external dependencies
- ✅ Full control
- ✅ Fast access

### Cons
- ❌ Requires server storage
- ❌ Not content-addressed
- ❌ Requires backups for durability

---

## 2. IPFS Storage (Recommended for Public Use)

### How It Works

Evidence bundles are uploaded to IPFS via web3.storage API and retrievable via public IPFS gateways using their Content ID (CID).

### Setup

1. **Get API Token**:
   - Sign up at https://web3.storage
   - Create an API token
   - Set `IPFS_TOKEN` environment variable

2. **Configure Environment**:
   ```bash
   # Required
   IPFS_TOKEN=eyJhbGc...  # From web3.storage

   # Optional
   IPFS_GATEWAY=https://w3s.link/ipfs/  # Custom gateway
   ```

3. **Update OrgConfig**:
   ```json
   {
     "evidence": {
       "store": {
         "mode": "ipfs"
       }
     }
   }
   ```

### How Files Are Stored

1. Bundle is created locally in `ARTIFACTS_DIR/{receipt_id}/`
2. ZIP file is uploaded to web3.storage
3. CID is returned and saved to `cid.txt` in receipt directory
4. Files remain available locally AND via IPFS

### Accessing Files

**Via API** (local copy):
```
GET /files/{receipt_id}/evidence.zip
```

**Via IPFS Gateway**:
```
https://w3s.link/ipfs/{CID}
https://ipfs.io/ipfs/{CID}
https://cloudflare-ipfs.com/ipfs/{CID}
```

### Verification

```bash
# Verify bundle via CID
POST /v1/iso/verify-cid
{
  "cid": "bafybei...",
  "store": "ipfs",
  "receipt_id": "optional-receipt-id"
}
```

### Pinning Services

For guaranteed availability, pin your content:

- **web3.storage**: Automatic pinning included
- **Pinata**: Manual pinning service
- **Infura IPFS**: Enterprise pinning
- **Self-hosted node**: Run your own IPFS node

### Pros
- ✅ Content-addressed (integrity built-in)
- ✅ Public accessibility
- ✅ Deduplication
- ✅ Low cost (free + optional pinning)
- ✅ Multiple gateways available

### Cons
- ❌ Requires pinning for persistence
- ❌ Gateway dependency for access
- ❌ Not guaranteed permanent (depends on pinning)

---

## 3. Arweave Storage (Permanent Archive)

### How It Works

Evidence bundles are uploaded to Arweave blockchain for permanent, immutable storage. Files are retrievable forever via transaction ID.

### Setup

1. **Get Bundlr/Turbo Account**:
   - Sign up at https://bundlr.network or https://turbo.ardrive.io
   - Fund your account (AR tokens or credit card)
   - Get authentication token

2. **Configure Environment**:
   ```bash
   # Required
   ARWEAVE_POST_URL=https://node2.bundlr.network/tx
   BUNDLR_AUTH=<your-bundlr-token>

   # Optional
   ARWEAVE_GATEWAY=https://arweave.net/  # Custom gateway
   ```

3. **Update OrgConfig**:
   ```json
   {
     "evidence": {
       "store": {
         "mode": "arweave"
       }
     }
   }
   ```

### Cost Structure

- **One-time payment** for permanent storage
- Approximate costs (as of 2026):
  - ~$5-10 per GB
  - Evidence bundles typically 10-100 KB each
  - Example: 10,000 receipts @ 50 KB = 500 MB = ~$3-5 total

### How Files Are Stored

1. Bundle is created locally
2. ZIP file is uploaded to Bundlr/Turbo
3. Transaction ID is returned and saved to `arweave_txid.txt`
4. Files remain available locally AND on Arweave

### Accessing Files

**Via API** (local copy):
```
GET /files/{receipt_id}/evidence.zip
```

**Via Arweave Gateway**:
```
https://arweave.net/{TX_ID}
https://ar-io.net/{TX_ID}
```

### Verification

```bash
# Verify bundle via Arweave TX ID
POST /v1/iso/verify-cid
{
  "cid": "arweave-tx-id-here",
  "store": "arweave",
  "receipt_id": "optional-receipt-id"
}
```

### Pros
- ✅ Guaranteed permanent storage (200+ years)
- ✅ Immutable once stored
- ✅ No ongoing fees or pinning needed
- ✅ Built-in replication
- ✅ Perfect for compliance/legal records

### Cons
- ❌ Upfront cost (though one-time)
- ❌ Slower initial upload
- ❌ Cannot delete or modify
- ❌ More complex setup

---

## Configuration Examples

### Development (Local)
```json
{
  "evidence": {
    "store": {
      "mode": "local"
    }
  }
}
```

### Production (IPFS with Local Fallback)
```json
{
  "evidence": {
    "store": {
      "mode": "ipfs",
      "files_base": "https://your-api.com"
    }
  }
}
```

### Compliance/Legal (Arweave)
```json
{
  "evidence": {
    "store": {
      "mode": "arweave"
    }
  }
}
```

---

## Hybrid Storage Strategy

You can use multiple storage backends:

1. **Local + IPFS**: Fast access + public sharing
2. **Local + Arweave**: Fast access + permanent archive
3. **All Three**: Maximum redundancy

### Implementation

The middleware stores files locally first, then uploads to configured backend. This provides:
- Fast local access during processing
- Public/permanent backup via IPFS/Arweave
- Verification via multiple methods

---

## Verification Workflows

### Verify via Bundle URL (Local)
```bash
POST /v1/iso/verify
{
  "bundle_url": "https://api.example.com/files/{rid}/evidence.zip"
}
```

### Verify via Bundle Hash
```bash
POST /v1/iso/verify
{
  "bundle_hash": "0xcc4cdd738ada83b7d7c04fd8d96415dfd78dfe1f0011b3250fcb508f77632f4f"
}
```

### Verify via IPFS CID
```bash
POST /v1/iso/verify-cid
{
  "cid": "bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi",
  "store": "ipfs"
}
```

### Verify via Arweave TX ID
```bash
POST /v1/iso/verify-cid
{
  "cid": "43-char-arweave-txid",
  "store": "arweave"
}
```

---

## Troubleshooting

### IPFS Upload Fails

**Symptoms**: CID not saved, `cid.txt` missing

**Common Causes**:
1. Missing or invalid `IPFS_TOKEN`
2. web3.storage API rate limits
3. Network connectivity issues
4. File too large (web3.storage has 100MB limit per file)

**Solutions**:
- Verify token: `echo $IPFS_TOKEN`
- Check web3.storage dashboard for quota
- Test manually: `curl -X POST https://api.web3.storage/upload -H "Authorization: Bearer $IPFS_TOKEN" -H "Content-Type: application/octet-stream" --data-binary @test.zip`
- Consider splitting large bundles or using Pinata

### Arweave Upload Fails

**Symptoms**: Arweave TX ID not saved, `arweave_txid.txt` missing

**Common Causes**:
1. Missing or invalid `BUNDLR_AUTH`
2. Insufficient funds in Bundlr account
3. Network issues
4. Invalid `ARWEAVE_POST_URL`

**Solutions**:
- Check Bundlr balance: Visit dashboard
- Verify auth token is current
- Test endpoint: `curl -X POST $ARWEAVE_POST_URL -H "Authorization: Bearer $BUNDLR_AUTH" --data-binary @test.zip`
- Try alternative endpoint (Turbo instead of Bundlr)

### Gateway Download Fails

**Symptoms**: Verification fails, "bundle not found"

**Common Causes**:
1. Content not yet propagated to gateway
2. Gateway temporarily unavailable
3. Invalid CID/TX ID
4. Content unpinned (IPFS only)

**Solutions**:
- Wait 1-2 minutes for propagation
- Try alternative gateway
- Verify CID format
- Check pinning status (IPFS)

---

## Best Practices

### 1. Choose the Right Backend

- **Development**: Use local
- **Public APIs**: Use IPFS
- **Compliance/Audit**: Use Arweave
- **High Volume**: Use local + periodic archiving to IPFS/Arweave

### 2. Monitor Storage Costs

- IPFS: Mostly free, monitor pinning service quotas
- Arweave: Pre-fund account, set up monitoring

### 3. Implement Fallbacks

```python
# Example: Try IPFS, fallback to local
if mode == "ipfs":
    cid = storage.upload_bundle(zip_path, "ipfs")
    if not cid:
        # IPFS failed, but local file still accessible
        log.warning("IPFS upload failed, using local only")
```

### 4. Document Storage Metadata

The middleware automatically saves storage identifiers:
- `cid.txt` - IPFS Content ID
- `arweave_txid.txt` - Arweave Transaction ID
- `manifest.json` - Bundle contents and hashes

### 5. Verify Storage Success

After upload, verify via the appropriate endpoint:
```python
# Check that CID works
verification = client.verify_cid(cid="...", store="ipfs")
assert verification['matches_onchain'] == True
```

---

## Migration Strategies

### From Local to IPFS

1. Update `OrgConfig` to mode="ipfs"
2. New receipts will upload to IPFS
3. Existing receipts remain local (still accessible)
4. Optional: Batch upload existing bundles to IPFS

### From IPFS to Arweave

1. Update `OrgConfig` to mode="arweave"
2. Fund Bundlr/Turbo account
3. New receipts will use Arweave
4. Optional: Archive critical receipts to Arweave

### Hybrid: Both IPFS and Arweave

Modify `app/jobs.py` to upload to multiple backends:
```python
# Upload to both
ipfs_cid = storage.upload_bundle(zip_path, "ipfs")
ar_txid = storage.upload_bundle(zip_path, "arweave")

# Save both identifiers
if ipfs_cid:
    storage.save_storage_metadata(receipt_dir, ipfs_cid, "ipfs")
if ar_txid:
    storage.save_storage_metadata(receipt_dir, ar_txid, "arweave")
```

---

## Security Considerations

### IPFS

- **Public by default**: All content is publicly accessible
- **CIDs are deterministic**: Same file = same CID (good for deduplication)
- **Metadata**: CID reveals nothing about content (hash-based)
- **Privacy**: Don't upload PII or sensitive data to public IPFS

### Arweave

- **Permanent and public**: Cannot be deleted or modified
- **Immutable**: Perfect for audit trails, bad for sensitive data
- **Cost**: Consider regulatory requirements before committing

### Best Practices

1. **Encrypt sensitive bundles** before uploading to IPFS/Arweave
2. **Use local mode** for PII or regulated data
3. **Audit storage decisions** against compliance requirements
4. **Document storage policies** in your security documentation

---

## Advanced: Custom Gateways

### IPFS Custom Gateway

```bash
# Use your own IPFS node
IPFS_GATEWAY=https://ipfs.mycompany.com/ipfs/
```

### Arweave Custom Gateway

```bash
# Use ar-io gateway
ARWEAVE_GATEWAY=https://ar-io.net/
```

### CDN Integration

```json
{
  "evidence": {
    "store": {
      "mode": "local",
      "files_base": "https://cdn.example.com"
    }
  }
}
```

Files will be served as:
```
https://cdn.example.com/files/{receipt_id}/evidence.zip
```

---

## Monitoring and Metrics

### Track Storage Success Rate

Monitor these metrics:
- IPFS upload success rate
- Arweave upload success rate
- Gateway availability
- Download success rate

### Example Prometheus Queries

```promql
# IPFS upload success rate
rate(ipfs_upload_success_total[5m]) / rate(ipfs_upload_attempts_total[5m])

# Arweave costs
sum(arweave_upload_bytes_total) * 0.000000010  # Approximate cost in AR
```

---

## API Reference

### Storage Module Functions

```python
from app.storage import upload_bundle, download_bundle, get_storage_backend

# Upload to IPFS
cid, mode = upload_bundle("/path/to/bundle.zip", mode="ipfs")

# Download from IPFS
content = download_bundle(cid, mode="ipfs")

# Auto-detect storage mode
content = download_bundle("bafybei...", mode="auto")  # Detects IPFS
content = download_bundle("43-char-txid", mode="auto")  # Detects Arweave
```

### Environment Variables Reference

```bash
# Local Storage
ARTIFACTS_DIR=artifacts

# IPFS Storage
IPFS_TOKEN=eyJhbGc...               # web3.storage API token (required)
IPFS_GATEWAY=https://w3s.link/ipfs/  # Gateway URL (optional)

# Arweave Storage
ARWEAVE_POST_URL=https://node2.bundlr.network/tx  # Upload endpoint (required)
BUNDLR_AUTH=<token>                                 # Auth token (required)
ARWEAVE_GATEWAY=https://arweave.net/               # Gateway URL (optional)
```

---

## FAQ

### Q: Can I use multiple storage backends simultaneously?

A: Yes, but requires custom code. The default behavior uses one mode at a time. You can modify `app/jobs.py` to upload to multiple backends and save all identifiers.

### Q: What happens if IPFS/Arweave upload fails?

A: The bundle is still saved locally and the receipt continues processing. Storage upload is best-effort and won't block anchoring.

### Q: How do I migrate existing receipts to IPFS/Arweave?

A: Write a migration script that:
1. Lists all receipt IDs
2. For each receipt, reads `evidence.zip`
3. Uploads to IPFS/Arweave
4. Saves CID/txid to metadata files

### Q: Can I delete files from IPFS?

A: Content can be "unpinned" from your pinning service, but may remain accessible via other nodes. Arweave is permanent and cannot be deleted.

### Q: What's the recommended setup for production?

A: Start with local mode, add IPFS for public receipts when needed. Consider Arweave for high-value transactions or regulatory compliance.

### Q: Do I need to anchor CIDs on-chain?

A: No. The middleware anchors the bundle hash (SHA-256 of ZIP file). The CID is additional metadata for retrievability. However, you can optionally anchor both for maximum auditability.

---

## Support

For storage-related issues:
- Check environment variables are set correctly
- Verify API tokens are valid and have quota
- Test gateways manually with curl
- Review worker logs for upload errors
- Monitor storage success metrics

For web3.storage support: https://web3.storage/docs
For Bundlr support: https://docs.bundlr.network
For Arweave docs: https://ar.io/docs
