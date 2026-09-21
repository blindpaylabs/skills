# Codex + BlindPay

Connect OpenAI Codex to BlindPay via the MCP server and Agent Skills: run payouts, quotes, and virtual accounts from the terminal or IDE.

Source: https://blindpay.com/docs/integrations/codex

[Codex](https://learn.chatgpt.com/docs/codex/cli) is OpenAI's coding agent for the terminal, IDE, and desktop app. Connect it to BlindPay with the **MCP server** and **Agent Skills** for agentic stablecoin payments while you build.

## Add the MCP server

```bash [Terminal]
codex mcp add blindpay \
  --env BLINDPAY_API_KEY=your-api-key \
  --env BLINDPAY_INSTANCE_ID=your-instance-id \
  -- npx -y @blindpay/mcp
```

Run `codex mcp list` to verify that the `blindpay` server is configured. Codex CLI, the IDE extension, and the desktop app share this MCP configuration.

## Add Agent Skills

```bash [Terminal]
npx skills add blindpaylabs/skills
```

Skills teach Codex BlindPay's rails, corridors, fees, KYC/KYB flows, and API patterns so it can build correct integrations in your codebase.

## Use it

### Get your credentials

Copy your API key and instance ID from the [BlindPay dashboard](https://app.blindpay.com/sign-up/).

### Ask Codex

```text [Codex prompt]
Use the BlindPay MCP tools to create a payout quote for 1000 USDC to a USD bank
account, show the fees and FX rate, then execute the payout after I confirm.
```

### Build integrations

With Skills loaded, Codex can scaffold full BlindPay integrations in your codebase, not just make one-off tool calls.

**Note:**

Codex stores MCP settings in `~/.codex/config.toml`. Keep that file private, and require explicit confirmation before executing any money movement.

## Next steps

- [BlindPay for AI agents](../getting-started/build-with-ai.md)
- [All AI builder integrations](https://blindpay.com/docs/integrations)
- [API reference](https://api.blindpay.com/reference)
