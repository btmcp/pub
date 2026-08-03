# BittensorMCP Premium Activation

> Self-serve upgrade from free reads to premium writes. No approval required — send TAO from your own coldkey, confirm the transaction, and write tools activate immediately.

---

## What premium unlocks

### Free tier (17 tools)

All read-only tools are free and require no subscription:

| Tool | What it does |
|------|-------------|
| `bittensor_balance` | Free TAO balance of an SS58 address |
| `bittensor_account` | Balance + staked positions per subnet |
| `bittensor_portfolio` | Portfolio valued at current alpha price |
| `bittensor_subnet_list` | List subnets with price, pools, emission, volume |
| `bittensor_subnet_metagraph` | Metagraph: top neurons by stake |
| `bittensor_subnet_hyperparams` | Subnet hyperparameters |
| `bittensor_subnet_cost` | Current TAO cost to create a subnet |
| `bittensor_subnet_directory` | Subnet on-chain identity (name, repo, contact, URL) |
| `bittensor_balance_batch` | Batch balance lookup (2–50 addresses) |
| `bittensor_validators` | Validators ranked by yield_per_alpha |
| `bittensor_dynamic` | Alpha price, AMM pools, moving price, volume |
| `bittensor_swap_sim` | Simulate TAO↔alpha swap |
| `bittensor_block` | Current block height and hash |
| `bittensor_blocks_until_next_epoch` | Blocks until next epoch for a subnet |
| `bittensor_epoch_status` | Full epoch timing overview |
| `bittensor_blocks_since_last_update` | Staleness check for a neuron |
| `bittensor_premium_status` | Check your own tier, subscription, and billing info |

### Premium tier (+17 tools)

Premium adds on-chain writes via self-custodial A2 signing. Your private key never leaves your device.

| Tool | What it does |
|------|-------------|
| `bittensor_stake_add` | Stake TAO on a subnet |
| `bittensor_stake_remove` | Unstake alpha from a subnet |
| `bittensor_stake_move` | Move stake between subnets |
| `bittensor_transfer` | Transfer TAO to another address |
| `bittensor_register` | Register a neuron on a subnet |
| `bittensor_serve_axon` | Set axon endpoint (IP + port) |
| `bittensor_weights_set` | Set validator weights |
| `bittensor_weights_commit` | Commit weights hash (commit-reveal step 1) |
| `bittensor_subnet_register` | Register a new subnet |
| `bittensor_subnet_start_call` | Activate emissions on your subnet |
| `bittensor_subnet_identity_set` | Set subnet branding metadata |
| `bittensor_delegate_take_increase` | Raise validator commission |
| `bittensor_delegate_take_decrease` | Lower validator commission |
| `bittensor_children_set` | Split stake across child hotkeys |
| `bittensor_root_register` | Register on root network (netuid 0) |
| `bittensor_root_claim` | Claim root-network emissions |
| `bittensor_activity_log` | Your write history (paginated audit log) |

Premium-only reads: `bittensor_activity_log` is gated behind premium (a free account has no write history to report).

---

## Price

- **Monthly price**: `0.1 TAO` (source: `MONTHLY_PRICE_TAO` env var, default 0.1)
- **Overpayment is credited proportionally**: 0.2 TAO = 60 days, 0.15 TAO = 45 days
- **Treasury address**: returned by `bittensor_premium_status` (see below)

---

## Activation flow

There are two paths: **agent path** (programmatic, recommended) and **manual path** (browser or external wallet).

### Agent path (recommended)

Use the SDK's `activatePremium()` function. No browser, no manual steps.

```javascript
import { generateWallet, authenticate, activatePremium } from '@bittensormcp/sign';

const wallet = await generateWallet();
// ... fund the wallet with TAO ...

const { token } = await authenticate({
  endpoint: 'https://bittensormcp.com',
  ss58: wallet.ss58,
  signer: wallet.sign
});

const { subscriptionValidUntil, creditedDays, txHash } = await activatePremium({
  endpoint: 'https://bittensormcp.com',
  token,
  signer: wallet.sign,
  amountTao: 0.1  // optional, defaults to 0.1
});

console.log(`Premium active until ${subscriptionValidUntil} (${creditedDays} days credited)`);
```

What `activatePremium()` does internally:
1. `POST /api/billing/sign-transfer { amountTao }` → receives unsigned payload + intent_id
2. Signs payload locally with sr25519
3. `POST /api/billing/submit-signed { intent_id, signature }` → submits to chain + credits atomically

### Manual path

Send TAO from your coldkey to the treasury address, then confirm with the transaction hash.

#### Step 1: Get your status and the treasury address

