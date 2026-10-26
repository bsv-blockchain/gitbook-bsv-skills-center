---
description: >-
  Broadcast through a node you operate, or through the public SV Node RPC
  endpoint, with sendrawtransaction. Plus direct submission to Teranode.
---

# Node RPC

If you run a node, you can broadcast through it. The transaction enters that node's mempool after the same validation miners apply, and the node relays it to its peers. There is no intermediary, no rate limit except what your hardware imposes, and the same connection serves every other query your application needs.

If you do not run a node, the BSV Association sponsors a public SV Node RPC endpoint that accepts `sendrawtransaction` without authentication. It is a reasonable fallback, not a primary route.

## SV Node

The SV Node JSON-RPC interface is documented under [RPC Interface](../network-topology/nodes/sv-node/rpc-interface/README.md), with the full method list on [RPC Methods](../network-topology/nodes/sv-node/rpc-interface/rpc-methods.md). Two methods broadcast transactions.

### `sendrawtransaction`

```bash
curl -s --user rpcuser:rpcpassword http://127.0.0.1:8332 \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"1.0","id":"bcast","method":"sendrawtransaction","params":["<hex>"]}'
```

```json
{ "result": "9f2b…", "error": null, "id": "bcast" }
```

The result is the transaction ID. Two optional boolean parameters follow the hex: `allowhighfees` to bypass the absurd-fee guard, and `dontcheckfee` to accept a transaction below the node's minimum fee rate into the mempool regardless.

Errors come back in the JSON-RPC `error` object:

| Code | Meaning |
| --- | --- |
| `-22` | The hex did not decode to a transaction. |
| `-26` | Rejected by mempool policy or consensus. The message names the reason: `txn-mempool-conflict`, `insufficient priority`, `non-final`, `bad-txns-inputs-missingorspent`, and so on. |
| `-27` | Already in the blockchain. Treat as success. |

### `sendrawtransactions`

Submits a batch in one call. Each element names the transaction and can carry its own policy overrides:

```bash
curl -s --user rpcuser:rpcpassword http://127.0.0.1:8332 \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"1.0","id":"batch","method":"sendrawtransactions","params":[[
        {"hex":"<hex parent>"},
        {"hex":"<hex child>","dontcheckfee":true}
      ]]}'
```

The result lists only the exceptions: `known` for transactions the node had already seen, `evicted` for ones accepted and then dropped for insufficient fee, and `invalid` for ones that failed validation, each with a `reject_code`, `reject_reason`, and for double spends the conflicting transactions under `collidedWith`. A transaction that appears in none of the arrays was accepted. List transactions in dependency order, parent before child.

### Public endpoint

The [Public RPC Endpoints](../network-topology/nodes/sv-node/rpc-interface/public-rpc-endpoints.md) page describes the hosted SV Node at `https://bsv-rpc.publicnode.com`. It allows `sendrawtransaction`, requires no credentials, and is CORS-enabled.

```bash
curl -s https://bsv-rpc.publicnode.com \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"1.0","id":"bcast","method":"sendrawtransaction","params":["<hex>"]}'
```

It is shared, best-effort infrastructure. Use it as a last-resort fan-out target when your primary route is down, or for prototyping, and read that page's guidance before relying on it in production.

## Teranode

Teranode accepts transactions through its Propagation service, which exposes an HTTP listener with two routes: `POST /tx` for a single transaction and `POST /txs` for a batch, both taking raw transaction bytes in [Extended Format](https://bsv.brc.dev/transactions/0030). The same operations are available over gRPC as `ProcessTransaction` and `ProcessTransactionBatch`, with batches limited to 1,024 transactions and 32 MB.

```bash
curl -X POST http://<teranode-host>:<propagation-port>/tx \
  -H "Content-Type: application/octet-stream" \
  --data-binary @tx.ef.bin
```

The TypeScript SDK's `Teranode` broadcaster targets this interface directly:

```ts
import { Teranode } from '@bsv/sdk'
const result = await tx.broadcast(new Teranode('http://<teranode-host>:<propagation-port>/tx'))
```

Teranode's propagation endpoint accepts and validates the transaction but does not track it afterwards or produce merkle paths. That is BSV Broadcaster's job, which is why [BSV Broadcaster](bsv-broadcaster.md) is the recommended front door to Teranode for applications. Submit directly only if you operate the Teranode yourself and have your own means of following the transaction. The listener address is set by the operator in the Propagation service settings; there is no public Teranode submission endpoint. See the [Teranode documentation](https://bsv-blockchain.github.io/teranode/) for the Propagation service reference.

## Operating guidance

- **Your node's view is authoritative for you.** When your own node accepts a transaction, you know it passed consensus and policy as that node applies them. When it rejects one, the error tells you exactly why, which is often more useful than a third-party API's response.
- **Relay is not confirmation.** The transaction is in your mempool and being gossiped. Check for it in a block, or run [BSV Broadcaster](bsv-broadcaster.md) alongside your node for lifecycle tracking.
- **Keep `txindex` on if you look up transactions later.** Without it, `getrawtransaction` only finds mempool entries and transactions in unpruned blocks.
- **Protect the RPC port.** Never expose it to the internet without authentication and TLS. The public endpoint exists precisely so that nobody has to.
