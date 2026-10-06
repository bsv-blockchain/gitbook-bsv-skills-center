# SV Node

## Quick start

The [`svnode-quickstart`](https://github.com/bsv-blockchain/svnode-quickstart) scripts offer a streamlined way to set up a BSV Blockchain SV Node. These helper scripts automate various steps of the installation process, significantly reducing the time and effort required compared to manual setup. They provide an efficient solution for quickly getting an SVNode operational, while the detailed manual installation instructions outlined below offer a comprehensive guide for those who prefer or require a thorough understanding of each step.

The `svnode-quickstart` scripts also provide an option to sync a snapshot, which can significantly expedite getting your setup operational. However, users should carefully read the snapshots disclaimer regarding trust and security considerations [here](https://github.com/bsv-blockchain/svnode-quickstart/tree/main?tab=readme-ov-file#snapshot-trust-and-security-considerations) before proceeding.

## Manual installation

The following instructions describe installing BSV Blockchain SV Node using tools available in most mainstream Linux distributions. The assumption has been made that you are using a Bourne-like shell such as `bash`.

To start the install of SV Node, make sure you use an account that can use `su` or `sudo` to install software into directories owned by the root user.

Download the zipped release of your choosing, for this example we are using 1.2.3 which is the latest release at the time of writing:

<pre class="language-sh"><code class="lang-sh"><strong>wget https://releases-svnode.bsvblockchain.org/svnode-1.2.3/bitcoin-sv-1.2.3-x86_64-linux-gnu.tar.gz
</strong></code></pre>

Confirm downloaded file sha hash matches those provided at [download.bitcoinsv.io](https://download.bitcoinsv.io/bitcoinsv/) for the version you have downloaded.

```sh
sha256sum bitcoin-sv-1.2.3-x86_64-linux-gnu.tar.gz
# Expected Output 
# 13f222f5b466fe4af2bb867628354e4ae9711a10019e89fc7bf018b90e30b3c8  bitcoin-sv-1.2.3-x86_64-linux-gnu.tar.gz
```

Locate the file you downloaded and extract it using the `tar` command followed by the argument `xzf` followed by the file name. The argument `xzf` means eXtract the gZipped tar archive file. For example, for a 64-bit tar archive in your current directory, the command is:

```bash
tar xvf bitcoin-sv-1.2.3-x86_64-linux-gnu.tar.gz
```

Create a symbolic link from a new directory called `bitcoin` to the `bitcoin-sv-1.2.3` directory you just made by unzipping for easier use and updates:

```bash
ln -s bitcoin-sv-1.2.3 bitcoin
```

Create a `bitcoin-data` directory to put bitcoin data in (or else Bitcoin will put data in `~/.bitcoin` by default):

```bash
mkdir bitcoin-data
```

## Data folder considerations

The `bitcoin-data` folder will contain the logs, blocks, UTXO set and various other files the SV Node needs to function. For mainnet this folder will get very big, around 350GB for the UTXO set and 12TB for the blocks as of January 2024. The UTXO set store in `bitcoin-data/chainstate` is used for lookups to validate transactions and should be stored on a high-performant SSD. Depending on your use case, the `bitcoin-data/blocks` folder can be stored on slower, cheaper HDD storage.

If setting up the node in AWS, see [aws-volumes-setup.md](aws-volumes-setup.md "mention")for more details on a recommended setup.

## Config

Create a `bitcoin.conf` file in the directory to configure the settings to run your node using:

```bash
cd bitcoin-data/
vim bitcoin.conf
```

A detailed list of available options can be found in [configuration.md](configuration.md "mention"). Below is an example `bitcoin.conf` file used by a node on the **mainnet**:

```editorconfig
# start in background
daemon=1

# Select network -- comment out all for mainnet
#testnet=1
#stn=1
#regtest=1

# Maintain at most <n> connections to peers
maxconnections=20
# Maximum number of inbound connections from a single address.
# Not applicable to whitelisted peers.
maxconnectionsfromaddr=1

# Ports - Leave commented for defaults
#port=8333
#rpcport=8332

# Accept command line and JSON-RPC commands
server=1
rpcworkqueue=600
rpcthreads=16
#rpcallowip=0.0.0.0/0 
rpcuser=CHANGE_ME
rpcpassword=CHANGE_ME

# Required Consensus Rules for Genesis
excessiveblocksize=10GB
maxstackmemoryusageconsensus=100MB

# Mempool usage allowance
maxmempool=8GB

# Maintain a full transaction index, used by the getrawtransaction rpc call
txindex=1

# Cache options
dbcache=8GB
maxsigcachesize=256
maxscriptcachesize=256

# TX options
# Minimum mining transaction fee, 1 sat / kb
minminingtxfee=0.00000001
# Max number and size of related Child and Parent transactions per block template 
#limitancestorcount=100
#limitdescendantcount=100
#limitancestorsize=25000000
#limitdescendantsize=25000000

# ZeroMQ notification options
#zmqpubhashtx=tcp://127.0.0.1:28332
#zmqpubhashblock=tcp://127.0.0.1:28332

# Debug options
# can be: net, tor, mempool, http, bench, zmq, db, rpc, addrman, selectcoins,
#       reindex, cmpctblock, rand, prune, proxy, mempoolrej, libevent,
#       coindb, leveldb, txnprop, txnsrc, journal, txnval.
# 1 = all options enabled.
# 0 = all off (default)
#debug=1

# debugexclude to ignore set log items, can be used to keep log file a bit cleaner
debugexclude=libevent
debugexclude=leveldb
debugexclude=zmq
debugexclude=txnsrc
debugexclude=net

# Setting to 1 prevents bitcoind from clearning the log file on restart. 0/off is default
#shrinkdebugfile=0 

# Stores the block data in files of 2GB on disk
# the default of 128MB will result in lots of small files
preferredblockfilesize=2GB

# Mining, biggest block size you want to mine
blockmaxsize=4GB 
# When mining, consider switching to a pruned node
# Using prune and txindex is only possible in 1.1.1+
#prune=100000 # Keep only last ~100GB of blocks
#txindex=0

# Non-mining businesses that do not want to run the Alert System can enable
# the following settings to remain in sync with any validly processed
# DAR Alert Messages.
#enableassumewhitelistedblockdepth=1
#assumewhitelistedblockdepth=6

# Prevent possible memory exhaustion attacks
maxpendingresponses_getheaders=50
maxpendingresponses_gethdrsen=10

# Tunings options
#threadsperblock=32
#maxparallelblocks=4
#scriptvalidatormaxbatchsize=128
#maxparallelblocksperpeer=3
#maxstdtxvalidationduration=500
#maxstdtxnsperthreadratio=1000
#maxnonstdtxvalidationduration
```

## Systemd

To run Bitcoind, pass in the location of the configuration file as well as the location of where to store the bitcoin data:

```bash
# Example based on user
/home/user/bitcoin/bin/bitcoind \
-conf=/home/user/bitcoin-data/bitcoin.conf \
-datadir=/home/user/bitcoin-data -daemon
```

Create the `bitcoind.service` file:

```bash
sudo vim /etc/systemd/system/bitcoind.service
```

```systemd
[Unit]
Description=Bitcoin service
After=network.target
[Service]
Type=forking
# Make sure to replace username
ExecStart=/home/user/bitcoin/bin/bitcoind -conf=/home/user/bitcoin-data/bitcoin.conf -datadir=/home/user/bitcoin-data -daemon
ExecStop=/home/user/bitcoin/bin/bitcoin-cli -conf=/home/user/bitcoin-data/bitcoin.conf -datadir=/home/user/bitcoin-data stop
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-abnormal
TimeoutStopSec=300
PrivateTmp=true
LimitNOFILE=65536
# Make sure to replace username
User=user
[Install]
WantedBy=multi-user.target
```

Then start:

```bash
sudo systemctl start bitcoind.service
sudo systemctl enable bitcoind.service
```

The SV Node will now start and you can monitor progress in the log file. It will take several days for a fresh sync of the entire chain as of January 2024.

```bash
tail -f bitcoin-data/bitcoind.log
```
