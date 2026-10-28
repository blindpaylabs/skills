# Build with AI

Connect AI coding agents and AI builders to BlindPay with an MCP server, Agent Skills, and a REST API.

Source: https://blindpay.com/docs/build-with-ai

BlindPay exposes its full payments API through three surfaces that drop into any AI coding agent or AI app builder: an **MCP server**, **Agent Skills**, and a **REST API**. Use them to create quotes, execute payments, manage customers, and configure webhooks directly from the terminal, your IDE, or a no-code builder.

## Prebuilt prompts

Curated prompts that walk an AI coding agent through a full quickstart, with the API gotchas already encoded. Copy one into your agent, or add it to your project rules or `CLAUDE.md`:

<CLLMPrompt src="prompts/quickstart-payin-autonomous.md" displayText="Payin: builds everything, one handoff at the end." />

<CLLMPrompt src="prompts/quickstart-payout-autonomous.md" displayText="Payout: builds everything, one handoff at the end." />

## The three surfaces

### MCP server

The Model Context Protocol server exposes the BlindPay API as callable tools. Any MCP-compatible agent (Claude, ChatGPT, Codex, Claude Code, Cursor, VS Code, Windsurf, and others) can call these tools in response to natural-language prompts.

The hosted server at `https://mcp.blindpay.com/mcp` is the recommended way to connect. You sign in with your BlindPay account and pick the instance the agent may use; no API key is pasted into the client.

```bash [Claude Code]
claude mcp add --transport http blindpay \
  https://mcp.blindpay.com/mcp
```

```bash [Codex]
codex mcp add blindpay --url https://mcp.blindpay.com/mcp
```

```json [.cursor/mcp.json]
{
  "mcpServers": {
    "blindpay": {
      "url": "https://mcp.blindpay.com/mcp"
    }
  }
}
```

```json [.vscode/mcp.json]
{
  "servers": {
    "blindpay": {
      "type": "http",
      "url": "https://mcp.blindpay.com/mcp"
    }
  }
}
```

In Claude or ChatGPT, add a custom connector with `https://mcp.blindpay.com/mcp` (ChatGPT needs Developer mode and OAuth authentication), then sign in. In Grok, open grok.com/connectors, choose **New connector**, then **Custom**, and paste the same URL.

To let an agent explore without being able to change anything, connect to `https://mcp.blindpay.com/mcp/readonly` instead. It serves only the read tools.

When it connects, the server sends the agent instructions covering the call order, amount units (cents for quotes and payouts, whole tokens for wallet balances), the steps a person must do in the dashboard, and how to test on a development instance. It also exposes every webhook event and its payload schema as resources under `blindpay://webhooks/events`.

#### Local server (API key)

For hosts that only run local MCP servers, or for CI, run the npm package with an API key instead:

```bash [Codex]
codex mcp add blindpay \
  --env BLINDPAY_API_KEY=YOUR_API_KEY \
  --env BLINDPAY_INSTANCE_ID=in_000000000000 \
  -- npx -y @blindpay/mcp
```

```bash [Claude Code]
claude mcp add blindpay \
  --env BLINDPAY_API_KEY=YOUR_API_KEY \
  --env BLINDPAY_INSTANCE_ID=in_000000000000 \
  -- npx -y @blindpay/mcp
```

```toml [Grok Build]
# ~/.grok/config.toml
[mcp_servers.blindpay]
command = "npx"
args = ["-y", "@blindpay/mcp"]
env = { BLINDPAY_API_KEY = "YOUR_API_KEY", BLINDPAY_INSTANCE_ID = "in_000000000000" }
```

For Claude Code or another JSON-configured MCP host, add it directly to `.mcp.json` in your project:

```json [.mcp.json]
{
  "mcpServers": {
    "blindpay": {
      "command": "npx",
      "args": ["-y", "@blindpay/mcp"],
      "env": {
        "BLINDPAY_API_KEY": "YOUR_API_KEY",
        "BLINDPAY_INSTANCE_ID": "in_000000000000"
      }
    }
  }
}
```

### Agent Skills

A packaged knowledge layer that teaches the agent BlindPay's rails, corridors, fees, and API patterns, so generated integrations are correct on the first pass.

```bash [Terminal]
npx skills add blindpaylabs/skills
```

### REST API

For builders that generate code but do not run an MCP host (Lovable, v0, Bolt, Replit), call the REST API directly from a backend route or server action.

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/payouts \
  -H "Authorization: Bearer YOUR_API_KEY"
