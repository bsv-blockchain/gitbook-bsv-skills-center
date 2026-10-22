# Overview of GitHub repositories

The main BSV GitHub is available [here](https://github.com/orgs/bsv-blockchain/repositories), showcasing all public repositories, enabling developers to contribute directly or use as the foundation for their own needs.

This page highlights the main repositories currently available on GitHub:

## SDKs

The BSV Blockchain Libraries Project aims to structure and maintain a middleware layer of the BSV Blockchain technology stack. By facilitating the development and maintenance of core libraries, it serves as an essential toolkit for developers looking to build on the BSV Blockchain.

Three core libraries have been developed and made available:

* GO SDK: [https://github.com/bsv-blockchain/go-sdk](https://github.com/bsv-blockchain/go-sdk)
  * Script templates: [https://github.com/bsv-blockchain/go-template](https://github.com/bsv-blockchain/go-template)
* TypeScript SDK: [https://github.com/bsv-blockchain/ts-sdk](https://github.com/bsv-blockchain/ts-sdk)
* Python SDK: [https://github.com/bsv-blockchain/py-sdk](https://github.com/bsv-blockchain/py-sdk)

More information available [here](../guides/sdks/)

## SPV Wallet

The SPV Wallet is a comprehensive non-custodial wallet for BSV, enabling Simplified Payment Verification (as described in the Bitcoin White Paper section 8).

The main repository is available under this link: [https://github.com/bitcoin-sv/spv-wallet](https://github.com/bitcoin-sv/spv-wallet).

### Add-ons

In addition, the following repositories are related to SPV Wallets:

* Administrative console: [https://github.com/bitcoin-sv/spv-wallet-admin](https://github.com/bitcoin-sv/spv-wallet-admin)
* TypeScript client: [https://github.com/bitcoin-sv/spv-wallet-js-client](https://github.com/bitcoin-sv/spv-wallet-js-client)
* GO client: [https://github.com/bitcoin-sv/spv-wallet-go-client](https://github.com/bitcoin-sv/spv-wallet-go-client)
* Web-Frontend: [https://github.com/bitcoin-sv/spv-wallet-web-frontend](https://github.com/bitcoin-sv/spv-wallet-web-frontend)
* Web-Backend: [https://github.com/bitcoin-sv/spv-wallet-web-backend](https://github.com/bitcoin-sv/spv-wallet-web-backend)
* AWS cloud formation template generator: [https://github.com/bitcoin-sv/spv-wallet-aws](https://github.com/bitcoin-sv/spv-wallet-aws)
* Helm charts: [https://github.com/bitcoin-sv/spv-wallet-helm](https://github.com/bitcoin-sv/spv-wallet-helm)
* Key generator admin: [https://github.com/bitcoin-sv/spv-wallet-admin-keygen](https://github.com/bitcoin-sv/spv-wallet-admin-keygen)
* Fireblocks bridge: [https://github.com/bitcoin-sv/fireblocks-paymail-spv-bridge](https://github.com/bitcoin-sv/fireblocks-paymail-spv-bridge)

More in-depth information and guidance about SPV Wallets is available [here](https://github.com/bitcoin-sv/docs/blob/master/bsv-skills-center/bsv-protocol-documentation/light-clients-and-spv-processes/README.md)

### Block Headers Service

The Block Headers Service is a Go application used to collect and return information about blockchain headers.

[https://github.com/bitcoin-sv/block-headers-client](https://github.com/bitcoin-sv/block-headers-client)

## ARC

**ARC** is a multi-layer transaction processor for BSV Blockchain that keeps track of the lifecycle of a transaction as it is processed by the network.

The main repository is available here: [https://github.com/bitcoin-sv/arc](https://github.com/bitcoin-sv/arc)

Full details on ARC are not yet available in this BSV Skills Center, but can be found here:[https://bitcoin-sv.github.io/arc/#/](https://bitcoin-sv.github.io/arc/#/)

ARC is superseded by **BSV Broadcaster**, the BSV Association's supported broadcaster for Teranode: [https://github.com/bsv-blockchain/arcade](https://github.com/bsv-blockchain/arcade). See the [Broadcasting](../broadcasting/README.md) section for public endpoints, the API, and every other way to broadcast a transaction.

## SV Node

**SV Node** is the main node software used within BSV Blockchain. It is based on the original implementation of the Bitcoin protocol implemented as a monolith. The main repository is available here: [https://github.com/bitcoin-sv/bitcoin-sv](https://github.com/bitcoin-sv/bitcoin-sv)\
\
For more details, see [Node Operations](https://github.com/bitcoin-sv/docs/blob/master/bsv-skills-center/bsv-protocol-documentation/node-operations/README.md) and the [SV Node installation guide](../network-topology/nodes/sv-node/installation/).

There is ongoing work for an improved and scalable microservice implementation of the node software (Teranode) to support a much larger network throughput of transaction processing. For more information, visit [bsvblockchain.org/teranode](https://www.bsvblockchain.org/teranode)

### Alert System

The Alert System must be run together with the SVNode software, to ensure that a node is able to receive alerts.

The main repository is available here: [https://github.com/bitcoin-sv/alert-system](https://github.com/bitcoin-sv/alert-system)
