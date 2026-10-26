---
description: >-
  Run your own BSV Broadcaster instance in about ten minutes, from source or from the
  container image, on mainnet, testnet, teratestnet, or regtest.
---

# Run Your Own BSV Broadcaster

BSV Broadcaster is open source under the Open BSV licence and is designed to be run by anyone. A single binary with an embedded key-value store is enough for a working instance; no PostgreSQL, Kafka, or other services are required. Running your own gives you a broadcaster that is entirely under your control, which is the right choice for any production application and a sensible one for serious development.

{% embed url="https://github.com/bsv-blockchain/broadcaster" %}

{% hint style="warning" %}
**Already running Arcade?** Version 0.19.0 renamed the default storage path (`~/.arcade` to `~/.bsv-broadcaster`), the config directory, the environment variable prefix, and the container image. The commands on this page use the new defaults, so running them against an existing deployment silently starts from empty storage, with no status records, block headers, or p2p identity. Before upgrading, keep your existing Docker volume (for example `arcade-data`), pin `storage_path` and `store.pebble.path` to your old locations, and follow the [migration guide](https://github.com/bsv-blockchain/broadcaster/blob/main/docs/migration-arcade-to-bsv-broadcaster.md). Legacy `ARCADE_*` variables and `/etc/arcade` are still read, with a deprecation warning.
{% endhint %}

## Why run your own

- **No third-party dependency.** The public endpoints are shared, best-effort infrastructure operated by others. Your instance is up when you say it is.
- **Private callbacks.** Webhook delivery, SSE streams, and status records stay inside your network.
- **Your own limits.** No shared rate limits, and policy values you set yourself.
- **Any network.** Mainnet, the public testnet, teratestnet for Teranode-scale testing, or a private regtest.
- **Proofs for your transactions.** Merkle paths are delivered to you as blocks are found, without you ingesting the chain.

## What you need

- Go 1.27 or newer with cgo enabled, or a container runtime if you prefer the image.
- About 1 GB of disk for the embedded store and block headers.
- Outbound network access so the instance can discover Teranode peers.
- For status tracking beyond `RECEIVED`, a URL at which the Merkle Service can reach your instance. See [Enable status tracking](#enable-status-tracking) below.

## Install

{% tabs %}
{% tab title="From source" %}
```bash
git clone https://github.com/bsv-blockchain/broadcaster.git
cd broadcaster
go build -o bsv-broadcaster ./cmd/bsv-broadcaster
```
{% endtab %}

{% tab title="Container image" %}
The published image is `ghcr.io/bsv-blockchain/broadcaster:latest`. It is distroless, so it contains only the binary; mount your configuration and expose the API and health ports.

```bash
docker run -d --name bsv-broadcaster \
  -v "$(pwd)/config.yaml:/etc/bsv-broadcaster/config.yaml:ro" \
  -v bsv-broadcaster-data:/data \
  -p 8080:8080 -p 8081:8081 \
  ghcr.io/bsv-blockchain/broadcaster:latest --config /etc/bsv-broadcaster/config.yaml
```

Set `storage_path` and the Pebble `path` in your configuration to somewhere under `/data` so state survives container restarts.
{% endtab %}
{% endtabs %}

## Configure

Save this as `config.yaml`. It runs every BSV Broadcaster service in one process, uses the embedded store, and discovers Teranode peers automatically.

```yaml
mode: all                      # run every service in one process
log_level: info
network: mainnet               # mainnet | testnet | teratestnet | regtest
storage_path: ~/.bsv-broadcaster

api:
  host: 127.0.0.1              # bind privately; see Security below
  port: 8080

health:
  port: 8081

kafka:
  backend: memory              # in-process event bus; no broker needed

store:
  backend: pebble              # embedded key-value store
  pebble:
    path: ~/.bsv-broadcaster/pebble
    memtable_size_mb: 64
    l0_compaction_threshold: 4
    sync_writes: false

p2p:
  datahub_discovery: true      # find Teranode endpoints over libp2p

chaintracks_server:
  enabled: true                # serve block headers at /chaintracks

# Status tracking (see "Enable status tracking" below)
callback_url: "https://broadcaster.example.com/api/v1/merkle-service/callback"
callback_token: "<output of: openssl rand -hex 32>"

merkle_service:
  url: "https://merkle-service-us-1.bsvb.tech"
```

Any value can be overridden by an environment variable prefixed `BSV_BROADCASTER_`, for example `BSV_BROADCASTER_NETWORK=testnet` or `BSV_BROADCASTER_LOG_LEVEL=debug`.

### Choosing a network

| `network` | Use | Merkle Service URL |
| --- | --- | --- |
| `mainnet` | Production BSV. Real money. | `https://merkle-service-us-1.bsvb.tech` |
| `testnet` | The public BSV testnet. Built-in bootstrap peers. | `https://merkle-service-testnet-us-1.bsvb.tech` |
| `teratestnet` | The Teranode scaling test network. Built-in bootstrap peers. | `https://merkle-service-ttn-us-1.bsvb.tech` |
| `regtest` | Your own private network. | None public; run your own. |

For regtest, peer discovery cannot use public bootstrap peers, so set `p2p.bootstrap_peers` to your own Teranode's multiaddr, or set `datahub_discovery: false` and list `datahub_urls` directly. The embedded chain tracker is disabled automatically on regtest.

## Enable status tracking

This is the step most often missed. BSV Broadcaster delegates proof construction to a separate **Merkle Service**, which watches the network for the transactions your instance has registered and calls back into your instance as they are seen, mined, and buried. The three keys at the bottom of the configuration above wire that up:

- `merkle_service.url` is the Merkle Service for your network. The BSV Association hosts public instances, listed in the table above.
- `callback_url` is the address at which **the Merkle Service can reach your instance**, with the path `/api/v1/merkle-service/callback`. It must be routable from the Merkle Service, which means a public hostname or a tunnel for a laptop. For a local setup with Merkle Service in Docker on the same host, `http://host.docker.internal:8080/api/v1/merkle-service/callback` works.
- `callback_token` authenticates those inbound callbacks. Generate it with `openssl rand -hex 32`. BSV Broadcaster refuses to start if `merkle_service.url` is set without a token, because an unauthenticated callback endpoint would accept forged status updates.

{% hint style="warning" %}
Without a Merkle Service, BSV Broadcaster still validates and broadcasts, but every transaction stays at `RECEIVED` forever and no merkle paths are ever produced.
{% endhint %}

## Run and verify

```bash
./bsv-broadcaster --config config.yaml
```

```bash
curl -s http://localhost:8081/health            # liveness
curl -s http://localhost:8080/health | jq       # API health with the Teranode endpoints it found
curl -s http://localhost:8080/policy            # the fee and size limits it will enforce
open http://localhost:8080/                     # interactive API docs
```

The `datahub_urls` array in the API health response should list several Teranode endpoints marked `healthy: true` within a minute of start-up. If it is empty, the instance cannot reach the peer network; check outbound connectivity.

Submit a transaction and watch it:

```bash
curl -s -X POST http://localhost:8080/tx -H "Content-Type: text/plain" --data "<hex>"
curl -s http://localhost:8080/tx/<txid>
```

With the Merkle Service configured, the status should reach `SEEN_ON_NETWORK` within seconds and `MINED` with a `merklePath` once a block is found.

## Point your application at it

Every example on the [BSV Broadcaster page](bsv-broadcaster.md#using-bsv-broadcaster-from-the-sdks) and the [Quickstart](quickstart.md) works unchanged against your own instance; replace the public URL with `http://localhost:8080` or wherever you expose it.

## Going to production

- **Separate the services.** `mode: all` is ideal for one machine. For resilience, run `api-server`, `propagation`, `webhook`, `sse`, `bump-builder`, `p2p-client`, and `chaintracks` as separate processes with `--mode <name>`, backed by an external Kafka cluster and an Aerospike, PostgreSQL, or MongoDB store. Start from `config.example.yaml` in the repository and the Kubernetes manifests under `deploy/`.
- **Pre-create Kafka topics** when using external Kafka. BSV Broadcaster does not create them, and the propagation topic has a fixed partition-count requirement described in `docs/production-kafka.md`.
- **Run more than one instance** if you need throughput or regional presence. Status records are per instance, so route status lookups to the instance that received the submission, or have the instances share a store.
- **Watch the SSE port.** The SSE service listens separately from the API, on port 8082 by default. Expose it if clients will use `/events`.
- **Monitor it.** `/metrics` exposes Prometheus metrics, and `/ready` and `/health` are suitable Kubernetes probes. The `blockHeight` in the health response lets you detect an instance whose chain view is lagging.

## Security

- **Do not expose the API unauthenticated.** It has no authentication by default. Bind `api.host` to a private interface, or put it behind a reverse proxy that enforces a bearer token, and keep it off the public internet unless you intend to run a public service.
- **The Merkle Service callback endpoint is bearer-authenticated** by `callback_token`. It is the only inbound route your instance needs from the outside.
- **Callback URLs you accept from clients are checked** against private and loopback address ranges, so a client cannot use your instance to probe your internal network.

The repository's [getting started guide](https://github.com/bsv-blockchain/broadcaster/blob/main/docs/getting-started.md) and [`config.example.yaml`](https://github.com/bsv-blockchain/broadcaster/blob/main/config.example.yaml) document every option.
