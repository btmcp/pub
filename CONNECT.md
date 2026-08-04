# Connect to BittensorMCP — Zero to First Call

This guide takes you from zero to a successful first MCP tool call in under 10 minutes.

---

## 1. Prerequisites

### For reads (free, no key needed)

- An MCP client that supports **Streamable HTTP** or **stdio** (Claude Desktop, Cursor, VS Code + Cline, Windsurf, Zed, etc.)
- Nothing else — reads are free and anonymous

### For writes (premium, self-custodial)

- A **Bittensor wallet** with a funded coldkey
  - You need the coldkey's **sr25519 private key** (mnemonic or raw seed) to sign locally
  - The server never sees your key — you sign every write yourself
- **TAO to spend**: at least `monthlyPriceTao` (check `bittensor_premium_status` for the current rate, typically ~0.1 TAO/month)

> **Self-custody in one sentence:** your private key stays on your machine; the server only receives signatures.

---

## 2. Get a JWT / Connect Your MCP Client

### Step 2a: Authenticate (get a JWT)

BittensorMCP uses wallet-based auth: you sign a challenge with your sr25519 coldkey, the server returns a 30-day JWT.

**Option A: Browser (fastest for humans)**

1. Go to [https://bittensormcp.com/connect](https://bittensormcp.com/connect)
2. Install a Bittensor-compatible browser extension (Polkadot.js, Talisman, or SubWallet)
3. Create or import a wallet
4. Sign the challenge message
5. Copy the JWT token (starts with `btmcp_...`)

**Option B: Node.js SDK (for agents / headless)**

```bash
npm install @bittensormcp/sign
```

```javascript
import { generateWallet, authenticate } from '@bittensormcp/sign';

// 1. Create a wallet (or load existing from mnemonic)
const wallet = await generateWallet();
console.log('Address:', wallet.ss58);
console.log('Mnemonic (save this):', wallet.mnemonic);

// 2. Authenticate — works even with an unfunded wallet
const { token } = await authenticate({
  endpoint: 'https://bittensormcp.com',
  ss58: wallet.ss58,
  signer: wallet.sign,
});
console.log('JWT:', token);
```

**Option C: Manual (advanced)**

1. `POST https://bittensormcp.com/api/auth/challenge` with `{ ss58, domain: "bittensormcp.com" }` → get `nonce`
2. Sign `bittensormcp-auth:<nonce>:bittensormcp.com` with your sr25519 key
3. `POST https://bittensormcp.com/api/auth/verify` with `{ ss58, nonce, signature }` → get `token`

### Step 2b: Configure your MCP client

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "bittensormcp": {
      "url": "https://bittensormcp.com/api/mcp",
      "headers": { "Authorization": "Bearer YOUR_JWT" }
    }
  }
}
```

**Cursor** (Settings → MCP → Add New):
- **Name**: `bittensormcp`
- **Type**: `HTTP`
- **URL**: `https://bittensormcp.com/api/mcp`
- **Headers**: `{"Authorization": "Bearer YOUR_JWT"}`

**Generic MCP client (stdio bridge for clients without native Streamable HTTP):**

```json
{
  "mcpServers": {
    "bittensormcp": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://bittensormcp.com/api/mcp",
        "--header", "Authorization:${AUTH_HEADER}"
      ],
      "env": { "AUTH_HEADER": "Bearer YOUR_JWT" }
    }
  }
}
```

> **Tip:** You can also pass the key in the URL path: `https://bittensormcp.com/api/mcp/btmcp_YOUR_KEY` — useful for clients that don't support custom headers.

---

## 3. First Read Call

Free reads work immediately — no premium, no wallet funding required. Try these:

### Check your premium status

```
→ bittensor_premium_status {}
```

**Expected output:**
```json
{
  "tier": "FREE",
  "ss58": "5CAAWTPEPUhApsBmvQ2LSAumyRkQbQQsFKDAcF2aVxwSGTRc",
  "writesAvailable": false,
  "subscription": {
    "active": false,
    "premiumDisabled": false,
    "subscriptionValidUntil": null,
    "remainingDays": 0
  },
  "monthlyPriceTao": 0.1,
  "treasuryAddress": "5DTestTreasury...",
  "nextSteps": "Subscription inactive. Send 0.1 TAO from your coldkey to 5DTestTreasury..."
}
```

### Query a balance

```
→ bittensor_balance { "address": "5CAAWTPEPUhApsBmvQ2LSAumyRkQbQQsFKDAcF2aVxwSGTRc" }
```

**Expected output:**
```json
{
  "block": 4928301,
  "truncated": false,
  "data": {
    "address": "5CAAWTPEPUhApsBmvQ2LSAumyRkQbQQsFKDAcF2aVxwSGTRc",
    "balance": {
      "tao": 1.234,
      "rao": 1234000000
    }
  }
}
```

### List subnets

```
→ bittensor_subnet_list { "limit": 5 }
```

