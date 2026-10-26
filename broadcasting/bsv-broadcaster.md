---
description: >-
  The BSV Association's supported transaction broadcaster for Teranode. Public
  endpoints, API reference, SDK integration, and self-hosting.
---

# BSV Broadcaster

[BSV Broadcaster](https://github.com/bsv-blockchain/broadcaster) is the BSV Association's transaction broadcast service. It accepts a signed transaction over HTTP, validates it against network policy, delivers it to Teranode, and then follows it through the network until it is mined, reporting each status change by webhook or server-sent events and handing back the merkle path once the transaction is in a block.

BSV Broadcaster is the successor to [ARC and mAPI](legacy-arc-mapi.md). Its HTTP API keeps ARC's request and response shapes wherever practical, so most ARC clients can be repointed with a URL change. Unlike ARC, it is built for Teranode only and runs as a single binary with no external database.

{% hint style="info" %}
BSV Broadcaster was previously called Arcade; the rename shipped in v0.19.0. The public hostnames and the Python SDK class still use the name `arcade`, and some public endpoints still run pre-rename releases, so their webhooks carry the old `arcade-webhook/` User-Agent. If you run your own instance, the repository's [migration guide](https://github.com/bsv-blockchain/broadcaster/blob/main/docs/migration-arcade-to-bsv-broadcaster.md) lists every renamed setting, path, and image.
{% endhint %}

{% embed url="https://github.com/bsv-blockchain/broadcaster" %}

{% hint style="success" %}
**Two ways to use it.** Point your application at one of the [public endpoints](#public-endpoints) below and start broadcasting immediately, or [run your own instance](run-your-own-bsv-broadcaster.md) for private callbacks, your own limits, and no third-party dependency. The API is identical either way. New here? Start with the [Quickstart](quickstart.md).
{% endhint %}

## Public endpoints

The following instances are open to anyone, on mainnet, with no API key.

| Operator | Region | URL |
| --- | --- | --- |
| BSV Association | United States | `https://arcade-us-1.bsvblockchain.tech` |
| BSV Association | Europe | `https://arcade-eu-1.bsvblockchain.tech` |
| BSV Association | Asia-Pacific | `https://arcade-ap-1.bsvblockchain.tech` |
| GorillaPool | — | `https://arcade.gorillapool.io` |

Each instance publishes its own fee and size limits at `GET /policy`. They differ by operator, so read the policy from the instance you intend to use rather than assuming one value across the network. Every instance also serves interactive API documentation at its root URL and a liveness report at `/health` that lists the Teranode endpoints it is connected to.

{% hint style="info" %}
The public endpoints are shared infrastructure, suitable for development and for production applications with ordinary volume. If you need guaranteed throughput, private callback handling, or simply no dependency on someone else's service, [run your own instance](run-your-own-bsv-broadcaster.md). It takes about ten minutes.
{% endhint %}

## Quick start

Submit a transaction as hex:

```bash
curl -X POST https://arcade-us-1.bsvblockchain.tech/tx \
  -H "Content-Type: text/plain" \
  --data "<hex-encoded transaction>"
```

```json
{ "txid": "9f2b…", "status": 202, "txStatus": "RECEIVED" }
```

Check its status:

```bash
curl https://arcade-us-1.bsvblockchain.tech/tx/<txid>
```

```json
{
  "txid": "9f2b…",
  "txStatus": "MINED",
  "timestamp": "2026-10-16T12:00:00Z",
  "blockHash": "0000…",
  "blockHeight": 971333,
  "merklePath": "<BUMP hex>"
}
```

## API reference

Every path is relative to the instance URL. The full OpenAPI description lives in the repository at [`openapi/bsv-broadcaster.openapi.yaml`](https://github.com/bsv-blockchain/broadcaster/blob/main/openapi/bsv-broadcaster.openapi.yaml).

### `POST /tx` — submit one transaction

Pick the body encoding that suits your client. Bodies are capped at 32 MiB.

| `Content-Type` | Body |
| --- | --- |
| `text/plain` | Hex-encoded transaction. |
| `application/octet-stream` | Raw transaction bytes. Most efficient. |
| `application/json` | `{ "rawTx": "<hex>" }`. The ARC-compatible form. |

Raw and [Extended Format](https://bsv.brc.dev/transactions/0030) are both accepted. BEEF is not; convert it to EF before submitting. BSV Broadcaster runs policy, fee, and script validation synchronously before answering, so EF is preferred because it lets the validator check fees and scripts without looking up the parent outputs.

Optional headers:

| Header | Effect |
| --- | --- |
| `X-CallbackUrl` | BSV Broadcaster POSTs status updates to this URL as the transaction progresses. |
| `X-CallbackToken` | Opaque token of your choosing. Sent back on every webhook and used to filter the SSE stream. |
| `X-FullStatusUpdates` | `true` to receive every intermediate status, not only the final ones. |
| `X-SkipFeeValidation`, `X-SkipScriptValidation` | Bypass the corresponding local check. Honoured only where the operator allows it. |

Responses:

| Status | Meaning |
| --- | --- |
| `202` | Accepted for processing. Body is `{ txid, status: 202, txStatus }`. Resubmitting a transaction BSV Broadcaster has already seen also returns 202, with `txStatus` reflecting its current state, which may already be `MINED`, `REJECTED`, or `DOUBLE_SPEND_ATTEMPTED`. |
| `400` | The transaction failed to decode or failed validation. The body is `{ error, reason, txid, status, title, detail, type }`, where `status` is the ARC error code (461 malformed scripts, 465 fee too low, 476 not final, and so on). This is terminal: the transaction is invalid as submitted, and sending it elsewhere will not help. The one exception is 476, a locktime that has not yet passed, which you may resubmit later. |
| `503` | BSV Broadcaster is under backpressure. Nothing was queued. Retry after a short delay. |

### `POST /txs` — submit a batch

Concatenate raw transaction bytes with no separators and send them as `application/octet-stream`. Bodies are capped at 256 MiB. The response reconciles the batch:

```json
{ "submitted": 42, "duplicates": 3, "total": 45 }
```

If any transaction in the batch fails validation the whole call returns `400` naming the offending txid, and nothing from that call is published.

### `GET /tx/{txid}` — status

Returns the transaction's current state. Once mined, the response includes `blockHash`, `blockHeight`, and `merklePath`, a hex-encoded [BUMP](https://bsv.brc.dev/transactions/0074) you can verify against block headers. Status records are kept indefinitely. A `404` means this instance has never issued a verdict for the txid, so if you are using several instances, query the one you submitted to.

Add `?callbackToken=<token>` to see the delivery state of your webhooks for this transaction.

### `GET /policy` — fee and size limits

Returns the policy the instance enforces at intake, in the same shape ARC used. `GET /v1/policy` is an alias.

```json
{
  "policy": {
    "miningFee": { "satoshis": 1, "bytes": 1000 },
    "maxtxsizepolicy": 100000000,
    "maxscriptsizepolicy": 100000000,
    "maxtxsigopscountspolicy": 4294967295,
    "standardFormatSupported": true
  },
  "timestamp": "2026-10-16T12:00:00Z"
}
```

Unless the operator has pinned a value, `miningFee` tracks the lowest rate currently advertised by Teranode peers, and the size limits track the most generous peer. Read it before building large or low-fee transactions.

### `GET /events` — server-sent events

A long-lived stream of status updates for every transaction submitted with a given `X-CallbackToken`:

```bash
curl -N "https://arcade-us-1.bsvblockchain.tech/events?callbackToken=<token>"
```

```
id: 1716205200123456789
event: status
data: {"txid":"9f2b…","txStatus":"SEEN_ON_NETWORK","timestamp":"2026-10-16T12:00:00.123Z"}

id: 1716205260987654321
event: status
data: {"txid":"9f2b…","txStatus":"MINED","timestamp":"2026-10-16T12:01:00.987Z","blockHash":"0000…","blockHeight":971333,"merklePath":"<BUMP hex>"}
```

A `: keepalive` comment arrives every 15 seconds. If the connection drops, reconnect with the `Last-Event-ID` header set to the last `id` you saw and BSV Broadcaster replays everything you missed. CORS is open, so a browser can subscribe directly with `EventSource`.

### Webhooks

When you submit with `X-CallbackUrl`, BSV Broadcaster POSTs a JSON body to that URL on each status change:

```json
{
  "timestamp": "2026-10-16T12:01:00Z",
  "txid": "9f2b…",
  "txStatus": "MINED",
  "blockHash": "0000…",
  "blockHeight": 971333,
  "merklePath": "<BUMP hex>"
}
```

Requests carry a `User-Agent` beginning `bsv-broadcaster-webhook/` and, if you supplied one, your callback token. Releases before v0.19.0 send `arcade-webhook/` instead, and some public endpoints still run them, so if you allowlist callbacks by User-Agent, allow both prefixes. Failed deliveries are retried with linear backoff (1, 2, 3 minutes and so on) up to the operator's configured limit. Callback URLs must be publicly routable; private and loopback addresses are refused at submit time.

### Transaction statuses

| Status | Meaning |
| --- | --- |
| `RECEIVED` | Validated locally and queued. |
| `SENT_TO_NETWORK` | Delivered to at least one Teranode. |
| `ACCEPTED_BY_NETWORK` | A Teranode accepted it into its mempool. |
| `SEEN_ON_NETWORK` | Observed in a miner's subtree. Safe to treat as zero-confirmation. |
| `SEEN_MULTIPLE_NODES` | Observed by more than one miner. Informational. |
| `MINED` | Included in a block. `merklePath` is now available. |
| `IMMUTABLE` | Deep enough that reorganisation is no longer a practical concern. |
| `PENDING_RETRY` | Delivery to Teranode failed; BSV Broadcaster will retry. |
| `STUMP_PROCESSING` | Mined; merkle path is being assembled. |
| `REJECTED` | Rejected by the network. Terminal. |
| `DOUBLE_SPEND_ATTEMPTED` | A conflicting transaction was seen. Terminal in BSV Broadcaster, unlike ARC where it could still resolve. `competingTxs` lists the conflicts. |
| `UNKNOWN` | No verdict yet. |

If a block is orphaned after a `MINED` report, BSV Broadcaster re-anchors the transaction to the replacement block and emits a new `MINED` event with a fresh merkle path. The superseded anchors are kept under `orphanedProofs` on the status record.

## Using BSV Broadcaster from the SDKs

### TypeScript

The `Teranode` broadcaster in `@bsv/sdk` posts a transaction as Extended Format bytes to whatever URL you give it, which is exactly what BSV Broadcaster's `/tx` endpoint accepts. Every input must have its `sourceTransaction` set so the SDK can produce EF.

```ts
import { Transaction, Teranode } from '@bsv/sdk'

const broadcaster = new Teranode('https://arcade-us-1.bsvblockchain.tech/tx')

const tx = Transaction.fromHex(signedTxHex)
const result = await tx.broadcast(broadcaster)
// { status: 'success', txid: '…', message: 'broadcast successful' }
```

If you need callbacks or a callback token, implement the `Broadcaster` interface directly. The [custom broadcaster example](../guides/sdks/ts/examples/example_building_custom_tx_broadcaster.md) shows the pattern; swap the WhatsOnChain URL for BSV Broadcaster's `/tx`, send `{ rawTx: tx.toHexEF() }` as JSON, and add the `X-CallbackUrl` and `X-CallbackToken` headers.

{% hint style="warning" %}
The SDK's `ARC` class posts to `{url}/v1/tx`, a path BSV Broadcaster does not serve. Do not point `ARC` at a BSV Broadcaster instance. The default broadcaster used by `tx.broadcast()` with no argument is still GorillaPool's ARC; pass an explicit broadcaster to use BSV Broadcaster.
{% endhint %}

### Go

The Go SDK's `Arc` broadcaster posts `{ "rawTx": "<EF or raw hex>" }` to `{ApiUrl}/tx` and accepts any 2xx response, so it works against BSV Broadcaster unchanged. Give it the instance root as `ApiUrl`.

```go
import (
    "github.com/bsv-blockchain/go-sdk/transaction"
    "github.com/bsv-blockchain/go-sdk/transaction/broadcaster"
)

tx, _ := transaction.NewTransactionFromHex(signedTxHex)

callbackURL := "https://your.app/broadcaster-webhook"
success, failure := tx.Broadcast(&broadcaster.Arc{
    ApiUrl:      "https://arcade-us-1.bsvblockchain.tech",
    CallbackUrl: &callbackURL,
})
if failure != nil {
    // failure.Code, failure.Description
}
// success.Txid
```

For ARC itself the same struct expects the versioned base, for example `https://arc.taal.com/v1`. The `WaitFor` and `WaitForStatus` fields are ARC-only and are ignored by BSV Broadcaster.

### Python

The Python SDK ships a dedicated `Arcade` class.

```python
from bsv.broadcasters import Arcade, ArcadeConfig

broadcaster = Arcade(
    "https://arcade-us-1.bsvblockchain.tech",
    ArcadeConfig(
        callback_url="https://your.app/broadcaster-webhook",
        callback_token="a-token-you-choose",
    ),
)

result = await tx.broadcast(broadcaster)        # async
result = broadcaster.sync_broadcast(tx)         # sync

status = broadcaster.check_transaction_status(txid)
```

It submits EF when every input has its `source_transaction`, falls back to raw hex otherwise, treats a `400` as terminal, and reports `REJECTED` or `DOUBLE_SPEND_ATTEMPTED` on a 202 resubmit as a `BroadcastFailure`. The SDK's own [`Arcade` class guide](https://github.com/bsv-blockchain/py-sdk/blob/master/docs/arcade_broadcasting.md) lists the remaining differences from its `ARC` class.

## Self-hosting

Anyone can run BSV Broadcaster. A single binary with an embedded store is a complete instance, and the BSV Association hosts the Merkle Service that drives status tracking for any instance on mainnet, testnet, or teratestnet. See [Run Your Own BSV Broadcaster](run-your-own-bsv-broadcaster.md) for a ten-minute walkthrough covering source and container installs, configuration for each network, enabling status tracking, and production deployment.
