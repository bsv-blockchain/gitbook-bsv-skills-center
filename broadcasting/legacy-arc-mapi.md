---
description: >-
  mAPI is retired and ARC is superseded by BSV Broadcaster. What still works, who
  still runs it, and how to migrate.
---

# ARC and mAPI (Legacy)

Before BSV Broadcaster, BSV went through two generations of miner-facing broadcast APIs. Both are still referred to in older code, documentation, and SDK defaults, so it helps to know where they stand.

## mAPI

The Merchant API (mAPI) was the original miner-operated endpoint for fee quotes and transaction submission. Its [reference implementation](https://github.com/bitcoin-sv/merchantapi-reference) is archived and it is retired. Miners no longer operate mAPI endpoints and the SDKs no longer include clients for it. Code that targets mAPI must be migrated; there is no compatibility shim.

## ARC

[ARC](https://github.com/bsv-blockchain/arc) replaced mAPI as a multi-layer transaction processor that sits in front of a miner's nodes, validates submissions, and tracks each transaction's lifecycle with callbacks. It introduced the status vocabulary (`RECEIVED`, `SEEN_ON_NETWORK`, `MINED`, and so on), the `X-CallbackUrl` mechanism, and the `{ "rawTx" }` request shape that BSV Broadcaster carries forward.

ARC was designed for SV Node. BSV Broadcaster is its Teranode-native successor and the BSV Association's supported path going forward. ARC is no longer being developed by the BSV Association, but operators continue to run it.

### Who still runs ARC

| Operator | Mainnet | Testnet | Notes |
| --- | --- | --- | --- |
| TAAL | `https://arc.taal.com` | — | API key required for commercial use; see [docs.taal.com](https://docs.taal.com/) |
| GorillaPool | `https://arc.gorillapool.io` | `https://testnet.arc.gorillapool.io` | Open access. The TypeScript and Python SDKs use this as their default broadcaster. |

The ARC endpoints are versioned under `/v1`: `POST /v1/tx`, `POST /v1/txs`, `GET /v1/tx/{txid}`, and `GET /v1/policy`. Full reference: [bitcoin-sv.github.io/arc](https://bitcoin-sv.github.io/arc/).

### Using ARC from the SDKs

```ts
import { ARC } from '@bsv/sdk'
const arc = new ARC('https://arc.taal.com', { apiKey: '…', callbackUrl: 'https://your.app/hook' })
await tx.broadcast(arc)
```

```go
success, failure := tx.Broadcast(&broadcaster.Arc{ApiUrl: "https://arc.taal.com/v1", ApiKey: "…"})
```

```python
from bsv.broadcasters import ARC, ARCConfig
await tx.broadcast(ARC("https://arc.taal.com", ARCConfig(api_key="…")))
```

{% hint style="info" %}
Calling `tx.broadcast()` with no argument in the TypeScript or Python SDK sends the transaction to GorillaPool's ARC. That works today, but it is a dependency on a single third-party service that you did not choose. Pass an explicit broadcaster.
{% endhint %}

## Migrating from ARC to BSV Broadcaster

BSV Broadcaster keeps the parts of ARC's interface that clients depend on, so most migrations are a configuration change.

| | ARC | BSV Broadcaster |
| --- | --- | --- |
| Submit | `POST /v1/tx` | `POST /tx` |
| Batch | `POST /v1/txs` (JSON array) | `POST /txs` (concatenated raw bytes) |
| Status | `GET /v1/tx/{txid}` | `GET /tx/{txid}` |
| Policy | `GET /v1/policy` | `GET /policy` or `GET /v1/policy` |
| Success response | HTTP 200 | HTTP 202, same `{ txid, status, txStatus }` body |
| Formats | Raw, EF, BEEF | Raw, EF. BEEF is rejected; convert to EF. |
| `X-WaitFor` blocking wait | Supported | Not supported; use webhooks or SSE |
| `DOUBLE_SPEND_ATTEMPTED` | May still resolve | Terminal |
| HTTP 400 | Various | Terminal validation failure; do not retry elsewhere |
| Error body | RFC 7807 (`title`, `detail`) | `{ error, reason }` plus ARC's `status`, `title`, `detail` when the failure maps to an ARC code |

Steps:

1. **Change the URL and drop the `/v1` prefix** on submit and status paths. In the Go SDK that means `ApiUrl` becomes the BSV Broadcaster root. In the Python SDK switch to the `Arcade` class. In the TypeScript SDK use the `Teranode` class or a custom `Broadcaster`; the `ARC` class hard-codes `/v1/tx`.
2. **Stop sending BEEF.** If your wallet exchanges BEEF with peers, convert to Extended Format before broadcasting. The SDKs do this when every input has its source transaction attached.
3. **Replace any `X-WaitFor` polling** with a webhook (`X-CallbackUrl`) or the SSE stream.
4. **Treat `DOUBLE_SPEND_ATTEMPTED` and HTTP 400 as final** in your retry logic.
5. **Re-read policy.** Fee floors and size caps differ between operators. Query `GET /policy` on the instance you move to.

See the [BSV Broadcaster page](bsv-broadcaster.md) for the full API and public endpoints.
