# API keys

Authenticate all BlindPay API requests with an instance-scoped API key created in the dashboard.

Source: https://blindpay.com/docs/learn/api-keys

An API key authenticates your requests to the BlindPay API. Each key is scoped to a single instance: a key created for one instance will not work against another, and a key created on a development instance will not work against a production instance.

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)

## Create an API key

Go to the [BlindPay dashboard](https://app.blindpay.com), select an instance, and open the **API Keys** tab.

![BlindPay dashboard API Keys tab](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/api_keys.jpg)

**Warning:**

Keys are shown once at creation. Copy and store them securely: you cannot retrieve the key value again after leaving the creation screen. The dashboard only ever displays a short prefix of the key afterward, for identification purposes.

Creating a key requires a dashboard (user) session; it cannot be done from an existing API key. This prevents a compromised key from minting new keys for itself.

**Note:**

If your instance has **Require passkey for critical actions** turned on (**Settings → Instance**), the dashboard asks you to verify with a registered passkey before it creates the key.

## Authenticate requests

Pass the key as a bearer token in every request:

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/customers \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

**Success:**

Done. Authenticate all requests by passing the header `Authorization: Bearer YOUR_API_KEY`.

## Security best practices

- Never expose API keys in client-side code, mobile apps, or version control.
- Use environment variables to inject keys at runtime instead of hardcoding them.
- Rotate keys immediately if you suspect a key has been compromised: delete the old key and create a new one in the dashboard. There is no separate rotate endpoint; delete and recreate is the only path, and deleting a key revokes it immediately.
- Use separate keys per environment: one for development, one for production. Keys created on a development instance will not work against a production instance, and vice versa.

## IP whitelisting

Restrict a key to a fixed set of source IPs by setting `ip_whitelist` when you create it via the API. Only IPv4 addresses are accepted; an IPv6 address is rejected with `400 VALIDATION_FAILED`.

Leave `ip_whitelist` `null` or omit it (the default) to allow requests from any IP. When it's set, it is enforced on every request made with the key, not just at creation: a request from an IP outside the list fails with `401 AUTH_IP_BLOCKED` before it reaches your resource.

## Key types

Every key has a `permission`, chosen when you create it:

| Permission | Access |
| --- | --- |
| `full_access` | Read and write access to the instance. |
| `read_only` | Read access to every resource in the instance. It cannot create, update or delete anything. |

A key's permission maps to a role in the same role-based access control the dashboard uses for team members, and every endpoint checks it. A `read_only` key can list and fetch resources, request FX rates and export payins and payouts, but any request that changes state (creating quotes, payouts or customers, uploading files, editing webhooks) fails with `403 AUTH_FORBIDDEN`. It also cannot sign blockchain wallet messages, read a webhook endpoint's signing secret, or open the webhook portal, since those produce signatures or return credentials. Use read-only keys for reporting, reconciliation and dashboards that never move money.

## Related

- [Instances](instances.md): each API key is scoped to one instance
- [Sandbox vs. production](sandbox-vs-production.md): use separate keys per environment
- [Webhooks](webhooks.md): a complementary way to receive updates without polling the API
- [Quickstart](../getting-started/quickstart-payin.md): see an API key used in your first request
