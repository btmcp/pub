# BittensorMCP First-User Onboarding Funnel

> North star: Active MCP clients (currently 0).  
> This document defines the stages from discovery to first successful MCP call, the measurable events at each stage, and the tooling gaps for tracking.

---

## Funnel Overview

| Stage | Name | What the user does | Success event | Current tracking |
|-------|------|-------------------|---------------|------------------|
| 1 | Discovery | Learns BittensorMCP exists | Landing page load or repo visit | **None** |
| 2 | Intent | Decides to try it | Clicks "Connect" or "Agent" CTA, or starts reading llms.txt | **None** |
| 3 | Auth | Authenticates wallet | Receives JWT token | Server logs only |
| 4 | Config | Configures MCP client | Client successfully connects to `/api/mcp` | **None** |
| 5 | First Call | Makes first tool call | Receives successful tool response | Server logs only |

---

## Stage 1: Discovery

**User state:** Has never heard of BittensorMCP.  
**Goal:** Land on https://bittensormcp.com or the GitHub repo.

### Entry points (verified from live sources)
- **Landing page**: https://bittensormcp.com — Next.js app with hero, tool explorer, connect flow
- **GitHub repo**: https://github.com/btmcp/pub (README.md, CONNECT.md, PREMIUM.md)
- **npm package**: https://www.npmjs.com/package/@bittensormcp/sign
- **llms.txt**: https://bittensormcp.com/llms.txt — machine-readable reference for AI agents
- **MCP directories**: Listed in REGISTRIES.md (submitted to mcp.run, smithery.ai, etc.)

### Measurable event
- `page_load` — landing page or /llms.txt
- `repo_view` — GitHub README view
- `npm_page_view` — npm package page view

### Tracking gap
**No analytics on any channel.** The landing page has no Google Analytics, Plausible, or server-side page-view logging. GitHub provides traffic insights only to repo admins. npm provides download counts but not page-view funnels.

### Recommended tooling
- Server-side request logging in service-a (Next.js middleware or API route wrapper)
- GitHub Traffic API (requires admin token)
- npm download stats via npm registry API

---

## Stage 2: Intent

**User state:** Interested enough to take action.  
**Goal:** Click a CTA or start the onboarding flow.

### Actions that signal intent (from landing page structure)
- Click "Connect" button → navigates to /connect
- Click "Agent" button → scrolls to #start section
- Click "Copy" on the MCP config snippet
- Click "Free" or "Pro" tab in the setup section
- Expand "Already have a wallet?" accordion
- Start reading llms.txt end-to-end (agent path)

### Measurable event
- `cta_click` — any primary or secondary CTA
- `config_copy` — user copies the MCP client config
- `tab_switch` — toggles between Free/Pro setup paths
- `accordion_expand` — reveals advanced config options

### Tracking gap
**No event tracking on the frontend.** The landing page is a static Next.js export with no event instrumentation. We cannot distinguish between a bounce and a user who read the full page.

### Recommended tooling
- Lightweight client-side event tracker (e.g., custom `data-track` attributes + POST to `/api/event`)
- Server-side log of `/connect` route visits
- Count of `llms.txt` downloads vs. page views

---

## Stage 3: Auth

**User state:** Decided to connect. Now needs a JWT.  
**Goal:** Successfully authenticate and receive a wallet-session JWT.

### Auth paths (from CONNECT.md and PREMIUM.md)

**Path A — Browser (humans):**
1. Visit https://bittensormcp.com/connect
2. Install/connect Polkadot.js / Talisman / SubWallet extension
3. Sign challenge message with sr25519 coldkey
4. Copy JWT (starts with `btmcp_...`)

**Path B — Node.js SDK (agents):**
1. `npm install @bittensormcp/sign`
2. `generateWallet()` or load from mnemonic
3. `authenticate({ endpoint, ss58, signer })` → receives JWT

**Path C — Manual (advanced):**
1. `POST /api/auth/challenge` → nonce
2. Sign `bittensormcp-auth:<nonce>:bittensormcp.com`
3. `POST /api/auth/verify` → JWT

### Measurable event
- `auth_challenge_requested` — `POST /api/auth/challenge`
- `auth_success` — `POST /api/auth/verify` returns 200 with token
- `auth_failure` — verify returns 4xx (bad signature, expired nonce, etc.)
- `sdk_install` — npm download of `@bittensormcp/sign`

### Tracking gap
Server logs exist for API calls, but **no structured funnel analytics.** We can count challenge/verify requests from server logs, but cannot correlate them with landing page visits or distinguish new vs. returning users.

### Recommended tooling
- Add `funnel_id` or `referrer` parameter to `/api/auth/challenge` (passed from landing page)
- Structured logging with user session correlation
- npm download trends via registry API

---

## Stage 4: Config

**User state:** Has JWT. Now must configure their MCP client.  
**Goal:** MCP client successfully connects to `https://bittensormcp.com/api/mcp`.

### Config paths (from README.md and CONNECT.md)

