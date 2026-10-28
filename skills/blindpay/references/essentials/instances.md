# Instances

Instances are isolated BlindPay environments, one per stage of your stack, created in the dashboard.

Source: https://blindpay.com/docs/learn/instances

An instance is an isolated BlindPay environment. If your product has separate development, staging, and production environments, create one instance for each. All features live inside an instance: customers, bank accounts, wallets, virtual accounts, payouts, and payins.

Instances are created through the [BlindPay dashboard](https://app.blindpay.com). You cannot create them via the API.

Each instance is either a `development` or `production` instance. Both expose the same features but differ in KYC handling, network availability, and whether fiat actually moves.

**Note:**

All payouts and payins made on development instances skip the real fiat payment rails. Payouts do not move real money, and payins auto-complete a simulated deposit instead of waiting for a real bank transfer.

## Development vs. production

| Feature | Development | Production |
| --- | --- | --- |
| Customers | Supported | Supported |
| Bank accounts | Supported | Supported |
| Payout quotes | Supported | Supported |
| Payouts | Supported (no real fiat movement) | Supported |
| Payin quotes | Supported | Supported |
| Payins | Supported (auto-completes in ~30s) | Supported |
| KYC | Auto-approved | Automated (~60s) or manual review |
| EVM networks | `sepolia`, `base_sepolia`, `arbitrum_sepolia`, `polygon_amoy` | `ethereum`, `base`, `polygon`, `arbitrum` |
| Stellar | Testnet | Mainnet |
| Solana | Devnet | Mainnet |
| Tron | N/A | Mainnet (beta) |
| API keys | Supported | Supported |
| Webhooks | Supported | Supported |

On development instances, use USDB as the test stablecoin: a BlindPay-issued test token that stands in for real USDC or USDT. See [Sandbox vs. production](sandbox-vs-production.md) for the full testing workflow.

## Create an instance

### Open the dashboard

Go to the [BlindPay dashboard](https://app.blindpay.com) and click **Create instance**.

### Choose an environment

Choose whether the new instance is `development` or `production`. Development instances are ready immediately. You can own up to 10 development instances (deleting one doesn't free up a slot); production instances have no fixed cap here.

When creating a development instance, you can turn on **Seed sample data** to populate it with sample customers, bank accounts, blockchain wallets, virtual accounts, and a handful of quotes, payouts, and payins, so the dashboard isn't empty while you explore.

![Create instance screen in the BlindPay dashboard](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/Frame%201216401961.jpg)

### Wait for activation (production only)

New production instances may take up to 3 business days to set up. See [Cut-off times](../kb/cut-off-times.md) for full SLAs.

## Sub-instances for your customers' businesses

If your customers are businesses that move money for their own clients, onboard each one as its own instance instead of as a customer of yours. This keeps every party visible and screened. See [Nested payments](../kb/nested-payments.md) for why.

There is no separate sub-instance type. A sub-instance is a regular production instance that you create and that the business completes onboarding for.

### Create the instance

In the [BlindPay dashboard](https://app.blindpay.com), open the instance switcher and click **Create instance**. Create one production instance per business.

### Send the onboarding link

While the new instance is still in onboarding, click **Invite team** in the side menu. Under **Invite someone outside BlindPay**, enter the email of a person at the business and click **Send invite**. They get a private link to fill in the business details and upload the KYB documents. They don't need a BlindPay account. The link expires in 30 days, and you can also copy it from the dialog.

### Compliance review

Once the business submits, our compliance team reviews the instance like any other production instance. After it is approved, that business's customers are onboarded inside the instance, which has its own API keys.

## API keys are per-instance

An API key authenticates requests to one instance only. A key created for instance A will not work against instance B, even within the same organization. See [API keys](api-keys.md) to create and manage keys.

## Manage your instance

### Update settings

```bash [cURL]
curl --request PUT \
  --url https://api.blindpay.com/v1/instances/in_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "name": "My instance",
    "email_notifications": true,
    "require_passkey": false
  }'
```

Requires the owner or admin role on the instance.

**Warning:**

`name` is required on every call, even when you are only changing one other field. Send the instance's current name along with whatever you are updating, or the request fails with `400`.

| Field | Type | Notes |
| --- | --- | --- |
| `name` | string | Required on every call. |
| `email_notifications` | boolean | Defaults to `true`. |
| `require_passkey` | boolean | Defaults to `false`. |
| `customer_rfi_emails_enabled` | boolean, optional | Omit it to leave the current value unchanged; there is no default on update. |
| `compliance_emails` | string[], optional, max 20 | When set, [instance RFI](../kb/instance-requests.md) and customer [RFI](rfi.md) emails go only to these addresses instead of every team member. Send an empty array to go back to notifying the whole team. |
| `customer_invite_redirect_url` | string, optional | Where a customer lands after finishing a hosted onboarding link. |

### Delete an instance

Only the instance's current owner can delete it (`400 user_not_owner` for anyone else). Deleting is a soft delete: the instance stops resolving immediately, but nothing is destroyed on BlindPay's side.

```bash [cURL]
curl --request DELETE \
  --url https://api.blindpay.com/v1/instances/in_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

## Team members

### List members

```bash [cURL]
curl --url https://api.blindpay.com/v1/instances/in_000000000000/members \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Returns every member of the instance with their `role`.

### Roles

| Role |
| --- |
| `owner` |
| `admin` |
| `finance` |
| `checker` |
| `operations` |
| `developer` |
| `viewer` |

`owner` is never assigned directly: `GET /instances/{id}/members` always reports it for whoever currently owns the instance, overriding whatever role is stored for that member. Use ownership transfer, below, to change who holds it.

### Update a member's role

```bash [cURL]
curl --request PUT \
  --url https://api.blindpay.com/v1/instances/in_000000000000/members/us_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "user_role": "developer"
  }'
```

Requires the owner, admin, or checker role on the instance. You cannot change the current owner's role this way: that request is rejected with `400 cannot_update_instance_owner`. Transfer ownership first if you need to change what the owner can do.

### Remove a member

```bash [cURL]
curl --request DELETE \
  --url https://api.blindpay.com/v1/instances/in_000000000000/members/us_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Requires the owner or admin role on the instance. The current owner cannot be removed this way (`400 cannot_delete_instance_owner`); transfer ownership first.

### Transfer ownership

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/ownership \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "user_id": "us_000000000000"
  }'
```

Only the current owner can transfer ownership, and only to someone who is already a member of the instance (`400 new_owner_not_instance_member` otherwise; `400 already_instance_owner` if you pass yourself). Both accounts end up with the `admin` role in the underlying membership record: `owner` itself is derived from who currently holds ownership, not from a stored role, so the previous owner keeps full admin access after the transfer.

**Note:**

You can pass an `Idempotency-Key` header (up to 255 characters) on any update, delete, or ownership-transfer call on this page. Retrying with the same key and the same body replays the original response instead of repeating the action; the same key with a different body is rejected. Keys are remembered for 24 hours.

## Related

- [Sandbox vs. production](sandbox-vs-production.md): behavior differences in depth
- [API keys](api-keys.md): create and manage instance keys
- [Billing](billing.md): how usage is charged per instance
