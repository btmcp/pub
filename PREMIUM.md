# BittensorMCP Premium Activation

> Self-serve upgrade from free reads to premium writes. No approval required — send TAO from your own coldkey, confirm the transaction, and write tools activate immediately.

---

## What premium unlocks

### Free tier (28 tools)

All read-only tools are free and require no subscription. The `bittensor_premium_status` tool is also free — use it to check your tier before attempting writes.

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
| `bittensor_neuron` | Neuron info by hotkey (uid, stake, emission, dividends) |
| `bittensor_netuids_for_hotkey` | All subnets where a hotkey is registered |
| `bittensor_owned_hotkeys` | All hotkeys owned by a coldkey |
| `bittensor_premium_status` | Check your own tier, subscription, and billing info (free) |

### Premium tier (23 write tools + 1 premium-only read)

Premium adds on-chain writes via self-custodial A2 signing. Your private key never leaves your device. `bittensor_activity_log` is the only premium-gated read (a free account has no write history to report).

| Tool | What it does |
|------|-------------|
| `bittensor_stake_add` | Stake TAO on a subnet |
| `bittensor_stake_remove` | Unstake alpha from a subnet |
| `bittensor_stake_move` | Move stake between subnets |
| `bittensor_stake_swap` | Swap stake through the AMM (alpha → TAO → alpha) |
| `bittensor_stake_transfer` | Transfer stake to a different coldkey |
| `bittensor_stake_add_limit` | Limit-order stake (price-protected) |
| `bittensor_stake_remove_limit` | Limit-order unstake (price-protected) |
| `bittensor_transfer` | Transfer TAO to another address |
| `bittensor_register` | Register a neuron on a subnet |
| `bittensor_serve_axon` | Set axon endpoint (IP + port) |
| `bittensor_weights_set` | Set validator weights |
| `bittensor_weights_commit` | Commit weights hash (commit-reveal step 1) |
| `bittensor_weights_reveal` | Reveal committed weights (commit-reveal step 2) |
| `bittensor_subnet_register` | Register a new subnet |
| `bittensor_subnet_start_call` | Activate emissions on your subnet |
| `bittensor_subnet_identity_set` | Set subnet branding metadata |
| `bittensor_delegate_take_increase` | Raise validator commission |
| `bittensor_delegate_take_decrease` | Lower validator commission |
| `bittensor_children_set` | Split stake across child hotkeys |
| `bittensor_root_register` | Register on root network (netuid 0) |
| `bittensor_root_claim` | Claim root-network emissions |
| `bittensor_set_auto_stake` | Auto-re-stake rewards to a hotkey |
| `bittensor_activity_log` | Your write history (paginated audit log) — premium-only read |


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
  "ss58": "5YourSs58Address...",
  "writesAvailable": false,
  "subscription": {
    "active": false,
    "premiumDisabled": false,
    "subscriptionValidUntil": null,
    "remainingDays": 0
  },
  "monthlyPriceTao": 0.1,
  "treasuryAddress": "5G7Agn...",
  "nextSteps": "Subscription inactive. Send 0.1 TAO from your coldkey (5Your...) to 5G7Agn.... Then call POST /api/billing/confirm { txHash }."
}
```

#### Step 2: Send TAO to the treasury address

Use any wallet (btcli, Polkadot.js extension, Talisman, etc.). Send **at least** `monthlyPriceTao` to `treasuryAddress`.

#### Step 3: Confirm with the transaction hash

**Prerequisite**: You need a **wallet-session JWT** to call the billing endpoint. The server uses it to verify the transfer sender matches your session's SS58 address. If you don't have one, run the auth flow first (see [Obtaining a wallet-session JWT](#obtaining-a-wallet-session-jwt) below).

```bash
curl -X POST https://bittensormcp.com/api/billing/confirm \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $WALLET_JWT" \
  -d '{"txHash": "0xabc123..."}'
```

**Auth header format**: The endpoint requires the token in the `Authorization: Bearer <token>` header. The server extracts it via `extractApiKey()` (an internal alias for `extractCredential()` that reads `Authorization: Bearer`). `x-api-key` is **not** used.

Expected response (success):

```json
{
  "subscriptionValidUntil": "2026-09-02T12:34:56.789Z",
  "creditedDays": 30
}
```

**Auth requirement**: This endpoint requires a **wallet-session JWT** passed via the `Authorization: Bearer <token>` header (not an API key, not `x-api-key`). The server verifies the transfer sender matches the session's SS58 address.

**Optional field**: You may include `blockHash` from the `isInBlock` callback to speed up verification (sub-second instead of scanning):

```bash
curl -X POST https://bittensormcp.com/api/billing/confirm \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $WALLET_JWT" \
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

### Obtaining a wallet-session JWT

The billing confirm endpoint requires a **wallet-session JWT** (not an API key). The server verifies the transfer sender matches the session's SS58 address. Here's how to get one:

#### Step 1: Request a challenge nonce

```bash
curl -X POST https://bittensormcp.com/api/auth/challenge \
  -H "Content-Type: application/json" \
  -d '{"ss58": "5YourSs58Address...", "domain": "bittensormcp.com"}'
```

Expected response:

```json
{
  "nonce": "a1b2c3d4e5f6..."
}
```

The nonce is valid for **120 seconds** and single-use.

#### Step 2: Sign the challenge message

Create the message to sign:

```
bittensormcp-auth:{nonce}:{domain}
```

Sign this exact string with your sr25519 coldkey. Example using the SDK:

```javascript
import { generateWallet } from '@bittensormcp/sign';

const wallet = await generateWallet();
const message = `bittensormcp-auth:${nonce}:bittensormcp.com`;
const signature = await wallet.sign(message);
```

#### Step 3: Verify and receive JWT

```bash
curl -X POST https://bittensormcp.com/api/auth/verify \
  -H "Content-Type: application/json" \
  -d '{
    "ss58": "5YourSs58Address...",
    "nonce": "a1b2c3d4e5f6...",
    "signature": "0xSignatureHex..."
  }'
```

Expected response:

```json
{
  "token": "eyJhbG...",
  "subscriptionValidUntil": null
}
```

Store the `token` value — this is your `$WALLET_JWT` for the billing confirm endpoint. The token expires in **30 days**; re-authenticate when it expires.

**Implementation source**: `service-a/src/app/api/auth/challenge/route.ts`, `service-a/src/app/api/auth/verify/route.ts`, `service-a/src/lib/session-jwt.ts`.

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
  "ss58": "5YourSs58Address...",
  "writesAvailable": true,
  "subscription": {
    "active": true,
    "premiumDisabled": false,
    "subscriptionValidUntil": "2026-09-02T12:34:56.789Z",
    "remainingDays": 28.5
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