Call `bittensor_premium_status` via MCP:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "bittensor_premium_status",
    "arguments": {}
  }
}
```

Expected response (free user):

```json
{
  "tier": "FREE",
  "writesAvailable": false,
  "subscription": {
    "active": false,
    "remainingDays": 0,
    "subscriptionValidUntil": null
  },
  "monthlyPriceTao": 0.1,
  "treasuryAddress": "5G7Agn...",
  "nextSteps": "Subscription inactive. Send 0.1 TAO from your coldkey (5Your...) to 5G7Agn.... Then call POST /api/billing/confirm { txHash }."
}
```

#### Step 2: Send TAO to the treasury address

Use any wallet (btcli, Polkadot.js extension, Talisman, etc.). Send **at least** `monthlyPriceTao` to `treasuryAddress`.

#### Step 3: Confirm with the transaction hash

```bash
curl -X POST https://bittensormcp.com/api/billing/confirm \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_WALLET_JWT" \
  -d '{"txHash": "0xabc123..."}'
```

Expected response (success):

```json
{
  "subscriptionValidUntil": "2026-09-02T12:34:56.789Z",
  "creditedDays": 30
}
```

**Auth requirement**: This endpoint requires a **wallet-session JWT** (not an API key). The server verifies the transfer sender matches the session's SS58 address.

**Optional field**: You may include `blockHash` from the `isInBlock` callback to speed up verification (sub-second instead of scanning):

```bash
curl -X POST https://bittensormcp.com/api/billing/confirm \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_WALLET_JWT" \
  -d '{
    "txHash": "0xabc123...",
    "blockHash": "0xdef456..."
  }'
```

#### Error responses

| Status | Signal | Meaning |
|--------|--------|---------|
| 400 | `INVALID_INPUT` | Missing or malformed txHash |
| 400 | `SENDER_MISMATCH` | Transfer sender does not match your session SS58 |
| 400 | `WRONG_DESTINATION` | Transfer was not sent to the treasury address |
| 400 | `INSUFFICIENT_AMOUNT` | Amount below `monthlyPriceTao` |
| 401 | `WALLET_SESSION_REQUIRED` | Use wallet-session JWT, not API key |
| 409 | `ALREADY_REDEEMED` | This txHash was already credited |
| 503 | `TREASURY_ADDRESS not configured` | Server misconfiguration |
| 504 | `FINALITY_TIMEOUT` | Chain service could not verify finalization within timeout |

---

## How to verify premium status

Call `bittensor_premium_status` at any time:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "bittensor_premium_status",
    "arguments": {}
  }
}
```

Expected response (premium, active):

```json
{
  "tier": "PREMIUM",
  "writesAvailable": true,
  "subscription": {
    "active": true,
    "remainingDays": 28.5,
    "subscriptionValidUntil": "2026-09-02T12:34:56.789Z"
  },
  "monthlyPriceTao": 0.1,
  "treasuryAddress": "5G7Agn...",
  "nextSteps": "Writes available. Sign with your sr25519 coldkey — your key never leaves your machine."
}
```

Use this **before** attempting write operations to avoid errors. No chain call — fast, always available.

---

## Renewal and extension

### How renewal works

- **Proportional credit**: The server computes credited days as `floor((amountRao * 30) / monthlyRao)` using integer RAO math to avoid float drift.
- **Extension**: If you have an active subscription, new payments extend from the current expiry date (not from today). Example: 10 days remaining + 30-day payment = 40 days total.
- **New activation**: If expired or free, credit starts from the payment time.

### Renewal flow

The renewal flow is identical to activation:

1. Check `bittensor_premium_status` to see remaining days
2. Send TAO to treasury (any amount ≥ 0.1 TAO)
3. `POST /api/billing/confirm { txHash }` (manual) or use `activatePremium()` (agent)

The `confirm` endpoint and `activatePremium()` function are **idempotent on txHash** — retrying is safe.

### Automatic extension via agent path

```javascript
// Extend by another month
const result = await activatePremium({
  endpoint: 'https://bittensormcp.com',
  token,
  signer: wallet.sign,
  amountTao: 0.2  // 60 days
});
```

---

## Source of truth

This document is derived from the following implementation files:

| What | File | Lines |
|------|------|-------|
| Tool registry (free/premium split) | `service-a/src/lib/mcp/tools.ts` | TOOLS array, `isWrite` / `premiumOnly` flags |
| Premium status tool | `service-a/src/lib/mcp/handler.ts` | `callPremiumStatus()` |
| Billing confirm endpoint | `service-a/src/app/api/billing/confirm/route.ts` | POST handler |
| Billing sign-transfer endpoint | `service-a/src/app/api/billing/sign-transfer/route.ts` | Agent path step 1 |
| Billing submit-signed endpoint | `service-a/src/app/api/billing/submit-signed/route.ts` | Agent path step 2 |
| Credit logic | `service-a/src/lib/billing-credit.ts` | `creditPremium()` |
| Subscription check | `service-a/src/lib/subscription.ts` | `isSubscriptionActive()` |
| Premium config | `service-a/src/lib/config.ts` | `loadPremiumConfig()` |
| SDK activatePremium | `SignSDK/packages/npm/src/index.ts` | `activatePremium()` |

To verify the curl example matches the implementation, diff it against `service-a/src/app/api/billing/confirm/route.ts`.
