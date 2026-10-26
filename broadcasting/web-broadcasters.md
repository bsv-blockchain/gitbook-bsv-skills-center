---
description: >-
  Paste a raw transaction into a web form and send it. No code, no account.
---

# Web Broadcasters

Two block explorers offer a web page where you paste a hex-encoded transaction and submit it with a button. Nothing to install and nothing to authenticate. They are the quickest way to push a single transaction when you already have the hex, and a convenient way to check whether a transaction your application failed to send is actually valid.

| Service | Mainnet | Testnet |
| --- | --- | --- |
| WhatsOnChain | [whatsonchain.com/broadcast](https://whatsonchain.com/broadcast) | [test.whatsonchain.com/broadcast](https://test.whatsonchain.com/broadcast) |
| Bitails | [bitails.io/broadcast](https://bitails.io/broadcast) | [test.bitails.io/broadcast](https://test.bitails.io/broadcast) |

## How to use them

1. Serialise the signed transaction to hex. With the TypeScript SDK that is `tx.toHex()`; with Go, `hex.EncodeToString(tx.Bytes())`; with Python, `tx.hex()`. The SV Node RPC `signrawtransaction` also returns hex.
2. Open the broadcast page for the network the transaction belongs to.
3. Paste the hex and submit.
4. On success the page shows the transaction ID and links to the explorer's transaction view. On failure it shows the node's rejection reason, for example `mempool conflict`, `insufficient priority`, or `mandatory-script-verify-flag-failed`.

The pages accept only the plain raw format. If you have an Extended Format or BEEF serialisation, convert it to raw first; the SDKs do this with `tx.toHex()` regardless of how the transaction was loaded.

## When they are the right tool

- **One-off sends.** Moving funds from a paper key, consolidating a few outputs, or publishing a single data transaction.
- **Debugging.** If your application's broadcast fails, pasting the same hex here tells you in seconds whether the transaction is the problem or your integration is.
- **Support.** A user can send you a transaction they could not broadcast and you can push it for them without touching your production systems.
- **Learning.** The explorer immediately shows the decoded transaction, so you can see inputs, outputs, and scripts as the network sees them.

## When they are not

- Anything automated. Use [BSV Broadcaster](bsv-broadcaster.md) or an [API broadcaster](api-broadcasters.md).
- Transactions that spend unconfirmed outputs of transactions you have not broadcast yet. Send the parents first, in order.
- Very large transactions. Browser forms have practical limits well below what the network accepts. Use Bitails's multipart API or BSV Broadcaster's binary endpoint instead.

Both explorers relay to their own nodes and from there into the mining network. Neither gives you a status feed; refresh the explorer's transaction page or query its API to see when the transaction is mined.
