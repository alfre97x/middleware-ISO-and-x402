# XMTP Agent Setup & Deployment Guide

## Overview

The ISO Middleware XMTP Agent is an autonomous AI agent that:
- Listens for natural language commands via XMTP messaging
- Automatically handles x402 micropayments for premium operations
- Provides instant access to ISO 20022 middleware functionality
- Operates 24/7 without human intervention

## Quick Start

### Prerequisites

- Node.js 18+ and npm
- Ethereum wallet with private key
- USDC on Base chain (for payments)
- ISO Middleware API access

### Installation

```bash
cd agents/iso-x402-agent
npm install
```

### Configuration

1. Copy environment template:
```bash
cp .env.example .env
```

2. Configure `.env`:
```bash
# XMTP Configuration
XMTP_ENV=dev                     # or 'production'
WALLET_PRIVATE_KEY=0x...         # Your wallet private key

# ISO Middleware API
ISO_MW_API_URL=http://localhost:8000
ISO_MW_API_KEY=your_api_key      # Optional

# x402 Payment
X402_RECIPIENT=0x0690d8cFb1897c12B2C0b34660edBDE4E20ff4d8
CHAIN_RPC_URL=https://mainnet.base.org
USDC_CONTRACT=0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913

# Agent Settings
AGENT_NAME=ISO Middleware Agent
LOG_LEVEL=info                   # debug, info, warn, error
```

### Build & Run

```bash
# Development mode
npm run dev

# Production build
npm run build
npm start
```

## Agent Registration

### Register with Middleware

```bash
curl -X POST http://localhost:8000/v1/agents \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your_api_key" \
  -d '{
    "name": "My ISO Agent",
    "wallet_address": "0x...",
    "xmtp_address": "0x...",
    "pricing_rules": {
      "verify": "0.001",
      "statement": "0.005"
    }
  }'
```

### Verify Registration

Access the UI at: `http://localhost:3000/agents`

You should see your agent listed with:
- ✅ Active status
- Wallet address
- XMTP address
- Creation timestamp

## Usage

### Connecting via XMTP

1. **Using Converse (Mobile)**
   - Install Converse app
   - Create/import wallet
   - Start new conversation with agent's address
   - Send commands

2. **Using XMTP SDK**
   ```typescript
   import { Client } from '@xmtp/xmtp-js';
   
   const client = await Client.create(wallet);
   const conversation = await client.conversations.newConversation(
     AGENT_ADDRESS
   );
   await conversation.send('list 5');
   ```

### Available Commands

#### Free Commands

**List Receipts**
```
list [limit]
```
Example: `list 10` - Shows 10 most recent receipts

**Get Receipt**
```
get <receipt_id>
```
Example: `get abc-123-def` - Shows receipt details

**Help**
```
help
```
Shows all available commands

#### Paid Commands (Auto-payment via x402)

**Verify Bundle**
```
verify <bundle_url>
```
- Cost: 0.001 USDC
- Example: `verify https://ipfs.io/ipfs/Qm...`
- Returns: Validation result with chain confirmations

**Generate Statement**
```
statement <date>
```
- Cost: 0.005 USDC
- Example: `statement 2026-01-20`
- Returns: camt.052/053 statement summary

**Initiate Refund**
```
refund <receipt_id> [reason]
```
- Cost: 0.003 USDC
- Example: `refund abc-123 duplicate payment`
- Returns: Refund confirmation with pacs.004

### Command Examples

```
# Check recent activity
> list 5

📋 Recent Receipts (5):

🧾 PAY-20260120-001
   ID: abc-123-def
   Amount: 100.00 USD
   Status: anchored
   Created: 1/20/2026, 2:00 PM

🧾 PAY-20260120-002
   ...

# Get specific receipt
> get abc-123-def

🧾 Receipt Details

Reference: PAY-20260120-001
ID: abc-123-def
Amount: 100.00 USD
Status: anchored
Chain: flare
From: 0x1234...
To: 0x5678...
Tx Hash: 0xabcd...
Created: 1/20/2026, 2:00 PM
Anchored: 1/20/2026, 2:05 PM

# Verify a bundle (paid)
> verify https://ipfs.io/ipfs/Qm...

⏳ Verifying bundle (paying 0.001 USDC)...

✅ Verification Complete

Valid: ✓ Yes
Bundle Hash: 0x1234...
Anchor Tx: 0x5678...
Chains: flare, ethereum
Timestamp: 1/20/2026, 2:05 PM

💰 Payment: 0.001 USDC paid

# Generate statement (paid)
> statement 2026-01-20

⏳ Generating statement (paying 0.005 USDC)...

✅ Statement Generated

Type: camt.053
Date: 2026-01-20
Transaction Count: 42

💰 Payment: 0.005 USDC paid

# Refund (paid)
> refund abc-123 Customer requested cancellation

⏳ Initiating refund (paying 0.003 USDC)...

✅ Refund Initiated

Original Receipt: abc-123
Refund Receipt: ref-456
Method: REVERSAL
Reason: Customer requested cancellation
Status: pending

💰 Payment: 0.003 USDC paid
```

