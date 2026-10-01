# nexa2inibox

Field-verified stratum notes for mining **NEXA** with **INIBOX-class ASIC/FPGA hardware**.

## Contents

- [INIBOX-Nexa-Stratum.md](./INIBOX-Nexa-Stratum.md) — the full guide: wire framing, session handshake, dialect detection, notify/submit formats, byte order, nonce layouts, rejection semantics, share acceptance lifecycle, job rotation, keepalives, PoW verification math, difficulty/targets, synthetic test vectors, field quirks.

## Who this is for

Pool developers adding INIBOX dialect support to an existing NEXA stratum listener, and operators bringing up a compatible listener from scratch.

## Scope

Protocol only: how the miner and the pool talk, and how a submitted solution is verified. No payouts, no accounting, no dashboards, no infrastructure.

## Disclaimer

Notes are field-verified against real hardware, but provided as-is, without warranty of any kind. **Use at your own risk: no responsibility or liability is accepted for any resulting loss of shares, blocks, revenue, or hardware.** Verify the test vectors (§13) against your implementation before going to production, then validate end-to-end with your own devices.