**Claude Code:**
```bash
claude mcp add --transport http bittensormcp https://bittensormcp.com/api/mcp --header "Authorization: Bearer YOUR_JWT"
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

**mcp-remote bridge (stdio):**
```json
{
  "mcpServers": {
    "bittensormcp": {
      "command": "npx",
      "args": ["mcp-remote", "https://bittensormcp.com/api/mcp", "--header", "Authorization:${AUTH_HEADER}"],
      "env": { "AUTH_HEADER": "Bearer YOUR_JWT" }
    }
  }
}
```

### Measurable event
- `mcp_client_connected` — client makes successful `POST /api/mcp` (tools/list or initialization)
- `config_error` — client fails to connect (bad JWT, wrong transport, CORS, etc.)

### Tracking gap
**No client-side telemetry.** We see HTTP requests to `/api/mcp` but cannot distinguish:
- A configured client making its first call vs. a health check
- Which MCP client software is being used (Claude, Cursor, ChatGPT, etc.)
- Whether the user successfully saw the tool list in their UI

### Recommended tooling
- Add `User-Agent` parsing or a custom `X-Client-Name` header to `/api/mcp`
- Log `tools/list` requests as a "client configured" milestone
- Track `mcp-remote` npm downloads as a proxy for stdio-bridge usage

---

## Stage 5: First Call

**User state:** MCP client is connected and showing tools.  
**Goal:** Execute first successful tool call.

### First call paths

**Free tier (no premium, no wallet funding):**
- `bittensor_balance` — check any address balance
- `bittensor_subnet_list` — list subnets
- `bittensor_premium_status` — check own tier

**Premium tier (after activation):**
- `bittensor_stake_add` — stake TAO
- `bittensor_transfer` — send TAO
- Any write tool (requires A2 signing flow)

### Measurable event
- `first_tool_call` — first `tools/call` request per JWT
- `first_successful_response` — tool returns structured content (not error)
- `tool_category` — read vs. write vs. premium-status check

### Tracking gap
**No per-user milestone tracking.** Server logs show all tool calls, but we do not flag the "first call" for each JWT. We cannot compute conversion rate from "authenticated" to "active user."

### Recommended tooling
- Database table or Redis flag: `first_call_at` per JWT
- Event stream: emit `milestone.first_call` when a JWT makes its first `tools/call`
- Segment/PostHog-style event ingestion (can be lightweight custom implementation)

---

## Key Drop-Off Points (Hypotheses)

Based on the current onboarding flow, these are the likely friction points:

| Stage transition | Friction | Evidence |
|------------------|----------|----------|
| Discovery → Intent | User doesn't understand MCP | Landing page assumes MCP knowledge; no "What is MCP?" explainer |
| Intent → Auth | Wallet setup complexity | Requires Polkadot.js/Talisman/SubWallet or Node.js SDK; no email/password fallback |
| Auth → Config | JWT copy-paste error | Manual config requires pasting token into JSON; easy to truncate or misformat |
| Config → First Call | Client doesn't support Streamable HTTP | Many clients (ChatGPT, some VS Code extensions) need mcp-remote bridge; user may not know |
| First Call → Premium | Funding barrier | Requires TAO on-chain; user must acquire TAO externally |

---

## Measurement Gaps Summary

| Gap | Impact | Priority |
|-----|--------|----------|
| No landing page analytics | Cannot measure discovery→intent conversion | P1 |
| No frontend event tracking | Cannot measure CTA clicks, config copies | P1 |
| No user session correlation | Cannot trace a single user through stages | P1 |
| No "first call" milestone | Cannot compute activation rate | P2 |
| No client identification | Cannot optimize per-client onboarding | P2 |
| No error funnel logging | Cannot diagnose where users drop off | P2 |

---

## Recommended Tracking Architecture (Definition Only)

This section defines what we need to track, not how to build it.

### Events to instrument

```
page_view          { url, referrer, user_agent }
cta_click          { button_id, page_url }
auth_challenge     { ss58_prefix, path: browser|sdk|manual }
auth_success       { ss58_prefix, path }
auth_failure       { ss58_prefix, path, error_code }
mcp_connect        { jwt_prefix, client_hint }
tools_list         { jwt_prefix, tool_count }
first_tool_call    { jwt_prefix, tool_name, tier: free|premium }
premium_activate   { ss58_prefix, amount_tao, credited_days }
```

### Metrics to compute

- **Discovery rate**: Unique landing page loads per day
- **Intent rate**: CTA clicks / page loads
- **Auth rate**: Successful auths / CTA clicks
- **Config rate**: MCP connections / successful auths
- **Activation rate**: First tool calls / MCP connections
- **Premium conversion**: Premium activations / first tool calls
- **Time-to-first-call**: Duration from page load to first tool call

### Data sources available today

| Source | What it has | What it lacks |
|--------|-------------|---------------|
| Server logs (service-a) | HTTP requests, status codes, JWT prefixes | No session correlation, no frontend events |
| npm registry | `@bittensormcp/sign` download counts | No install-to-usage funnel |
| GitHub (admin-only) | Repo traffic, clone counts | No correlation to active users |
| PostgreSQL (service-a) | Auth sessions, billing records | No per-user milestone flags |

---

## Appendix: Source References

All claims in this document are derived from files in the workspace:

| File | What it documents |
|------|-------------------|
| `repos/pub/README.md` | Landing page content, quick start, tool overview |
| `repos/pub/CONNECT.md` | Step-by-step connection for all clients |
| `repos/pub/PREMIUM.md` | Premium activation flow, billing endpoints |
| `repos/bittensormcp-server/docs/premium-upgrade.md` | Premium upgrade guide, rate limits |
| Live `https://bittensormcp.com` | Landing page structure, CTAs, tool explorer |
| Live `https://bittensormcp.com/llms.txt` | Agent onboarding script, complete tool reference |

---

*Document version: 2026-08-08*  
*Scope: Funnel definition and measurement gaps only. Does not include tracking implementation.*
