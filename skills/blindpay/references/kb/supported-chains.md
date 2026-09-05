# Supported chains

Reference for the blockchains, stablecoins, and per-feature chain support across BlindPay payins, payouts, wallets, and transfers.

Source: https://blindpay.com/docs/kb/supported-chains

BlindPay settles stablecoins on Ethereum, Polygon, Base, Arbitrum, Tempo (EVM), Stellar, Solana, and Tron. Production instances run on mainnets with USDC or USDT; development instances run on the matching testnets with USDB, BlindPay's test stablecoin.

There is no runtime endpoint that lists supported chains or tokens. This page is the reference: the enums below are fixed by the API and are the source of truth for which `network` and `token` values you can send.

## Chains

| Chain | Mainnet chain ID | Testnet (development) | Testnet chain ID | Mainnet stablecoins |
| --- | --- | --- | --- | --- |
| Ethereum | 1 | Ethereum Sepolia | 11155111 | USDC, USDT |
| Polygon | 137 | Polygon Amoy | 80002 | USDC, USDT |
| Base | 8453 | Base Sepolia | 84532 | USDC |
| Arbitrum | 42161 | Arbitrum Sepolia | 421614 | USDC |
| Tempo | 4217 | Tempo Testnet | 42431 | USDC, USDT |
| Stellar | n/a | Stellar Testnet | n/a | USDC |
| Solana | n/a | Solana Devnet | n/a | USDC, USDT |
| Tron | n/a | none | n/a | USDT |

Stellar, Solana, and Tron do not have a real numeric chain ID; the API accepts the network name directly (`stellar`, `solana`, `tron`), not a chain ID. Tron has no testnet, so it cannot be exercised on a development instance. Tempo is an EVM chain like Ethereum, Polygon, Base, and Arbitrum: it uses the same `0x` + 40 hex character address format.

USDB, BlindPay's test stablecoin, is available only on development instances and only on the seven testnets above (`sepolia`, `base_sepolia`, `arbitrum_sepolia`, `polygon_amoy`, `stellar_testnet`, `solana_devnet`, `tempo_testnet`). It has no mainnet deployment on any chain. Wallets receive it through [payins](../payins/payins.md), which auto-complete on development instances.

A production instance can only use `USDC` or `USDT` on a mainnet network. A development instance can only use `USDB` on a testnet network. Sending a mismatched combination (for example `USDC` on `sepolia`, or `USDB` on `polygon`) is rejected. The one exception is a cross-chain USDC transfer (see below): on a development instance it uses real testnet USDC instead of USDB, since [Circle CCTP v2](../wallets/transfer-quotes.md#cross-chain-usdc-transfers-circle-cctp-v2) can only move native USDC.

## Support by feature

"Payins" are the stablecoin delivery side of a bank deposit (on-ramp); "payouts" are the stablecoin source side of a bank transfer out (off-ramp). See [Bank transfer in](../payins/payins.md) and [Pay out to bank](../payouts/payouts.md) for the fiat-side flow, or [Payins](../payins/payins.md) and [Payouts](../payouts/payouts.md) for the full crypto mechanics.

| Feature | Chains | Stablecoins | Notes |
| --- | --- | --- | --- |
| Payins (on-ramp delivery) | Ethereum, Polygon, Base, Arbitrum, Tempo, Stellar, Solana, Tron | USDC, USDT (network-dependent; USDC not on Tron) | |
| Payouts (off-ramp source) | Ethereum, Polygon, Base, Arbitrum, Tempo, Stellar, Solana, Tron | USDC, USDT (network-dependent; USDC not on Tron; USDT not on Base/Arbitrum/Stellar) | |
| Managed wallets (beta) | Ethereum, Polygon, Base, Arbitrum, Tempo, Solana | USDC, USDT (USDB on testnet) | |
| Yield on managed wallets | Base, Polygon | USDC | See [Yield](https://blindpay.com/docs/yield) |
| External blockchain wallets | Ethereum, Polygon, Base, Arbitrum, Tempo, Stellar, Solana, Tron | USDC, USDT (network-dependent; USDC not on Tron; USDT not on Base/Arbitrum/Stellar) | Store an address you already control; see [Managed wallet](../wallets/store.md) |
| Transfers | Same network, except USDC which can also cross Ethereum, Polygon, Base, Arbitrum via Circle CCTP v2 | USDC, USDT (USDT restricted to Polygon) | Tempo supports same-network transfers only; it is not part of the CCTP v2 route set. No token conversion. |

**Note:**

Managed wallets are in beta. Production access is granted on request.

Not every token is deployed on every chain. In particular:

- USDC is not deployed on Tron.
- USDT is only deployed on Polygon, Ethereum, Tron, Solana, and Tempo, not on Base, Arbitrum, or Stellar.
- USDB only exists on the seven development testnets, never on a mainnet.

The quote-creation endpoints enforce these combinations and return a descriptive error (for example, a token not supported on the requested chain) if you request an unsupported pairing.

## Related

- [Bank transfer in](../payins/payins.md): payin quotes and payment methods on the fiat side
- [Pay out to bank](../payouts/payouts.md): payout quotes and bank account types on the fiat side
- [Payins](../payins/payins.md): payin delivery networks and corridors
- [Payouts](../payouts/payouts.md): payout authorization per network, including Stellar and Solana
- [Managed wallet](../wallets/store.md): wallet chain support and balance operations
