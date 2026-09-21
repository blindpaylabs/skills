# Cursor + BlindPay

Connect Cursor to BlindPay via the MCP server and Agent Skills: move money, run payouts, and query corridors in natural language.

Source: https://blindpay.com/docs/integrations/cursor

[Cursor](https://cursor.com) is an AI code editor. Connect it to BlindPay with the **MCP server** (live tools) and **Agent Skills** (domain knowledge) for agentic stablecoin payments inside your editor.

## Add the MCP server

Create `.cursor/mcp.json` in your project (or add to the global config):

```json [.cursor/mcp.json]
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

Restart Cursor, then enable the `blindpay` server in **Settings → MCP**.

## Add Agent Skills

```bash [Terminal]
npx skills add blindpaylabs/skills
```

Skills teach the agent BlindPay's rails, corridors, fees, KYC/KYB flows, and API patterns so it writes correct integrations.

## Use it

### Get your credentials

Copy your API key and instance ID from the [BlindPay dashboard](https://app.blindpay.com/sign-up/).

### Ask the agent

In Cursor chat, try:

```text [Cursor prompt]
Using the BlindPay MCP tools, create a payout quote for 1000 USDC to a USD
bank account, then show me the fees and FX rate before executing.
```

### Build the integration

Ask Cursor to scaffold the integration in your app: it has both live tools and Skills context.

**Note:**

The MCP server runs locally and reads your key from the config env. Keep `.cursor/mcp.json` out of version control if it contains live credentials.

## Next steps

- [BlindPay for AI agents](../getting-started/build-with-ai.md)
- [All AI builder integrations](https://blindpay.com/docs/integrations)
- [API reference](https://api.blindpay.com/reference)
