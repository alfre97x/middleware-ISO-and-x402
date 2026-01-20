# Agent Anchoring Feature Documentation

## Overview

The Agent Anchoring feature enables AI agents to automatically anchor payment data to the blockchain for immutable audit trails. This provides cryptographic proof of payment processing through the x402 payment protocol.

## Architecture

### Database Schema

**Agent Configuration**
- `auto_anchor_enabled`: Enable automatic anchoring for agent actions
- `anchor_on_payment`: Anchor data when x402 payments are processed
- `anchor_wallet`: Optional dedicated wallet for anchoring transactions

**Anchor Records**
- Links to parent `Agent` via `agent_id`
- Stores anchor transaction data (`anchor_tx_hash`, `anchor_contract`)
- Tracks anchored data hash and blockchain status
- Timestamp tracking for audit purposes

### Backend Components

#### Models (`app/models.py`)
```python
class Agent:
    auto_anchor_enabled: bool = False
    anchor_on_payment: bool = False
    anchor_wallet: Optional[str] = None
    anchors: List[Anchor] = relationship(...)

class Anchor:
    agent_id: str
    anchor_hash: str
    anchor_tx_hash: Optional[str]
    anchor_contract: Optional[str]
    data: dict
    status: str
    created_at: datetime
```

#### API Routes (`app/api/routes/agent_anchoring.py`)

**GET** `/v1/agents/{agent_id}/anchors`
- Lists all anchors for an agent
- Returns anchor metadata and blockchain status

**GET** `/v1/agents/{agent_id}/anchoring-config`
- Retrieves agent anchoring configuration

**PUT** `/v1/agents/{agent_id}/anchoring-config`
- Updates agent anchoring settings
- Request body:
  ```json
  {
    "auto_anchor_enabled": true,
    "anchor_on_payment": true,
    "anchor_wallet": "0x..."
  }
  ```

**POST** `/v1/agents/{agent_id}/anchor-data`
- Manually trigger data anchoring
- Request body:
  ```json
  {
    "data": {...},
    "description": "Payment processing data"
  }
  ```

### Frontend Components

#### AgentAnchoring Component (`web-alt/components/agents/AgentAnchoring.tsx`)

**Features:**
- Toggle automatic anchoring
- Configure anchor-on-payment behavior
- Set dedicated anchoring wallet
- View anchor history with blockchain verification
- Manual data anchoring interface

**UI Sections:**
1. **Configuration Panel**
   - Enable/disable auto-anchoring
   - Toggle payment-triggered anchoring
   - Wallet address configuration

2. **Anchor History**
   - List of all anchors with timestamps
   - Transaction hash links to block explorer
   - Contract address display
   - Status indicators (pending/confirmed/failed)

3. **Manual Anchoring**
   - JSON editor for custom data
   - Description field
   - Submit button for manual anchoring

## Usage Guide

### Enabling Automatic Anchoring

1. Navigate to **Agents** page
2. Select an agent
3. Click **Anchoring** tab
4. Toggle "Enable Automatic Anchoring"
5. (Optional) Toggle "Anchor on x402 Payment"
6. (Optional) Set dedicated anchor wallet
7. Click **Save Configuration**

### Manual Data Anchoring

1. Navigate to **Anchoring** tab
2. Scroll to "Manual Anchoring" section
3. Enter JSON data in the editor
4. Add a description
5. Click **Anchor Data**

### Viewing Anchor History

The anchor history table displays:
- Timestamp of anchoring
- Data hash (truncated with copy button)
- Transaction hash (links to Etherscan)
- Contract address
- Status badge (Confirmed/Pending/Failed)

## Integration with x402 Payment Protocol

When `anchor_on_payment` is enabled:

1. Agent processes x402 payment request
2. Payment data is captured
3. Automatic anchoring is triggered
4. Payment metadata is hashed and submitted to blockchain
5. Transaction hash is stored in anchor record
6. Agent can reference anchor in payment receipts

## Security Considerations

