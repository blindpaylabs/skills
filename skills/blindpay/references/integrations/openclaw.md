# OpenClaw + BlindPay

Connect OpenClaw to BlindPay with the MCP server and Agent Skills. The autonomous agent runs stablecoin payouts and quotes via natural language.

Source: https://blindpay.com/docs/integrations/openclaw

[OpenClaw](https://openclaw.ai) is an open-source autonomous AI agent that runs locally and makes sequential tool calls to get work done. Connect it to BlindPay with the **MCP server** and **Agent Skills** for agentic stablecoin payments.

## Add the MCP server

Install BlindPay's MCP server through OpenClaw's MCP registry:

```bash [Terminal]
openclaw mcp add blindpay \
  --command npx \
  --arg -y \
  --arg @blindpay/mcp \
  --env BLINDPAY_API_KEY=your-api-key \
  --env BLINDPAY_INSTANCE_ID=your-instance-id
```

Or add it directly under `mcp.servers` in your OpenClaw config:

```json [OpenClaw config]
{
  "mcp": {
    "servers": {
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
}
```

Verify the server is reachable:

```bash [Terminal]
openclaw mcp doctor blindpay --probe
```

## Add Agent Skills

```bash [Terminal]
npx skills add blindpaylabs/skills
```

## Use it

### Get your credentials

Copy your API key and instance ID from the [BlindPay dashboard](https://app.blindpay.com/sign-up/).

### Add the MCP server

Run the `openclaw mcp add` command above (or edit the config), then reload.

### Ask the agent

Tell OpenClaw:

```text [OpenClaw prompt]
Use the BlindPay MCP tools to create a payout quote for 1000 USDC to a USD bank
account, show the fees and FX rate, then execute the payout after I confirm.
```

**Note:**

OpenClaw runs autonomously and chains tool calls. Keep money-movement gated on explicit confirmation, and scope the API key to what the agent should do.

## Next steps

- [BlindPay for AI agents](../getting-started/build-with-ai.md)
- [All AI builder integrations](https://blindpay.com/docs/integrations)
- [API reference](https://api.blindpay.com/reference)
