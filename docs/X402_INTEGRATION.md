# x402 Payment Protocol Integration Guide

## Overview

The x402 payment protocol enables autonomous AI agents to make micropayments for API access. This implementation uses USDC on Base chain for instant, low-cost payments.

## Architecture

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│ XMTP Agent  │────────▶│ x402 Payment │────────▶│ ISO MW API  │
│             │ 1. Pay  │  Facilitator │ 2. Call │             │
└─────────────┘         └──────────────┘         └─────────────┘
       │                        │                        │
       │                        │                        │
       ▼                        ▼                        ▼
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│ Base Chain  │         │ USDC Contract│         │  Database   │
│             │         │              │         │             │
└─────────────┘         └──────────────┘         └─────────────┘
```

## Payment Flow

### 1. Agent Initiates Request

```typescript
const result = await client.verifyBundle(bundleUrl);
```

### 2. Payment Processing

```typescript
// Step 1: Transfer USDC on Base
const txHash = await wallet.transfer({
  to: recipient,
  amount: parseUnits(price, 6), // USDC has 6 decimals
  token: USDC_CONTRACT
});

// Step 2: Create payment proof
const proof = {
  tx_hash: txHash,
  amount: price,
  recipient: recipient,
  currency: 'USDC',
  chain: 'base'
};
```

### 3. API Call with Proof

```typescript
const response = await axios.post(endpoint, data, {
  headers: {
    'X-PAYMENT': JSON.stringify(proof)
  }
});
```

### 4. Server-Side Verification

```python
@require_payment("0.001", RECIPIENT)
async def premium_endpoint(request: Request):
    # Payment automatically verified by decorator
    # Proceed with operation
    return result
```

## Protected Endpoints

### Available Premium Endpoints

| Endpoint | Price | Description |
|----------|-------|-------------|
| `/v1/x402/premium/verify-bundle` | 0.001 USDC | Verify evidence bundle |
| `/v1/x402/premium/generate-statement` | 0.005 USDC | Generate camt.052/053 |
| `/v1/x402/premium/iso-message/{type}` | 0.002 USDC | Get specific ISO message |
| `/v1/x402/premium/fx-lookup` | 0.001 USDC | FX rate lookup |
| `/v1/x402/premium/bulk-verify` | 0.010 USDC | Bulk verification |
| `/v1/x402/premium/refund` | 0.003 USDC | Initiate refund |

## Configuration Endpoints

### Get Pricing

```bash
GET /v1/x402/pricing
```

Response:
```json
[
  {
    "path": "/v1/x402/premium/verify-bundle",
    "price": "0.001",
    "currency": "USDC",
    "recipient": "0x..."
  }
]
```

### Update Pricing (Admin)

```bash
POST /v1/x402/pricing
Content-Type: application/json

[
  {
    "path": "/v1/x402/premium/verify-bundle",
    "price": "0.002",
    "recipient": "0x...",
    "enabled": "true"
  }
]
```

### Get Revenue Analytics

```bash
GET /v1/x402/revenue?days=7
```

Response:
```json
{
  "total_revenue": "1.234",
  "payment_count": 456,
  "days": 7,
  "by_endpoint": [
    {
      "endpoint": "/v1/x402/premium/verify-bundle",
      "count": 123,
      "revenue": "0.123"
    }
  ]
}
```

## Implementation Guide

### Backend Integration

#### 1. Add Payment Decorator

```python
from app.x402 import require_payment

@require_payment("0.001", RECIPIENT_ADDRESS)
@router.post("/v1/x402/premium/my-endpoint")
async def my_premium_endpoint(request: Request):
    # Payment verified automatically
    return {"result": "success"}
```

#### 2. Configure Pricing

```python
# In database or config
endpoint = ProtectedEndpoint(
    path="/v1/x402/premium/my-endpoint",
    price=Decimal("0.001"),
    currency="USDC",
    recipient="0x...",
    enabled="true"
)
```

### Frontend/Agent Integration

#### TypeScript Client

```typescript
import { ISOMiddlewareClient } from './x402/client';

const client = new ISOMiddlewareClient({
  apiUrl: 'http://localhost:8000',
  x402Recipient: '0x...',
  chainRpcUrl: 'https://mainnet.base.org',
  usdcContract: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',
  wallet: ethersWallet
});

// Automatically handles payment
const result = await client.verifyBundle(bundleUrl);
```

#### Python Client

```python
from iso_middleware_sdk import Client

