---
description: >-
  Explorer REST endpoints that accept a raw transaction: WhatsOnChain, Bitails,
  and BananaBlocks. Endpoints, request shapes, limits, and examples.
---

# API Broadcasters

Several BSV block explorers expose an HTTP endpoint that accepts a raw transaction and relays it to the network. They are simple to call, need no account to get started, and are often already part of an application's stack for UTXO and history lookups.

They are not transaction processors. None of them track a transaction's lifecycle for you, deliver merkle paths, or notify you when it is mined; you poll their transaction endpoints for that. Their free tiers are rate-limited at levels suited to prototypes and low-volume applications. For anything beyond that, use [BSV Broadcaster](bsv-broadcaster.md).

| Service | Mainnet base URL | Testnet base URL | Free limit | API key |
| --- | --- | --- | --- | --- |
| [WhatsOnChain](#whatsonchain) | `https://api.whatsonchain.com/v1/bsv/main` | `https://api.whatsonchain.com/v1/bsv/test` | 3 requests/s | `Authorization: <key>` |
| [Bitails](#bitails) | `https://api.bitails.io` | `https://test-api.bitails.io` | 10 requests/s, 1,000/day | `apikey: <key>` |
| [BananaBlocks](#bananablocks) | `https://bananablocks.com/api/v1` | `https://bananablocks.com/api/v1/bsv/test` (WoC-compatible mirror) | 10 broadcasts/min | `Authorization: Bearer <key>` or `X-API-Key: <key>` |

All three take the plain raw serialisation as hex. Convert Extended Format or BEEF to raw before sending.

## WhatsOnChain

Full documentation: [docs.whatsonchain.com](https://docs.whatsonchain.com/)

```bash
curl -X POST https://api.whatsonchain.com/v1/bsv/main/tx/raw \
  -H "Content-Type: application/json" \
  -d '{"txhex": "<hex>"}'
```

The response body is the transaction ID as plain text on success, or the node's rejection message. HTTP status is 200 on success.

Pass an API key from the [WhatsOnChain dashboard](https://whatsonchain.com) in an `Authorization` header to raise the limit; paid tiers go up to 40 requests per second. WhatsOnChain describes this endpoint as intended for small-scale use and points commercial users elsewhere.

The TypeScript SDK ships a `WhatsOnChainBroadcaster` class that wraps this endpoint, and the Go SDK has an equivalent in its `broadcaster` package. The [custom broadcaster example](../guides/sdks/ts/examples/example_building_custom_tx_broadcaster.md) is built around it.

```ts
import { WhatsOnChainBroadcaster } from '@bsv/sdk'
const result = await tx.broadcast(new WhatsOnChainBroadcaster('main'))
```

## Bitails

Full documentation: [docs.bitails.io](https://docs.bitails.io/)

Single transaction:

```bash
curl -X POST https://api.bitails.io/tx/broadcast \
  -H "Content-Type: application/json" \
  -d '{"raw": "<hex>"}'
```

```json
{ "txid": "9f2b…" }
```

Several at once, in dependency order:

```bash
curl -X POST https://api.bitails.io/tx/broadcast/multi \
  -H "Content-Type: application/json" \
  -d '{"raws": ["<hex parent>", "<hex child>"]}'
```

The response is an array with one `{ txid, error }` entry per input. An entry's `error` object carries the node's `code` and `message`, for example code 64 for a transaction already in a block.

Transactions up to 32 MB can be sent as a file instead of a JSON string, which avoids holding a huge hex string in memory:

```bash
curl -X POST https://api.bitails.io/tx/broadcast/multipart \
  -F "raw=@tx.bin"
```

Add an `apikey` header to raise the free limits.

## BananaBlocks

Full documentation: [bananablocks.com/api/docs](https://bananablocks.com/api/docs)

```bash
curl -X POST https://bananablocks.com/api/v1/tx/broadcast \
  -H "Content-Type: application/json" \
  -d '{"rawtx": "<hex>"}'
```

Batch, up to 100 transactions and 32 MiB per request, as either a JSON array of `{ "rawtx" }` objects or plain text with one hex transaction per line:

```bash
curl -X POST https://bananablocks.com/api/v1/tx/broadcast/multi \
  -H "Content-Type: application/json" \
  -d '[{"rawtx": "<hex parent>"}, {"rawtx": "<hex child>"}]'
```

A batch response is HTTP 200 with one entry per transaction in submission order. Read each entry's `accepted` field rather than the HTTP status; a malformed or invalid transaction fails the whole batch with 400 and nothing is broadcast. A 502 or 503 may mean some transactions did get through, so check their status before rebuilding anything that spends the same inputs.

BananaBlocks also serves a WhatsOnChain-compatible mirror at `/api/v1/bsv/main` and `/api/v1/bsv/test`, so existing WhatsOnChain client code can be repointed by changing the host. Its `POST /tx/raw` on that path accepts the same `{ "txhex" }` body.

Anonymous broadcasting is limited to 10 transactions per minute per IP. Keyed tiers raise that to 60 or 120. Requests carrying a browser `Origin` header from another site are refused with 403, so call it from a server, not from client-side JavaScript. Responses include `X-RateLimit-Limit` and `X-RateLimit-Remaining` headers, and exceeding a limit returns 429.

## Working with explorer APIs

- **Already known is success.** If you resubmit a transaction the explorer's node has seen, you get an error such as "already in the mempool" or "already in block chain". Treat these as confirmation that an earlier attempt worked.
- **Send parents before children.** These endpoints do not accept ancestor bundles. If a transaction spends an unconfirmed output, broadcast the parent first and wait for a successful response, or use a multi-transaction endpoint and list them in order.
- **Confirm independently.** A relayed transaction is in one operator's node. Poll the explorer's transaction endpoint, or query a second source, before treating it as seen by the network.
- **Plan for the limit.** The free tiers are counted per IP. A burst of a few hundred transactions will be throttled on every one of these services. BSV Broadcaster's public endpoints have no such ceiling.