**Expected output:**
```json
{
  "block": 4928301,
  "truncated": false,
  "data": {
    "count": 5,
    "subnets": [
      {
        "netuid": 1,
        "name": "Root",
        "symbol": "τ",
        "price_tao": 1.0,
        "emission_tao": 1.25
      }
    ]
  }
}
```

---

## 4. First Write Call (Self-Custody Flow)

Writes require premium + active subscription. The signing is **local** — your key never leaves your machine.

### The A2 two-step flow

```
┌─────────────┐      unsigned payload       ┌─────────────┐
│   Your Key  │  <───────────────────────   │  Bittensor  │
│  (local)    │      sign locally           │    MCP      │
│             │  ───────────────────────>   │   Server    │
└─────────────┘      signature only         └──────┬──────┘
                                                   │
                                            broadcast to
                                                   ▼
                                            ┌─────────────┐
                                            │  Bittensor  │
                                            │   Finney    │
                                            └─────────────┘
```

### Step-by-step: stake 0.01 TAO

**Prerequisites:**
1. Premium activated (send TAO to treasury, confirm via `POST /api/billing/confirm`)
2. `bittensor_premium_status` returns `"writesAvailable": true`
3. Coldkey funded with enough TAO for the stake + fees

**Using the SDK:**

```javascript
import { signAndSubmit } from '@bittensormcp/sign';

const result = await signAndSubmit({
  endpoint: 'https://bittensormcp.com',
  token: 'YOUR_JWT',
  tool: 'bittensor_stake_add',
  args: {
    netuid: 1,
    amount: 0.01,
    // hotkey is optional — omit to auto-pick best validator by yield_per_alpha
  },
  signer: wallet.sign,  // your local sr25519 signer
});

console.log('txHash:', result.txHash);
console.log('block:', result.block);
```

**What happens under the hood:**

1. **Step 1** — `bittensor_stake_add` returns an `UNSIGNED_PAYLOAD` signal:
   ```json
   {
     "signal": "UNSIGNED_PAYLOAD",
     "intent_id": "si_abc123",
     "payload": "0x040300...",  // hex-encoded signing payload
     "expires_at": "2026-08-03T01:17:00Z",
     "hint": "Sign payload (sr25519) with your coldkey and call bittensor_submit_signed"
   }
   ```

2. **Step 2** — You sign the payload bytes locally with your sr25519 coldkey

3. **Step 3** — Submit the signature:
   ```
   → bittensor_submit_signed { "intent_id": "si_abc123", "signature": "0x..." }
   ```

4. **Result** — Server broadcasts the extrinsic and returns:
   ```json
   {
     "txHash": "0x1234...",
     "blockHash": "0xabcd...",
     "block": 4928305
   }
   ```

> **Note:** `bittensor_submit_signed` does not appear in `tools/list` — call it directly after signing.

---

## 5. Troubleshooting: 3 Most Likely First-Run Errors

### Error 1: `UNAUTHORIZED` — "Missing / invalid / revoked API key"

**Cause:** JWT expired, malformed, or the account was deleted.

**Fix:**
- Check the JWT hasn't expired (valid for 30 days)
- Re-authenticate via [bittensormcp.com/connect](https://bittensormcp.com/connect) or the SDK
- Ensure the `Authorization: Bearer YOUR_JWT` header is present (not `apiKey`, not `x-api-key`)

### Error 2: `WRITE_NOT_AVAILABLE` — "write operations require premium"

**Cause:** You're calling a write tool on a free account, or your premium subscription expired.

**Fix:**
1. Call `bittensor_premium_status` to check your tier and subscription state
2. If tier is `FREE`: send TAO to the `treasuryAddress` shown in the status response
3. Call `POST /api/billing/confirm { txHash }` with your transfer's transaction hash
4. Wait for confirmation, then retry the write tool

### Error 3: `INVALID_SIGNATURE` — "Signature verification failed"

**Cause:** You signed the wrong bytes, or used the wrong key (e.g., hotkey instead of coldkey).

**Fix:**
- Ensure you sign the **raw payload bytes** (hex-decode the `payload` field first, don't sign the hex string itself)
- Use your **coldkey** for signing — not the hotkey
- Verify your signer function returns a 64-byte sr25519 signature
- If using `@bittensormcp/sign`, the `signAndSubmit` helper handles hex decoding for you

---

## Quick Reference

| Item | Value |
|---|---|
| **Endpoint** | `https://bittensormcp.com/api/mcp` |
| **Auth** | `Authorization: Bearer <JWT>` header or URL path |
| **Free reads** | 17 tools — no signup, no key, no wallet |
| **Premium writes** | 17 tools — requires subscription + local signing |
| **Signing model** | A2 two-step: `UNSIGNED_PAYLOAD` → local sign → `bittensor_submit_signed` |
| **SDK** | `npm install @bittensormcp/sign` |
| **Full reference** | [https://bittensormcp.com/llms.txt](https://bittensormcp.com/llms.txt) |

## Premium / Billing

For the complete premium activation flow (treasury address, `/api/billing/confirm`, programmatic activation), see the [SignSDK documentation](https://github.com/btmcp/SignSDK) or call `bittensor_premium_status` for your account's current rate and next steps.
