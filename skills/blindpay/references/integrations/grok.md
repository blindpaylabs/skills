# Grok + BlindPay

Connect Grok and Grok Build to BlindPay with the MCP server: run payouts, quotes, and virtual accounts in natural language from grok.com or the terminal.

Source: https://blindpay.com/docs/integrations/grok

[Grok](https://grok.com) is xAI's assistant, and [Grok Build](https://docs.x.ai/build/features/mcp-servers) is its coding agent for the terminal. Connect either one to BlindPay with the **MCP server**. In Grok you add the remote server at `https://mcp.blindpay.com/mcp` and sign in with your BlindPay account over OAuth, so no API key goes into the chat.

## Add the connector in Grok

### Open connectors

Go to [grok.com/connectors](https://grok.com/connectors).

### Create a custom connector

Choose **New connector**, then **Custom**, and paste `https://mcp.blindpay.com/mcp`.

### Sign in

Sign in with your BlindPay account and pick the instance Grok may use.

### Turn it on

Turn the connector on in the chat before you ask.

On Grok Business and Enterprise, an admin enables the connector first.

## Add the MCP server in Grok Build

Declare the server in `~/.grok/config.toml`:

```toml [~/.grok/config.toml]
[mcp_servers.blindpay]
command = "npx"
args = ["-y", "@blindpay/mcp"]
env = { BLINDPAY_API_KEY = "your-api-key", BLINDPAY_INSTANCE_ID = "your-instance-id" }
```

Copy your API key and instance ID from the [BlindPay dashboard](https://app.blindpay.com/sign-up/), then run `grok mcp list` to verify that the `blindpay` server is configured.

## Use it

Ask Grok or Grok Build:

```text [Grok prompt]
Use the BlindPay MCP tools to create a payout quote for 1000 USDC to a USD bank
account, show the fees and FX rate, then execute the payout after I confirm.
```

For a Grok connector that should never move money, use `https://mcp.blindpay.com/mcp/readonly` instead. It exposes only reads.

**Note:**

The MCP server does not ask for confirmation on its own. Keep money movement gated on an explicit yes in your prompt, and connect a development instance while you test.

## Next steps

- [BlindPay for AI agents](../getting-started/build-with-ai.md)
- [All AI builder integrations](https://blindpay.com/docs/integrations)
- [API reference](https://api.blindpay.com/reference)