### Wallet Management
- Dedicated anchor wallets recommended for production
- Ensure wallet has sufficient gas for anchoring transactions
- Use hardware wallets for high-value anchoring operations

### Data Privacy
- Only hashes are stored on-chain (not raw data)
- Original data stored in encrypted database
- Consider data sensitivity before anchoring

### Access Control
- Anchoring configuration requires agent ownership
- API endpoints use standard authentication
- Anchor data verification is publicly accessible

## Blockchain Details

### Supported Networks
- Ethereum Mainnet
- Sepolia Testnet
- Base
- Optimism
- Custom EVM-compatible chains

### Smart Contracts
Uses the Evidence Anchor contract system:
- `EvidenceAnchorBasic.sol` for simple anchoring
- `EvidenceAnchorFactory.sol` for agent-specific contracts
- `FactoryRegistry.sol` for contract discovery

## Performance Considerations

### Gas Optimization
- Batch multiple anchors when possible
- Use Layer 2 solutions (Base, Optimism) for lower costs
- Monitor gas prices for optimal submission timing

### Rate Limiting
- Default: 100 anchors per agent per hour
- Configurable via environment variables
- Prevents accidental spam and gas waste

## Monitoring & Debugging

### Logs
Agent anchoring logs include:
```
[AGENT_ANCHOR] agent_id=xxx auto_enabled=true payment_trigger=true
[ANCHOR_TX] hash=0x... contract=0x... status=pending
[ANCHOR_CONFIRMED] hash=0x... block=12345678
```

### Metrics
- Total anchors per agent
- Failed anchor rate
- Average confirmation time
- Gas costs per anchor

### Troubleshooting

**Issue: Anchors stuck in pending**
- Check wallet balance for gas
- Verify network connectivity
- Check gas price settings

**Issue: Anchor verification fails**
- Ensure block explorer API is accessible
- Verify contract deployment
- Check network configuration

## API Examples

### Python SDK
```python
from iso_middleware_sdk import Client

client = Client(api_key="your-key")

# Enable anchoring
client.update_agent_anchoring_config(
    agent_id="agent-123",
    auto_anchor_enabled=True,
    anchor_on_payment=True
)

# Manual anchor
anchor = client.anchor_agent_data(
    agent_id="agent-123",
    data={"payment_id": "pay-456", "amount": 100},
    description="Payment verification"
)

# List anchors
anchors = client.list_agent_anchors(agent_id="agent-123")
```

### REST API
```bash
# Enable anchoring
curl -X PUT http://localhost:8000/v1/agents/agent-123/anchoring-config \
  -H "Content-Type: application/json" \
  -d '{
    "auto_anchor_enabled": true,
    "anchor_on_payment": true,
    "anchor_wallet": "0x..."
  }'

# Manual anchor
curl -X POST http://localhost:8000/v1/agents/agent-123/anchor-data \
  -H "Content-Type: application/json" \
  -d '{
    "data": {"payment_id": "pay-456"},
    "description": "Manual verification"
  }'

# List anchors
curl http://localhost:8000/v1/agents/agent-123/anchors
```

## Future Enhancements

### Planned Features
- [ ] Multi-chain anchoring support
- [ ] Anchor verification webhooks
- [ ] Batch anchoring API
- [ ] Anchor expiration/archival
- [ ] Zero-knowledge proof integration
- [ ] Cross-chain anchor verification

### Integration Opportunities
- Chainlink VRF for verifiable randomness
- IPFS for large data storage
- The Graph for anchor querying
- ENS for anchor contract naming

## References

- [EvidenceAnchor Smart Contract](../../contracts/EvidenceAnchorBasic.sol)
- [Factory Registry](../../contracts/FactoryRegistry.sol)
- [Agent API Documentation](../../API_Documentation.md#agents)
- [x402 Protocol Guide](./X402_INTEGRATION.md)

## Support

For issues or questions:
- GitHub Issues: [Report Bug]
- Documentation: [docs/]
- Community: [Discord/Telegram]