## Payment System

### How Payments Work

1. **Command Received**
   - Agent parses natural language
   - Identifies required operation
   - Checks if payment needed

2. **Automatic Payment**
   - Transfers USDC on Base chain
   - Creates payment proof
   - Includes proof in API request

3. **Verification**
   - API verifies transaction
   - Records payment
   - Processes request

4. **Response**
   - Agent receives result
   - Formats response
   - Sends to user via XMTP

### Funding the Agent

The agent's wallet needs USDC on Base for payments:

```bash
# Check balance
cast balance --rpc-url https://mainnet.base.org \
  0xYOUR_AGENT_WALLET \
  0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913

# Transfer USDC to agent
# Use Bridge: https://bridge.base.org
# Or direct transfer from exchange
```

Recommended funding:
- Development: 1 USDC (~1000 operations)
- Production: 10-100 USDC

### Payment Tracking

Monitor agent spending:

```bash
# Via API
GET /v1/agents/{agent_id}/stats?days=7

# Via UI
Navigate to: /agents → Select agent → Stats tab
```

## Deployment

### Development

```bash
# Local testing
npm run dev

# View logs
tail -f logs/agent.log
```

### Production

#### Option 1: PM2 (Recommended)

```bash
# Install PM2
npm install -g pm2

# Start agent
pm2 start dist/index.js --name iso-agent

# Monitor
pm2 status
pm2 logs iso-agent

# Auto-restart on reboot
pm2 startup
pm2 save
```

#### Option 2: Docker

```bash
# Build image
docker build -t iso-agent .

# Run container
docker run -d \
  --name iso-agent \
  --env-file .env \
  --restart unless-stopped \
  iso-agent

# View logs
docker logs -f iso-agent
```

#### Option 3: Systemd

```ini
# /etc/systemd/system/iso-agent.service
[Unit]
Description=ISO Middleware XMTP Agent
After=network.target

[Service]
Type=simple
User=agent
WorkingDirectory=/opt/iso-agent
ExecStart=/usr/bin/node dist/index.js
Restart=always
EnvironmentFile=/opt/iso-agent/.env

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl enable iso-agent
sudo systemctl start iso-agent

# Check status
sudo systemctl status iso-agent

# View logs
journalctl -u iso-agent -f
```

### Cloud Deployment

#### AWS (EC2/ECS)

```bash
# EC2 User Data
#!/bin/bash
cd /home/ubuntu/iso-agent
npm install --production
npm run build
pm2 start dist/index.js --name iso-agent
```

#### Google Cloud Run

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]
```

```bash
gcloud run deploy iso-agent \
  --source . \
  --region us-central1 \
  --allow-unauthenticated
```

#### Heroku

```json
// Procfile
worker: node dist/index.js
```

```bash
heroku create iso-agent
heroku config:set WALLET_PRIVATE_KEY=0x...
git push heroku main
heroku ps:scale worker=1
```

## Monitoring

### Health Checks

```typescript
// Add to agent.ts
setInterval(async () => {
  const status = {
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    xmtp_connected: !!this.xmtpClient,
    last_message: this.lastMessageTime
  };
  
  // Send to monitoring service
  await axios.post('https://healthcheck.io/webhook', status);
}, 60000); // Every minute
```

### Logging

```typescript
// Structured logging
logger.info('Payment processed', {
  tx_hash: txHash,
  amount: amount,
  endpoint: endpoint,
  agent_id: agentId
});

