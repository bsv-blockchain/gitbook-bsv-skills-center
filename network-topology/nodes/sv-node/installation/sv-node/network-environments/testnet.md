# Testnet

## Getting Started with Bitcoin testnet

This guide will help you in setting up a BitcoinSV node which connects to the _testnet_.

### What is testnet?

Testnet is a public test network used for general testing. It is typically used to test software before it is released onto _mainnet_. As _testnet_ is public and unmanaged, software on _testnet_ may not be 100% stable.  Transactions per second and block difficulty are low.

System requirements for running a Testnet node: [system-requirements.md](../../../system-requirements.md "mention")

### QuickStart Instructions

#### Step 1:

Install SV Node according to the installation guide: [..](../ "mention")

#### Step 2:

Configure bitcoind to connect to the _testnet_ and not Mainnet, as well as setting mandatory parameters. Make the following changes to your `bitcoin.conf`

```editorconfig
testnet=1
maxstackmemoryusageconsensus=2000000000
excessiveblocksize=10000000000
minminingtxfee=0.00000001
```

#### Step 3:

If you wish to add any other custom configurations to your Bitcoin SV node you can appended them to the bitcoin.conf file with the editor of your choice.

#### Step 4:&#x20;

Start the bitcoind process. If you have been following the installation guide you can use systemd

```bash
sudo systemctl start bitcoind.service
```

If this is the first time you have started the node, it may take several hours or even days as the node downloads blocks and checks that they have not been tampered with. In this case, it may make sense to run the node in foreground to see status messages.

If this is not the first time you have run the node, it should start up quickly and it may make sense to run the node in the background.

#### Step 5:

&#x20;Check the status of the node and the bitcoind process. Type the following at the command line:

```bash
[bitcoin-sv installation directory]/bin/bitcoin-cli getinfo
```

This should generate an output similar to

```json
{
  "version": 101001600,
  "protocolversion": 70016,
  "walletversion": 160300,
  "balance": 0.00000000,
  "initcomplete": true,
  "blocks": 1595779,
  "timeoffset": 0,
  "connections": 12,
  "proxy": "",
  "difficulty": 1,
  "testnet": true,
  "stn": false,
  "keypoololdest": 1705486144,
  "keypoolsize": 2000,
  "paytxfee": 0.00000000,
  "relayfee": 0.00000000,
  "errors": "",
  "maxblocksize": 10000000000,
  "maxminedblocksize": 4000000000,
  "maxstackmemoryusagepolicy": 100000000,
  "maxstackmemoryusageconsensus": 100000000
}
```

### Funding

If you wish to use _testnet_ for testing, you will need to obtain unspent coins. A number of faucets are available:
* [https://bsvfaucet.com](https://bsvfaucet.com)
* [https://scrypt.io/faucet](https://scrypt.io/faucet)
* [https://testnet.satoshisvision.network](https://testnet.satoshisvision.network)
* [https://witnessonchain.com/faucet/tbsv](https://witnessonchain.com/faucet/tbsv)
* [https://testnet.help/en/bsvfaucet/testnet](https://testnet.help/en/bsvfaucet/testnet)

