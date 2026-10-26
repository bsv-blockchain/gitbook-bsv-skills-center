---
description: Transaction Broadcasting from SPV Wallet
---

# Broadcasting

{% hint style="info" %}
This page describes how the SPV Wallet component broadcasts internally. For every way to broadcast a transaction to BSV, including the BSV Association's supported [BSV Broadcaster](../../../broadcasting/bsv-broadcaster.md) service, see the [Broadcasting](../../../broadcasting/README.md) section.
{% endhint %}

The SPV Wallet broadcasts all valid transactions it receives or creates to ARC.

{% embed url="https://github.com/bitcoin-sv/go-broadcast-client/" %}

## ARC Endpoints

We use the first endpoint to determine the correct fee model to use when creating a transaction.

{% swagger src="https://bitcoin-sv.github.io/arc/arc.json" path="/v1/policy" method="get" %}
[https://bitcoin-sv.github.io/arc/arc.json](https://bitcoin-sv.github.io/arc/arc.json)
{% endswagger %}

Thereafter we simply broadcast to ARC and expect a `SEEN_ON_NETWORK` txStatus response in most cases.

{% swagger src="https://bitcoin-sv.github.io/arc/arc.json" path="/v1/tx" method="post" %}
[https://bitcoin-sv.github.io/arc/arc.json](https://bitcoin-sv.github.io/arc/arc.json)
{% endswagger %}

Usually a callbackUrl would be set for async status updates - but if you'd like to manually check the most recent state of a given transaction, you can use this:

{% swagger src="https://bitcoin-sv.github.io/arc/arc.json" path="/v1/tx/{txid}" method="get" %}
[https://bitcoin-sv.github.io/arc/arc.json](https://bitcoin-sv.github.io/arc/arc.json)
{% endswagger %}
