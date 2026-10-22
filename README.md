<h1 align="center">Celestia Fibre</h1>

<p align="center">
  <i>operator recommendations from Orion</i>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/NETWORK-celestia%20mainnet%20beta-black?style=flat-square"/>
  <img src="https://img.shields.io/badge/RELEASE-v10.5.0-grey?style=flat-square"/>
  <img src="https://img.shields.io/badge/V10-live%20since%20Oct%2020%2C%202026-grey?style=flat-square"/>
  <a href="https://docs.celestia.org/operate/consensus-validators/fibre/"><img src="https://img.shields.io/badge/SOURCE-docs.celestia.org-lightgrey?style=flat-square"/></a>
</p>

---

Starting with app version 10, every bonded Celestia validator is expected to run a **Fibre server**: a separate data availability service that receives blob shards from clients, stores them for the retention window, and serves them back.

> **v10 is live on Mainnet Beta.** It activated on Oct 20, 2026 at upgrade height `15116658`. If your Fibre server is not running and registered yet, go to [section 03](#03--after-activation-order-matters).

This page collects what we consider the right way to run it. The official docs explain *how*; this is *what to choose* and *what tends to break*. Every recommendation is based on the celestia-app `v10.5.0` documentation and Foundation notices; links are at the bottom.

<p align="center">
  <i>separate host • loopback or mTLS • colocated storage • watch the signer</i>
</p>

---

##  00 — Timeline

| When | What |
|---|---|
| Sep 15, 2026 | Requirements sheet published. Mocha and Mainnet Beta both start at **148 MB/s** |
| Oct 14, 2026 | Mainnet Beta Fibre server setup notice |
| Oct 20, 2026 | **app version 10 activated on Mainnet Beta** at upgrade height `15116658` (first v10 block `15116659`, 03:45 UTC). `x/fibre` and `x/valaddr` are live |
| November (target) | possible raise to **2.2 GB/s**. The Foundation will give notice before validators must provision for it |

Size bandwidth and disk with the Foundation's calculator sheet, not with a rule of thumb: the numbers depend on your voting power and the target throughput, and both change. Make your own copy of the sheet so you can rerun it.

---

##  01 — Topology

```text
                    clients (upload / download, always TLS)
                                   │
                                   ▼  :7980  (public, registered in x/valaddr)
                      ┌──────────────────────────┐
                      │       Fibre server       │──► object storage
                      │      dedicated host      │    (same region)
                      └──────────────────────────┘
                         │                    │
       app gRPC :9090    │                    │   privval gRPC :26669
       (no TLS, private) │                    │   (mTLS when remote)
                         ▼                    ▼
                      ┌──────────────────────────┐
                      │    celestia-app node     │──► KMS / signer
                      │       (validator)        │    (SignRawBytes, ≤10 ms)
                      └──────────────────────────┘
```

- **Run Fibre on its own host**, not on the validator. The Foundation recommends this for security, and it keeps Fibre's memory and disk pressure away from consensus.
- **Keep it close to the validator.** Endorsements wait on the signer, so the network path Fibre → node → signer is on the hot path.
- **The app gRPC link has no TLS.** Keep it on a private network or VPN, never on a public interface.

---

##  02 — Before activation

Prepare everything that does not need v10 to be live. Mainnet is already on v10, but the checklist still applies to anyone setting up Fibre now, and to Mocha.

* install the `fibre` binary from the **same** celestia-app release, verify `checksums.txt`, check `fibre version`
* on Linux use glibc **2.34+** (Ubuntu 22.04, Debian 12). On arm64 the archive is named `arm64`, even though `uname -m` prints `aarch64`
* put Fibre's home (`--home` / `FIBRE_HOME`) on a **different disk** from celestia-app data, so growth in one cannot fill the other
* in `app.toml`, set `[grpc] enable = true`. A fresh config ships with it off
* in `config.toml`, check `priv_validator_grpc_laddr`. Use `127.0.0.1` or `[::1]`, **not** `localhost`, which newer builds no longer treat as loopback. An empty value disables the service
* if the consensus key lives in a KMS, confirm it supports `SignRawBytes` **now**. Otherwise you find out after activation
* run the CPU benchmark from `celestia-app/tools/cpu_requirements` on the validator. The 128 MB / 6 s target expects 32 cores with GFNI and SHA-NI
* open port `7980` from the outside and test it from a machine **outside** your network

---

##  03 — After activation: order matters

```bash
# 1. confirm v10 is live
curl -s http://127.0.0.1:26657/abci_info        # app_version >= 10

# 2. start Fibre and read the startup log: chain ID, addresses, storage backend
fibre start --home /data/fibre \
  --app-grpc-address 127.0.0.1:9090 \
  --signer-grpc-address 127.0.0.1:26669 \
  --server-listen-address 0.0.0.0:7980

# 3. only now register the public address
celestia-appd tx valaddr set-host <public_host>:7980 --from <validator_account_key> --chain-id celestia

# 4. verify
celestia-appd query valaddr provider $(celestia-appd comet show-address)   # found: true
```

- **Start before you register.** A registered but unreachable host makes clients dial you and time out.
- **`set-host` before v10 is rejected.** The module does not exist yet.
- **Registration persists.** Rerun it only when the address changes, or after a long jail. The entry is removed if the validator is jailed or unbonded for more than 7 days.
- A DNS name works as well as an IP. TLS identity is bound to the consensus key, not to the address.

---

##  04 — Signing

Fibre never talks to the KMS. It asks the node's `PrivValidatorAPI`, and the node signs with whatever backend it uses.

* **local:** `127.0.0.1:26669`, plaintext, nothing else to configure
* **remote:** mutual TLS on both sides, with a **dedicated** CA. The node gets a server certificate whose SAN matches the address Fibre dials, and Fibre gets a client certificate. Set all three files (`ca`, `cert`, `key`) on each side
* **never** use `signer_grpc_allow_insecure` / `privval-grpc-allow-insecure` outside a lab. Anyone who reaches an unprotected signer can request signatures with your consensus key
* point the signer at the **validator**, not at a sentry. A sentry holds a different key, so clients get TLS identity verification failures
* after any signer change, **restart Fibre**. Its certificate is endorsed once at startup
* median signing latency must stay **≤ 10 ms**. The node warns when the median of the last 50 signatures exceeds it. A slow HSM or a far-away signer shows up here first

---

##  05 — Memory and connections

gRPC buffers a full upload message (~132 MiB) per in-flight stream:

```text
worst-case RAM ≈ max_connections × max_concurrent_streams × 132 MiB
```

| `max_connections` | `max_concurrent_streams` | worst-case buffers | fits |
|---|---|---|---|
| 16 (default) | 13 (default) | ≈ 26.8 GiB | 32 GiB host |
| 24 | 13 | ≈ 40.2 GiB | 64 GiB host |
| 32 | 13 | ≈ 53.6 GiB | 64 GiB host, little headroom |

**Our recommendation:** give Fibre a dedicated 64 GiB host and set `max_connections = 24`. A single upload uses 16 signers and takes all 16 default slots, which blocks downloads. Going above 16 keeps slots free for reads.

```toml
# server_config.toml
max_connections = 24
max_concurrent_streams = 13
```

Leave `min_upload_size` at its default (256 KiB) unless validators agree on a change together. It is a local admission policy, clients are not told about it, and mismatched minimums can stop uploads from collecting enough signatures.

---

##  06 — Storage

Local disk is the default and the simplest. If you use object storage (S3, R2):

* create the bucket in the **same region** as the Fibre host. A remote bucket adds seconds of latency and causes missed endorsements
* the metadata database still needs a persistent **local** disk
* credentials come from the environment only. For systemd, put them in `/etc/fibre/s3.env` (root, `0600`) and add `EnvironmentFile=` with `systemctl edit fibre.service`. Fibre refuses to start without them
* grant only `GetObject`, `PutObject`, `DeleteObject`, `ListBucket` (or R2 Object Read & Write), on that one bucket
* switching `storage_backend` affects **new** shards only. After going back to `local`, keep `[object_storage]` and the credentials until every object shard is pruned
* do not change the endpoint, bucket or prefix in place. For a migration: stop Fibre, sync objects with an identical key layout, update the config, start **once** with `--override-object-namespace`, then remove the flag. The flag does not copy or verify anything

---

##  07 — Monitoring

Fibre exports metrics and traces over OTLP/HTTP (`--otel-endpoint`, base URL only). Prometheus usually renames them, for example dots become underscores and units are added as suffixes, so check the final names in your backend.

| Watch | Metric | Why |
|---|---|---|
| no traffic | `fibre.server.upload_shard.last_success_timestamp` | the clearest "we dropped out" signal: registration, firewall or bonding |
| upload / download errors | `upload_shard.duration`, `download_shard.duration` counts with `success=false` | error rate straight from histogram counts |
| endorsement delay | `fibre.server.sign.duration` | time spent waiting on the signer for payment promises |
| signer latency | `cometbft_privval_signing_latency_*` (node side) | the **≤ 10 ms** requirement lives here, not in `sign.duration` |
| slot saturation | `upload_shard.in_flight`, `download_shard.in_flight` | sitting near `max_connections` means downloads are being starved |
| slow storage | `store.put.duration`, `backend.get.duration` (`outcome=timeout\|throttled`) | early sign of a far or throttled bucket |
| pruning | `prune.duration` with `success=false` | failed pruning slowly fills the disk |
| process | Go runtime (`fibre-runtime.json` dashboard) | memory against the buffer budget from section 05 |

- The reference stack in celestia-app publishes Prometheus on host port `9090`, which **collides with app gRPC**. On the validator, remap it (`9091:9090`).
- The official dashboards (`fibre.json`, `fibre-runtime.json`) expect a Prometheus datasource with UID `prometheus`.
- For a view from the outside, community dashboards already track reachability and endorsements: **Valopers** and **Tensile** by Huginn Tech.
- Keep `--pprof` off by default. When you need it, bind it to `localhost` only.

---

##  08 — Upgrades

* the `fibre` version must match the celestia-app release it ships with
* new config fields do not appear in an existing `server_config.toml` by themselves. After every upgrade run `celestia-appd config sync --fibre-home <fibre_home> --dry-run`, review the diff, then run it without `--dry-run`. It keeps your values and backs up the file
* unknown keys make Fibre fail at startup. Fix typos before restarting
* the first `SIGTERM` shuts down gracefully, a second one forces it. Give systemd enough `TimeoutStopSec`
* restarting rotates the TLS certificate automatically. There is nothing to renew

---

##  09 — When something is off

| Symptom | First check |
|---|---|
| runs, but no uploads | `valaddr provider` → `found`, an outside TCP connect to `host:7980`, bonding |
| `derived storage budget is 0` | the validator is not bonded, or `--app-grpc-address` points at a syncing node or the wrong network |
| TLS identity verification failed | the signer address points at a node without your consensus key, then restart Fibre |
| `payment promise verification failed` | chain ID mismatch between Fibre's node and the client. Escrow and stale height issues are on the client's side |
| `unknown service cosmos.base.tendermint...` | wrong gRPC address (RPC instead of app gRPC), or v10 is not active yet |
| `object namespace mismatch` | the bucket or prefix changed while object shards remain. Restore it, or migrate as in section 06 |

---

##  Sources

- [Fibre server setup, Celestia docs](https://docs.celestia.org/operate/consensus-validators/fibre/)
- [Fibre monitoring, Celestia docs](https://docs.celestia.org/operate/consensus-validators/fibre/metrics/)
- [`fibre/cmd/README.md`, celestia-app v10.4.0](https://github.com/celestiaorg/celestia-app/blob/v10.4.0/fibre/cmd/README.md)
- [CPU requirements tool, celestia-app](https://github.com/celestiaorg/celestia-app/blob/main/tools/cpu_requirements/README.md)
- [Mainnet Beta Fibre server setup, status.celestia.org](https://status.celestia.org/incidents/01M4XAT1AAHAM0QDR3623CE662)
- [Mocha and Mainnet Beta Fibre requirements sheet, status.celestia.org](https://status.celestia.org/incidents/01M2JMQ568VR39JYH97QFKGT02)
- [Introducing Fibre, Celestia blog](https://blog.celestia.org/introducing-fibre-1tb-s-of-blockspace/)

Fibre is moving fast. If something here no longer matches the current release, the official docs win. Open an issue and we will update the page.

---

<p align="center">
  <b>Orion</b><br/>
  <i>deterministic systems • continuous operation • no compromise</i>
</p>

<p align="center">
  <a href="https://orionlabsone.cc">orionlabsone.cc</a> · <a href="https://orionlabsone.cc/celestia">celestia daily brief</a> · <a href="https://github.com/orioninfra">github.com/orioninfra</a>
</p>
