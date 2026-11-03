# SDKs

Official BlindPay SDKs for Node.js, Python, Go, PHP, and Swift, plus the OpenAPI spec and REST API reference.

Source: https://blindpay.com/docs/sdks

BlindPay ships official SDKs for Node.js, Python, Go, PHP, and Swift. Every SDK wraps the same REST API, so for any other language you can generate a typed client from the OpenAPI spec or call the REST API directly.

## Official SDKs

| SDK | Install | Repository |
| --- | --- | --- |
| Node.js / TypeScript | `npm install @blindpay/node` | [blindpay-node](https://github.com/blindpaylabs/blindpay-node) |
| Python | `pip install blindpay` | [blindpay-python](https://github.com/blindpaylabs/blindpay-python) |
| Go | `go get github.com/blindpaylabs/blindpay-go` | [blindpay-go](https://github.com/blindpaylabs/blindpay-go) |
| PHP | `composer require blindpay/php` | [blindpay-php](https://github.com/blindpaylabs/blindpay-php) |
| Swift | Swift Package Manager: `https://github.com/blindpaylabs/blindpay-swift.git` | [blindpay-swift](https://github.com/blindpaylabs/blindpay-swift) |

Each repository's README has the full install and usage guide.

A small number of endpoints are excluded from every generated SDK: `POST /upload/extract` (see [Invoice payables](../payables/payable-invoice.md#read-the-invoice-with-ai)), `POST /instances/{instance_id}/tos/passthrough` (see [Terms of service](../essentials/terms-of-service.md)), the [yield](https://blindpay.com/docs/yield) endpoints under `/wallets/{id}/yield`, the card payment endpoints (`POST /instances/{instance_id}/payins/{payin_id}/card-payment` and `POST /e/payins/{payin_id}/card-payment`), and the development-only virtual account deposit simulators (`simulate-spei-deposit`, `simulate-breb-deposit`). Call them over raw HTTP even if you use an SDK for everything else.

## OpenAPI spec

Download the OpenAPI 3.1 spec to generate a typed client in any language:

```bash
curl https://api.blindpay.com/doc -o blindpay-openapi.json
```

Use [openapi-typescript](https://github.com/openapi-ts/openapi-typescript) for TypeScript:

```bash
npx openapi-typescript https://api.blindpay.com/doc -o src/blindpay.d.ts
```

## REST API

The base URL for all API requests is:

```
https://api.blindpay.com/v1
```

Authenticate with a Bearer token in every request:

```bash
curl https://api.blindpay.com/v1/instances/in_000000000000/customers \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

The full API reference, including all endpoints, request bodies, and response schemas, is at:

[api.blindpay.com/reference](https://api.blindpay.com/reference)
