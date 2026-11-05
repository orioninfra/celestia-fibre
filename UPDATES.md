# Updates

What changed on this page and why. The newest entries are at the top.

## 2026-11-05: v10.6.0

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
