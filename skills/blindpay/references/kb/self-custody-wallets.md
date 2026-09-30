# Self-custody wallets (Brazil)

Brazilian customers must declare whether each external wallet is self-custodied. What changes, who is affected, and how to do it in the dashboard or the API.

Source: https://blindpay.com/docs/kb/self-custody-wallets

## Summary

Brazil's Central Bank Resolution 588 requires BlindPay to report to COAF every transfer of **US$10,000 or more** to or from a **self-custodied wallet**. To comply, starting **October 1, 2026**, every external blockchain wallet added for a **Brazilian customer** must say whether the customer holds the wallet's private keys.

- **New wallets:** the answer is required when the wallet is added.
- **Existing wallets:** answer once for each wallet that has no answer yet.
- **The answer is final.** It can be saved once. If it was saved by mistake, contact support.

## Who is affected

| | Affected |
| --- | --- |
| Customers with `country` = `BR` (individuals and businesses) | Yes |
| Customers in any other country | No. The field is optional for them |
| [Blockchain wallets](../payins/blockchain-wallets.md) (external wallets you add for a customer) | Yes |
| [Managed wallets](../wallets/wallets.md) (wallets BlindPay creates and custodies) | No |

## What is a self-custody wallet

A wallet is **self-custodied** when the customer alone controls its private keys or seed phrase.

| Self-custody (answer **Yes**) | Not self-custody (answer **No**) |
| --- | --- |
| Browser and mobile wallets such as MetaMask, Rabby, Phantom, Trust Wallet | A deposit address at an exchange, such as Binance or Coinbase |
| Hardware wallets such as Ledger or Trezor | A wallet held by a custodian or a regulated custody provider |
| A multisig or smart-account wallet the customer controls | A wallet at another payment provider that holds the keys for the customer |

If you're unsure, ask the customer: "Can a third party move funds out of this wallet without your approval?" If the answer is no, it is self-custodied.

## What you need to do

### In the dashboard

1. Open **Customers**, pick a Brazilian customer, then open **External Wallets**.
2. **New wallet:** the **Add External Wallet** form asks "Is this a self-custody wallet?". Choose **Yes** or **No**, then add the wallet.
3. **Existing wallet:** wallets without an answer show a yellow alert, **Self-custody not set**. Click it, choose **Yes** or **No**, and click **Save**.

Wallets answered **Yes** show a **Self-custody** badge.

### With the API

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

#### Add a wallet

Send `is_self_custody` when you add a wallet for a Brazilian customer. Without it, the request fails with `400 self_custody_required`.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/blockchain-wallets \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "name": "John personal wallet",
    "network": "polygon",
    "is_account_abstraction": true,
    "address": "0x...",
    "is_self_custody": true
  }'
```

```ts [Node SDK]
await blindpay.wallets.blockchain.createWithAddress({
  customer_id: 're_000000000000',
  name: 'John personal wallet',
  network: 'polygon',
  address: '0x...',
  is_self_custody: true,
})
```

The same field works on the signed-message flow (`is_account_abstraction: false`).

#### Find wallets without an answer

List the customer's wallets. Wallets that still need an answer have `is_self_custody: null`.

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/blockchain-wallets \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

```json
[
  {
    "id": "bw_000000000000",
    "name": "John personal wallet",
    "network": "polygon",
    "address": "0x...",
    "is_account_abstraction": true,
    "is_self_custody": null,
    "customer_id": "re_000000000000"
  }
]
```

#### Answer for an existing wallet

```bash [cURL]
curl --request PATCH \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/blockchain-wallets/bw_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{ "is_self_custody": true }'
```

```ts [Node SDK]
await blindpay.wallets.blockchain.setSelfCustody({
  customer_id: 're_000000000000',
  id: 'bw_000000000000',
  is_self_custody: true,
})
```

The response is the updated wallet. A second `PATCH` on the same wallet fails with `409 self_custody_already_set`.

#### Webhook

`blockchainWallet.update` fires when the answer is saved. The payload is the wallet, the same shape as `blockchainWallet.new`. See [Webhooks](../essentials/webhooks.md).

## Errors

| Status | Code | When | What to do |
| --- | --- | --- | --- |
| `400` | `self_custody_required` | You added a wallet for a Brazilian customer without `is_self_custody` | Send `is_self_custody` as `true` or `false` |
| `409` | `self_custody_already_set` | You tried to change an answer that was already saved | Contact support to correct it |
| `404` | `blockchain_wallet_not_found` | The wallet doesn't exist, was removed, or belongs to another customer | Check the wallet and customer IDs |

## FAQ

**Are wallets without an answer blocked?**
No. They keep working and show the alert until you answer. Answer them as soon as you can.

**I saved the wrong answer. Can I change it?**
Not through the dashboard or the API. Contact support with the wallet ID and the correct answer, and we'll fix it.

**Do I need to do anything for customers outside Brazil?**
No. You can send `is_self_custody` for them, but it is optional.

**What does BlindPay do with this information?**
BlindPay uses it to report transfers of US$10,000 or more to or from self-custodied wallets to COAF, as Resolution 588 requires.

## Related

- [Blockchain wallets](../payins/blockchain-wallets.md): add, list, and remove external wallets
- [Webhooks](../essentials/webhooks.md): event delivery and signature verification