client = Client(
    api_url="http://localhost:8000",
    x402_enabled=True,
    wallet_private_key="0x...",
    recipient="0x..."
)

# Automatically handles payment
result = client.verify_bundle(bundle_url)
```

## Security Considerations

### Payment Verification

1. **Transaction Validation**
   - Verify transaction exists on-chain
   - Confirm correct recipient
   - Validate amount matches price

2. **Replay Protection**
   - Store transaction hashes
   - Reject duplicate payments
   - Time-based expiry

3. **Amount Verification**
   ```python
   if payment.amount < required_amount:
       raise HTTPException(402, "insufficient_payment")
   ```

### Best Practices

1. **Rate Limiting**
   - Limit requests per agent
   - Prevent abuse
   - Track usage patterns

2. **Monitoring**
   - Log all payments
   - Alert on failures
   - Track revenue metrics

3. **Error Handling**
   - Clear error messages
   - Refund on failures
   - Retry logic

## Testing

### Development Mode

```python
# Disable actual payments for testing
X402_MOCK_MODE = True

# Use test recipient
X402_TEST_RECIPIENT = "0xtest..."
```

### Integration Tests

```python
async def test_paid_endpoint():
    # Create mock payment
    payment_proof = {
        "tx_hash": "0xtest...",
        "amount": "0.001",
        "recipient": RECIPIENT,
        "currency": "USDC",
        "chain": "base"
    }
    
    response = await client.post(
        "/v1/x402/premium/verify-bundle",
        json={"bundle_url": url},
        headers={"X-PAYMENT": json.dumps(payment_proof)}
    )
    
    assert response.status_code == 200
```

## Monitoring & Analytics

### Payment Dashboard

Access at: `/agents` → Revenue tab

Metrics:
- Total revenue (7/30/90 days)
- Payment count
- Revenue by endpoint
- Failed payments
- Average payment value

### Agent Analytics

Access at: `/agents` → Select agent → Stats

Metrics:
- Payments made
- Total spent
- Endpoints accessed
- Success rate
- Recent activity

## Troubleshooting

### Common Issues

1. **Payment Not Found**
   - Check transaction hash
   - Verify chain (Base mainnet)
   - Confirm transaction confirmed

2. **Insufficient Payment**
   - Check amount (6 decimals for USDC)
   - Verify pricing endpoint
   - Ensure currency match

3. **Invalid Recipient**
   - Confirm recipient address
   - Check configuration
   - Verify environment variables

### Debug Logs

```python
# Enable debug logging
LOG_LEVEL=debug

# Check payment verification
logger.debug(f"Verifying payment: {tx_hash}")
logger.debug(f"Expected amount: {required_amount}")
logger.debug(f"Actual amount: {payment.amount}")
```

## Production Deployment

### Environment Variables

```bash
# x402 Configuration
X402_RECIPIENT_ADDRESS=0x0690d8cFb1897c12B2C0b34660edBDE4E20ff4d8
X402_CHAIN=base
X402_RPC_URL=https://mainnet.base.org
X402_USDC_CONTRACT=0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913

# Optional
X402_MOCK_MODE=false
X402_MIN_CONFIRMATIONS=1
X402_TIMEOUT_SECONDS=30
```

### Database Migrations

```bash
# Apply x402 tables
alembic upgrade head
```

### Monitoring

```bash
# Revenue tracking
GET /v1/x402/revenue?days=30

# Payment history
GET /v1/x402/payments?limit=100

# Failed payments
grep "payment_failed" logs/app.log
```

## Future Enhancements

1. **Multi-Chain Support**
   - Ethereum mainnet
   - Polygon
   - Arbitrum
   - Optimism

2. **Additional Tokens**
   - USDT
   - DAI
   - Native ETH

3. **Subscription Model**
   - Monthly plans
   - Prepaid credits
   - Volume discounts

4. **Advanced Features**
   - Automated refunds
   - Payment streaming
   - Escrow services
   - Cross-chain payments

## Resources

- x402 Protocol Spec: https://eips.ethereum.org/EIPS/eip-402
- Base Chain Docs: https://docs.base.org
- USDC Contract: https://developers.circle.com/stablecoins/usdc
- XMTP Docs: https://xmtp.org/docs

## Support

For issues or questions:
- GitHub Issues: [repository]/issues
- Discord: [server link]
- Email: support@isomiddleware.com
