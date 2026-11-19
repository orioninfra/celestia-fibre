# Updates

What changed on this page and why. The newest entries are at the top.

## 2026-11-19: v11.0.0, app version 11

- `v11.0.0` is an emergency, coordinated upgrade to app version 11, announced for Mainnet Beta on Nov 19, 2026. It restricts the accounts whose keys were exposed in the Keplr/Web3Auth incident. The release badge, intro and source links now point to it.
- Timeline: added the announcement, what changes, and that bridge and light nodes need celestia-node `v0.36.0`.
- Section 08: covers how a new app version differs from a minor release, with the signal, tally and upgrade-height commands, the 5/6 threshold, the 232,616-block delay, and running the matching `fibre` binary.
- Fibre's own documentation and config did not change between `v10.6.0` and `v11.0.0`.

## 2026-11-10: v10.6.0 on Mainnet Beta

- `v10.6.0` was announced for Mainnet Beta on Nov 9, 2026. It is now the recommended release, and the release badge, intro and source links were updated.
- Timeline: added the announcement, with a link to the upgrade notice.

## 2026-11-05: v10.6.0 released

- Timeline: `v10.6.0` is released for Mainnet Beta, and Mocha upgraded to `v10.6.0-mocha`. It is a minor update without an upgrade height, but operators are asked to upgrade promptly. The mainnet announcement is pending.
- Section 02: glibc requirements are now split. `fibre` needs 2.34+, and the `celestia-appd` multiplexer needs 2.38+.
- Section 06: since `v10.5.0` the object storage endpoint must be `https://`, and the bucket is checked at startup.
- Section 08: covers how to roll out minor v10 releases and which Go version source builds need.
- Section 09: added a row for startup failures caused by the object storage changes.

## 2026-10-26: v10.5.0

- `v10.5.0` is now the recommended Mainnet Beta release (Mocha: `v10.5.0-mocha`). The release badge, intro and source links were updated.
- New **Health checks** subsection in section 07, covering gRPC readiness and liveness probes, the six built-in checks, and what they cannot see.
- Section 08: graceful shutdown now reports `NOT_SERVING` before requests drain.
- Section 09: troubleshooting now starts with the readiness probe.

## 2026-10-22: v10 activation on Mainnet Beta

- App version 10 activated on Oct 20, 2026 at upgrade height `15116658`. The first v10 block was `15116659`, at 03:45 UTC.
- Added a status note under the intro, and the activation row in the timeline now shows the date.
- Section 02 now notes that the pre-activation checklist still applies to new setups and to Mocha.

## 2026-10-17: first version

- Operator recommendations for the Fibre server, based on celestia-app `v10.4.0`: timeline, topology, pre- and post-activation steps, signing, memory sizing, storage, monitoring, upgrades and failure modes.
