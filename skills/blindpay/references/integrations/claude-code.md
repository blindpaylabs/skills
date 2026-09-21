# Claude Code + BlindPay

Connect Claude Code to BlindPay via the MCP server and Agent Skills: run payouts, quotes, and virtual accounts from the terminal in natural language.

Source: https://blindpay.com/docs/integrations/claude-code

[Claude Code](https://claude.com/claude-code) is Anthropic's terminal coding agent. Connect it to BlindPay with the **MCP server** and **Agent Skills** for agentic stablecoin payments from the command line.

## Add the MCP server

```bash [Terminal]
claude mcp add blindpay \
  --env BLINDPAY_API_KEY=your-api-key \
  --env BLINDPAY_INSTANCE_ID=your-instance-id \
  -- npx -y @blindpay/mcp
```

Or add it manually to your project's `.mcp.json`:

```json [.mcp.json]
{
  "mcpServers": {
    "blindpay": {
      "command": "npx",
      "args": ["-y", "@blindpay/mcp"],
      "env": {
        "BLINDPAY_API_KEY": "your-api-key",
        "BLINDPAY_INSTANCE_ID": "your-instance-id"
      }
    }
  }
}
```

## Add Agent Skills

```bash [Terminal]
npx skills add blindpaylabs/skills
```

## Use it

### Get your credentials

Copy your API key and instance ID from the [BlindPay dashboard](https://app.blindpay.com/sign-up/).

### Ask Claude Code

```text [Claude Code prompt]
Use the BlindPay MCP tools to create a payout quote for 1000 USDC to a USD bank
account, show the fees and FX rate, then execute the payout after I confirm.
```

### Build integrations

With Skills loaded, Claude Code can scaffold full BlindPay integrations in your codebase, not just one-off calls.

**Note:**

The MCP server reads your key from the env you pass to `claude mcp add`. Money-movement tools should require explicit confirmation before executing.

## Next steps

- [BlindPay for AI agents](../getting-started/build-with-ai.md)
- [All AI builder integrations](https://blindpay.com/docs/integrations)
- [API reference](https://api.blindpay.com/reference)