```

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

## What you can ask

Once connected, the agent can:

- Create payout and payin quotes, and preview fees and FX rates
- Execute payouts across EVM chains, Stellar, and Solana
- Create and verify customers
- Generate virtual accounts
- Configure webhooks
- Query transaction status

```text [Prompt]
Use the BlindPay tools to create a payout quote for 1000 USDC to a USD ACH
bank account. Show me the fees and rate, then execute after I confirm.
```

**Warning:**

Money-movement tools should require explicit confirmation before executing. Every tool carries MCP annotations (read-only, destructive), so clients that support them can ask before running anything that is not a read, and the server instructions tell the agent to confirm amounts and recipients on production instances. The server does not block a call on its own: keep a confirmation step in your agent prompt or app flow, start on a development instance, or connect to `/mcp/readonly` until you are ready.

## Per-platform setup

| Platform | Method | Notes |
| --- | --- | --- |
| Codex | `codex mcp add` | Run `codex mcp add` from the terminal, then load Agent Skills for full scaffolding. The CLI, IDE extension, and desktop app share the configuration. |
| Claude Code | `claude mcp add` or `.mcp.json` | Run `claude mcp add --transport http blindpay https://mcp.blindpay.com/mcp`, or add the server to `.mcp.json`, and load Agent Skills for full scaffolding. |
| Cursor | `.cursor/mcp.json` | Add `{ "url": "https://mcp.blindpay.com/mcp" }` as the `blindpay` server, restart Cursor, then enable it in Settings -> MCP and sign in. |
| Grok | grok.com/connectors or `~/.grok/config.toml` | In Grok, add a custom connector with `https://mcp.blindpay.com/mcp` and sign in with your BlindPay account. In Grok Build, declare the server in `~/.grok/config.toml`. See [Grok + BlindPay](../integrations/grok.md). |
| Windsurf | MCP settings | Add `https://mcp.blindpay.com/mcp` as a remote server in Windsurf's MCP configuration. |
| Lovable | Copy-paste prompt + REST API | No MCP host. Paste a prompt describing the backend function and add `BLINDPAY_API_KEY` / `BLINDPAY_INSTANCE_ID` as secrets. |
| v0 | Copy-paste prompt + REST API | Paste a prompt describing a Next.js route handler; add credentials as Vercel environment variables. |
| Bolt | Copy-paste prompt + REST API | Paste a prompt describing a server route; add credentials as environment variables in the Bolt project. |
| Replit | Copy-paste prompt + REST API | Give the prompt to Replit Agent; add credentials in Replit Secrets (Tools -> Secrets). |

### Codex, Claude Code, Cursor, and Windsurf (MCP)

These hosts call BlindPay tools directly, so the agent can run multi-step flows (quote, confirm, execute) in one conversation. Add the MCP server as shown above, then optionally layer in Agent Skills for richer domain knowledge of corridors and fees.

### Lovable, v0, Bolt, and Replit (prompt + REST API)

These builders generate application code rather than hosting MCP tools, so the integration runs through a backend function calling the REST API. The pattern is the same across all four:

```text [Builder prompt]
Add stablecoin payments to this app using the BlindPay API (https://api.blindpay.com).

- Create a backend route/function that calls BlindPay to:
  1. Create a payout quote (POST /v1/instances/{instance_id}/quotes)
  2. Execute a payout (POST /v1/instances/{instance_id}/payouts/evm)
- Read BLINDPAY_API_KEY and BLINDPAY_INSTANCE_ID from environment variables/secrets.
- Authenticate with: Authorization: Bearer ${BLINDPAY_API_KEY}
- Build a form where a user enters an amount in USDC and a destination
  (bank account / blockchain wallet), shows the live quote, and submits the payout.
- Never expose the API key in client code: all BlindPay calls go through the backend.
```

```ts [Backend route]
const res = await fetch(
  `https://api.blindpay.com/v1/instances/${instanceId}/quotes`,
  {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.BLINDPAY_API_KEY}`,
      'Content-Type': 'application/json',
    },
    // request_amount is in cents: 1000 = 10.00
    body: JSON.stringify({
      bank_account_id: 'ba_000000000000',
      currency_type: 'sender',
      cover_fees: true,
      request_amount: 1000,
      network: 'base',
      token: 'USDC',
    }),
  },
)
const quote = await res.json()
```

**Note:**

Keep the secret API key server-side in all four builders: a backend/edge function, a Next.js route handler, or a server-only environment variable. Never call BlindPay from client-side code.

## Related

- [API reference](https://api.blindpay.com/reference)
- [Fiat overview](overview.md)
- [Stablecoin overview](overview.md)
- [API keys](../essentials/api-keys.md)
- [Webhooks](../essentials/webhooks.md)