// Log levels
LOG_LEVEL=debug  // Development
LOG_LEVEL=info   // Production
LOG_LEVEL=warn   // Critical only
```

### Metrics

Track key metrics:
- Messages received/hour
- Payments processed/day
- Success rate
- Average response time
- Error rate

### Alerts

Set up alerts for:
- Agent offline > 5 minutes
- Low USDC balance (< 1 USDC)
- Payment failures > 5/hour
- High error rate (> 10%)

## Troubleshooting

### Common Issues

#### 1. XMTP Connection Failed

```bash
Error: Failed to initialize XMTP client
```

**Solutions:**
- Check wallet private key format
- Verify XMTP_ENV (dev/production)
- Ensure network connectivity
- Try different RPC endpoint

#### 2. Payment Failed

```bash
Error: Payment failed: insufficient funds
```

**Solutions:**
- Check USDC balance
- Verify chain (Base mainnet)
- Confirm recipient address
- Check gas fees

#### 3. API Connection Failed

```bash
Error: connect ECONNREFUSED 127.0.0.1:8000
```

**Solutions:**
- Verify ISO_MW_API_URL
- Check API is running
- Confirm firewall rules
- Test with curl

#### 4. Command Not Recognized

```bash
❌ Invalid command. Type "help" for available commands.
```

**Solutions:**
- Check command syntax
- See help for examples
- Verify command is lowercase
- Check for typos

### Debug Mode

Enable detailed logging:

```bash
LOG_LEVEL=debug npm run dev
```

Debug output:
```
[2026-01-20T14:20:00.000Z] DEBUG: Received message from 0x1234...
[2026-01-20T14:20:00.100Z] DEBUG: Parsed command: { action: 'verify', args: {...} }
[2026-01-20T14:20:00.200Z] DEBUG: Making payment of 0.001 USDC...
[2026-01-20T14:20:01.000Z] DEBUG: Payment successful: 0xabcd...
[2026-01-20T14:20:01.500Z] DEBUG: API request successful
[2026-01-20T14:20:01.600Z] DEBUG: Sent reply to 0x1234...
```

### Reset Agent

If agent becomes unresponsive:

```bash
# Stop agent
pm2 stop iso-agent

# Clear cache
rm -rf dist/
npm run build

# Restart
pm2 restart iso-agent
```

## Security

### Best Practices

1. **Wallet Security**
   - Use dedicated wallet for agent
   - Keep minimal USDC balance
   - Rotate keys regularly
   - Use hardware wallet for main funds

2. **Environment Variables**
   - Never commit `.env` to git
   - Use secrets management (AWS Secrets Manager, etc.)
   - Rotate API keys periodically
   - Encrypt sensitive data

3. **Access Control**
   - Whitelist allowed users (optional)
   - Rate limit requests
   - Monitor for abuse
   - Log all operations

4. **Network Security**
   - Use HTTPS for API calls
   - Enable firewall
   - Restrict outbound connections
   - Monitor network traffic

### Incident Response

If compromised:

1. **Immediate Actions**
   - Stop agent: `pm2 stop iso-agent`
   - Revoke API keys
   - Transfer remaining USDC
   - Change wallet

2. **Investigation**
   - Review logs
   - Check payment history
   - Identify breach point
   - Document findings

3. **Recovery**
   - Create new wallet
   - Update configuration
   - Deploy new agent
   - Monitor closely

## Advanced Configuration

### Custom Commands

Add new commands by:

1. **Update Parser**
```typescript
// src/utils/parser.ts
if (trimmed.startsWith('custom ')) {
  return { action: 'custom', args: { ... } };
}
```

2. **Create Handler**
```typescript
// src/handlers/custom.ts
export async function handleCustom(client, args) {
  // Implementation
  return response;
}
```

3. **Register in Agent**
```typescript
// src/agent.ts
case 'custom':
  response = await handleCustom(this.isoClient, command.args);
  break;
```

### Multi-Agent Setup

Run multiple agents:

```bash
# Agent 1 - Production
pm2 start dist/index.js \
  --name iso-agent-prod \
  --env production

# Agent 2 - Staging
pm2 start dist/index.js \
  --name iso-agent-staging \
  --env staging
```

### Load Balancing

Use multiple agents for high volume:

```bash
pm2 start dist/index.js -i 4 --name iso-agent-cluster
```

## Resources

- XMTP Docs: https://xmtp.org/docs
- Base Chain: https://docs.base.org
- PM2 Guide: https://pm2.keymetrics.io
- Docker Docs: https://docs.docker.com

## Support

- GitHub Issues: [repository]/issues
- Discord: [server]
- Email: support@isomiddleware.com
