# BittensorMCP

> Live, self-custodial Bittensor chain access for AI agents via MCP.

[![Endpoint](https://img.shields.io/badge/endpoint-bittensormcp.com-blue)](https://bittensormcp.com)
[![SDK](https://img.shields.io/npm/v/@bittensormcp/sign)](https://www.npmjs.com/package/@bittensormcp/sign)

## What is BittensorMCP?

**BittensorMCP** is a hosted [Model Context Protocol (MCP)](https://modelcontextprotocol.io) server that gives AI agents live access to the Bittensor blockchain. Connect any MCP client — Claude, Cursor, ChatGPT, VS Code, Windsurf, Zed, Codex — and query balances, subnets, metagraphs, validators, and Alpha prices in real time.

- **Free reads**: 17 tools, no signup, no API key
- **Premium writes**: 17 on-chain actions (stake, transfer, register, weights) via self-custodial signing
- **One endpoint**: `https://bittensormcp.com/api/mcp` with Streamable HTTP transport

## Why self-custodial?

Your **private key never leaves your device**.

BittensorMCP uses an A2 two-step signing protocol: the server returns an `UNSIGNED_PAYLOAD` plus `intent_id`, you sign it locally with your sr25519 key, and only the signature goes back. The server broadcasts the signed extrinsic — it never sees, stores, or handles your private key.

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

## Quick start

### 1. Install the SDK

```bash
npm install @bittensormcp/sign
```

### 2. Authenticate (self-custody)

```javascript
import { generateWallet, authenticate } from '@bittensormcp/sign';

const wallet = await generateWallet();
const { token } = await authenticate({
  endpoint: 'https://bittensormcp.com',
  ss58: wallet.ss58,
  signer: wallet.sign
});
```

### 3. Connect your MCP client

**Claude Code:**
```bash
claude mcp add --transport http bittensormcp \
  https://bittensormcp.com/api/mcp \
  --header "Authorization: Bearer YOUR_JWT"
```

**Claude Desktop / Cursor / VS Code:**
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

**Clients without native Streamable HTTP:**
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

For step-by-step connection instructions for every client, see the [Connect Guide](CONNECT.md).

## Tools at a glance

| Category | Count | Examples |
|----------|-------|----------|
| **Free reads** | 17 | `bittensor_balance`, `bittensor_account`, `bittensor_subnet_list`, `bittensor_subnet_metagraph`, `bittensor_validators`, `bittensor_dynamic`, `bittensor_swap_sim`, `bittensor_block` |
| **Premium writes** | 17 | `bittensor_stake_add`, `bittensor_stake_remove`, `bittensor_transfer`, `bittensor_register`, `bittensor_weights_set`, `bittensor_subnet_register`, `bittensor_root_register` |

All write tools use the A2 self-custody signing flow. Premium activation requires sending TAO to the treasury address (check `bittensor_premium_status` for the current rate).

## Architecture

```
┌─────────────┐     HTTPS      ┌─────────────┐     internal     ┌─────────────┐
│   MCP Client │ ─────────────> │  service-a  │ ───────────────> │  service-b  │
│ (Claude/etc) │   Streamable   │  (Next.js)  │   HTTP/WSS       │  (FastAPI)  │
└─────────────┘     HTTP       └─────────────┘                  └──────┬──────┘
                                                                      │
                                                                      ▼
                                                              ┌─────────────┐
                                                              │  Bittensor  │
                                                              │   Finney    │
                                                              └─────────────┘
```

- **service-a** (public): MCP endpoint, auth, web UI, rate limiting
- **service-b** (private): Chain backend, direct Subtensor connection
- **Database**: PostgreSQL for auth state, intents, audit log

## Links

- **Website**: https://bittensormcp.com
- **Connect**: https://bittensormcp.com/connect
- **MCP Endpoint**: `https://bittensormcp.com/api/mcp`
- **SDK**: https://www.npmjs.com/package/@bittensormcp/sign
- **Connect Guide**: [CONNECT.md](CONNECT.md) — step-by-step for all clients
- **Full Agent Reference**: https://bittensormcp.com/llms.txt

## License

MIT
