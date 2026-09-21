# Hermes Agent + BlindPay

Connect Hermes Agent to BlindPay with the MCP server and Agent Skills. The Nous Research agent runs stablecoin payouts and quotes via natural language.

Source: https://blindpay.com/docs/integrations/hermes

[Hermes Agent](https://hermes-agent.nousresearch.com) by Nous Research is a self-hosted AI agent with native MCP support. Connect it to BlindPay with the **MCP server** and **Agent Skills** for agentic stablecoin payments.

## Add the MCP server

Add BlindPay under `mcp_servers` in `~/.hermes/config.yaml`:

```yaml [~/.hermes/config.yaml]
mcp_servers:
  blindpay:
    command: "npx"
    args: ["-y", "@blindpay/mcp"]
    env:
      BLINDPAY_API_KEY: "your-api-key"
      BLINDPAY_INSTANCE_ID: "your-instance-id"
```

Then reload MCP from within a Hermes session:

```text [Hermes session]
/reload-mcp
```

## Add Agent Skills

```bash [Terminal]
npx skills add blindpaylabs/skills
```

## Use it

### Get your credentials

Copy your API key and instance ID from the [BlindPay dashboard](https://app.blindpay.com/sign-up/).

### Add the MCP server

Add the `blindpay` block to `~/.hermes/config.yaml`, then run `/reload-mcp`.

### Ask the agent

Tell Hermes:

```text [Hermes prompt]
Use the BlindPay MCP tools to create a payout quote for 1000 USDC to a USD bank
account, show the fees and FX rate, then execute the payout after I confirm.
```

**Note:**

Hermes writes reusable skills as it works and runs continuously. Keep money-movement gated on explicit confirmation, and scope the API key to what the agent should do.

## Next steps

- [BlindPay for AI agents](../getting-started/build-with-ai.md)
- [All AI builder integrations](https://blindpay.com/docs/integrations)
- [API reference](https://api.blindpay.com/reference)
