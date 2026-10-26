---
description: >-
  Take a signed transaction from hex to mined in a few minutes using a public
  BSV Broadcaster endpoint, from the command line or any of the SDKs.
---

# Quickstart

This page gets one transaction onto the BSV blockchain and shows you how to watch it get mined. It uses a public [BSV Broadcaster](bsv-broadcaster.md) endpoint, so there is nothing to install or sign up for. Where it says `<hex>`, substitute your own signed transaction.

## 1. Get a signed transaction

You need a fully signed transaction serialised as hex. If you do not have one yet:

- Follow the [Quick Start](../intro/quick-start.md) in the Intro section, which funds a key and builds a transaction with the TypeScript SDK. Stop before the `tx.broadcast()` line and use `tx.toHex()` instead.
- Or use one of the SDK examples: [TypeScript](../guides/sdks/ts/examples/example_simple_tx.md), [Go](../guides/sdks/go/examples/create_simple_tx.md), or [Python](../guides/sdks/py/examples/example_simple_tx.md).
- Or export a raw transaction from any wallet that offers it.

The transaction must spend outputs that exist and are unspent, and must pay at least the fee rate the endpoint requires, which you check next.

## 2. Pick an endpoint and read its policy

Choose the public instance nearest to you. This page uses the BSV Association's US endpoint throughout; the full list is on the [BSV Broadcaster page](bsv-broadcaster.md#public-endpoints).

```bash
export BROADCASTER=https://arcade-us-1.bsvblockchain.tech
curl -s $BROADCASTER/policy
```

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

`miningFee` is the minimum rate the instance will accept, here one satoshi per kilobyte. If your transaction pays less, build it again with a higher fee before continuing.

## 3. Submit it

```bash
curl -s -X POST $BROADCASTER/tx \
  -H "Content-Type: text/plain" \
  --data "<hex>"
```

```json
{ "txid": "9f2b…", "status": 202, "txStatus": "RECEIVED" }
```

A `202` means BSV Broadcaster validated the transaction and is delivering it to the network. If you get a `400` instead, the body tells you what is wrong with the transaction itself; fix it and resubmit. Nothing else is at fault, so do not try another service with the same bytes.

Sending the same transaction twice is harmless. You get another `202` with whatever status it has reached.

## 4. Watch it progress

```bash
curl -s $BROADCASTER/tx/<txid>
```

Run that a few times. Within seconds `txStatus` should move from `RECEIVED` through `SENT_TO_NETWORK` to `SEEN_ON_NETWORK`, at which point miners have it and it is safe to treat as a zero-confirmation payment. Within roughly ten minutes it should reach `MINED`:

```json
{
  "txid": "9f2b…",
  "txStatus": "MINED",
  "timestamp": "2026-10-16T12:09:41Z",
  "blockHash": "0000…",
  "blockHeight": 971340,
  "merklePath": "<BUMP hex>"
}
```

The `merklePath` is a [BUMP](https://bsv.brc.dev/transactions/0074) you can verify against block headers yourself. That is the proof of inclusion an SPV client needs, and you got it without running a node.

If you would rather not poll, open a stream instead. Submit with an `X-CallbackToken: mytoken` header, then:

```bash
curl -N "$BROADCASTER/events?callbackToken=mytoken"
```

Every status change for transactions submitted with that token arrives as a server-sent event.

## 5. The same from the SDKs

Each snippet loads a transaction from hex and broadcasts it to the same endpoint.

{% tabs %}
{% tab title="TypeScript" %}
BSV Broadcaster's `/tx` endpoint takes a hex body, so a broadcaster is a few lines. The SDK's built-in `Teranode` class also works, but only when every input carries its `sourceTransaction`, which a transaction loaded from bare hex does not.

```ts
import { Transaction } from '@bsv/sdk'
import type { Broadcaster, BroadcastResponse, BroadcastFailure } from '@bsv/sdk'

class BsvBroadcaster implements Broadcaster {
  constructor (private readonly url: string) {}

  async broadcast (tx: Transaction): Promise<BroadcastResponse | BroadcastFailure> {
    const res = await fetch(`${this.url}/tx`, {
      method: 'POST',
      headers: { 'Content-Type': 'text/plain' },
      body: tx.toHex()
    })
    const body = await res.json()
    if (res.ok) {
      return { status: 'success', txid: body.txid, message: body.txStatus }
    }
    return { status: 'error', code: String(res.status), description: body.reason ?? body.error }
  }
}

const tx = Transaction.fromHex('<hex>')
console.log(await tx.broadcast(new BsvBroadcaster('https://arcade-us-1.bsvblockchain.tech')))
```
{% endtab %}

{% tab title="Go" %}
The Go SDK's `Arc` broadcaster posts to `{ApiUrl}/tx`, which is BSV Broadcaster's path, and accepts the `202` response.

```go
package main

import (
    "fmt"

    "github.com/bsv-blockchain/go-sdk/transaction"
    "github.com/bsv-blockchain/go-sdk/transaction/broadcaster"
)

func main() {
    tx, err := transaction.NewTransactionFromHex("<hex>")
    if err != nil {
        panic(err)
    }

    success, failure := tx.Broadcast(&broadcaster.Arc{
        ApiUrl: "https://arcade-us-1.bsvblockchain.tech",
    })
    if failure != nil {
        panic(failure)
    }
    fmt.Println(success.Txid, success.Message)
}
```
{% endtab %}

{% tab title="Python" %}
The Python SDK has a native `Arcade` class.

```python
from bsv.broadcasters import Arcade
from bsv import Transaction

tx = Transaction.from_hex("<hex>")
broadcaster = Arcade("https://arcade-us-1.bsvblockchain.tech")

result = broadcaster.sync_broadcast(tx)
print(result.txid, result.message)

print(broadcaster.check_transaction_status(result.txid))
```
{% endtab %}
{% endtabs %}

## Where to go next

- **Get notified instead of polling.** Submit with `X-CallbackUrl` and BSV Broadcaster will POST each status change to your server. See [Webhooks](bsv-broadcaster.md#webhooks).
- **Run your own BSV Broadcaster.** The public endpoints are shared. For private callbacks, your own policy, and no third-party dependency, [run an instance](run-your-own-bsv-broadcaster.md); it takes about ten minutes.
- **Use a different route.** Every other way to broadcast, and when each one makes sense, is on the [overview](README.md).
