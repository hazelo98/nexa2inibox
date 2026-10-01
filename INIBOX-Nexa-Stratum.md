# INIBOX → NEXA Stratum Guide

Field-verified notes for mining **NEXA** with **INIBOX-class ASIC/FPGA firmware**, plus the exact proof-of-work verification a pool must perform. Sanitized to protocol level only.

---

## 1. Scope

- **Covers:** wire framing, session handshake, the two stratum dialects (legacy INIBOX firmware vs modern GPU), notify/submit formats and job rotation, byte order, nonce reconstruction, rejection semantics and share-acceptance accounting, keepalives, legacy NexaPoW verification math, difficulty/targets, synthetic test vectors.
- **Does not cover:** payouts, fees, PPLNS/SOLO accounting, vardiff policy, storage, dashboards, deployment — implement those per your own design (see §15).
- **Chain mode:** pre-HF2 **legacy** NexaPoW. After a network hard fork, verification changes (a parent summary hash becomes an input) — re-verify against current network documentation before relying on this document.
- **No responsibility:** everything here is provided as-is, without warranty; no responsibility or liability is accepted for any loss of shares, blocks, revenue, or hardware arising from its use. Validate on your own hardware first.
- All examples are synthetic. `YourPool/1.0`, placeholder addresses, and random hex only.

---

## 2. Wire framing

The connection is a plain TCP stream carrying JSON objects.

**Modern framing:** one JSON object per line, `\n` terminated.

**Legacy firmware quirks (must be handled):**

1. **Multiple frames in one segment, no trailing newline.** INIBOX-class firmware can flush several objects (or a final object without `\n`) in a single TCP segment. A newline-only parser silently drops those shares. Parser algorithm: scan the buffer for brace-balanced complete objects (string/escape aware), parse each, keep the incomplete remainder for the next chunk:

```js
// find first '{' — skip garbage before it
// track depth, in-string, escapes; when depth returns to 0 → object complete
// push parsed object; continue after it; keep tail in session buffer
```

2. **Bare submit (no `method` field).** Firmware sends:

```json
{"id": 5, "params": ["nexa:<your_address>.rig1", "0a1b2c3d", "00112233445566778899aabbccddeeff"]}
```

If an object has no `method`, has an array `params` with ≥3 entries, and `params[1]` looks like an 8-hex job id — treat it as `mining.submit`.

3. **Recommended hygiene:** cap frame size and JSON nesting depth (drop connection on abuse); disconnect connections that never subscribe within ~30 s; drop idle connections (~120 s without data).

---

## 3. Session handshake

Strict order matters — clients wait for responses with matching `id`.

```
miner → {"id":1,"method":"mining.subscribe","params":["<userAgent>",null,"<sessionId>"]}
pool  → {"id":1,"result":[...subscribe result per dialect...],"error":null}      (§5)

miner → {"id":2,"method":"mining.authorize","params":["nexa:<your_address>.rig1","x"]}
pool  → {"id":2,"result":true,"error":null}

pool  → {"id":null,"method":"mining.set_difficulty","params":[<diff>], ...}      (immediately after authorize)
pool  → {"id":null,"method":"mining.notify","params":[...], ...}                 (first job)
```

- `params[0]` of subscribe = user agent (used for dialect detection, §4).
- Password in authorize is ignored; worker format = `nexa:<address>.<workername>`.
- All responses echo the request `id`. Correlation is by `id` only — there is no jsonrpc request/response matching in these dialects.
- Send `set_difficulty` **before** the first `notify`, once, with the final value (§14 quirk 1).

---

## 4. Dialect detection

Two dialects exist on the wire. Detect and tag the session at subscribe/authorize time (first match wins):

| Signal | → dialect |
|---|---|
| user agent matches `/ini(miner\|box)\|inibox/i` | `inibox` (legacy firmware) |
| worker name matches the same pattern | `inibox` |
| explicit server-side config | as configured |
| otherwise | `gpu` (modern stratum dialect) |

Every response format below is dialect-dependent.

---

## 5. Message envelopes and response formats

### Envelope rule

- **`inibox` dialect:** every message the **pool sends** carries `"jsonrpc":"2.0"` (responses, `mining.notify`, `mining.set_difficulty`, ping/pong).
- **`gpu` dialect:** no `jsonrpc` member anywhere.

### subscribe result

```json
// inibox — extranonce1 full 16 hex, extranonce2 size field = 8
{"id":1,"result":[null,"4a7f2c9e13b805d6",8],"error":null,"jsonrpc":"2.0"}

// gpu — subscription list, extranonce1 truncated to 8 hex chars, size = 8
{"id":1,"result":[[["mining.notify","9d8c7b6a5e4d3c2b"],["mining.set_difficulty","1f2e3d4c5b6a7988"]],"4a7f2c9e",8],"error":null}
```

### authorize result

```json
{"id":2,"result":true,"error":null}                                          // accepted (both)
{"id":2,"result":false,"error":null,"jsonrpc":"2.0"}                          // rejected, inibox
{"id":2,"result":null,"error":[24,"Unauthorized worker",null]}                // rejected, gpu
```

### submit result

```json
{"id":3,"result":true,"error":null,"jsonrpc":"2.0"}                           // accepted, inibox
{"id":3,"result":true,"error":null}                                           // accepted, gpu
{"id":3,"result":false,"error":null,"jsonrpc":"2.0"}                          // rejected, inibox (legacy keeps result:false — no error arrays)
{"id":3,"result":null,"error":[21,"Job not found",null]}                      // rejected, gpu
```

### set_difficulty / ping / pong / get_version

```json
{"id":null,"method":"mining.set_difficulty","params":[1],"jsonrpc":"2.0"}       // inibox (+jsonrpc)
{"id":null,"method":"mining.set_difficulty","params":[2]}                        // gpu  (no jsonrpc)
{"id":null,"method":"mining.ping","params":[],"jsonrpc":"2.0"}                // pool → miner keepalive (inibox)
{"id":7,"result":"pong","error":null,"jsonrpc":"2.0"}                         // miner → pool reply
{"id":8,"result":"YourPool/1.0","error":null}                                 // mining.get_version
```

### Unknown methods (feature negotiation)

Firmware probes methods it supports; respond:

- `inibox` → **success**: `{"id":<id>,"result":true,"error":null,"jsonrpc":"2.0"}` (strict error strings trip firmware failover logic)
- `gpu` → `{"id":<id>,"result":null,"error":[20,"Unknown method",null]}`
- methods starting with `client.` → ignore silently (no response)

---

## 6. mining.notify formats

```json
// inibox — 5 params: jobId, headerCommitment, nbits, time, cleanJobs
{"id":null,"method":"mining.notify",
 "params":["0a1b2c3d",
           "a1b2c3d4e5f60718293a4b5c6d7e8f90112233445566778899aabbccddeeff00",
           "1b0404cb",
           "0000000068e5f2a1",
           false],
 "jsonrpc":"2.0"}

// gpu — 4 params: jobId, headerCommitment, height, nbits (no jsonrpc)
{"id":null,"method":"mining.notify",
 "params":["0a1b2c3d",
           "a1b2c3d4e5f60718293a4b5c6d7e8f90112233445566778899aabbccddeeff00",
           1000000,
           "1b0404cb"]}
```

- `jobId`: opaque 8-hex string, unique per job. **Keep an archive of recent jobs (≥10 min)** — a stale share may still be a valid block.
- `nbits`: compact target, 8 hex chars (decode in §11).
- `time` (inibox only): unix timestamp, 8-byte big-endian hex, zero-padded to 16 chars.
- `cleanJobs`: broadcast `true` when height changed (miners drop old jobs); otherwise `false`.

---

## 7. Byte order (the classic integration bug)

> ⚠️ **Getting this wrong = ZERO blocks.** Shares keep being accepted and hashrate looks perfectly normal — but no block will ever hit. This is the hardest bug to spot, because the pool keeps running "fine" while quietly producing nothing. The node computes internally in little-endian (`CHashWriter << uint256`); the three parties (miner firmware, pool, node) must all agree on the same byte order.

| Value | Wire form |
|---|---|
| `headerCommitment` | **byte-reversed** relative to the node's candidate field — the pool must reverse it before broadcast. Miners hash the wire value **as-is**. |
| solution nonce | big-endian hex string of 16 bytes; hashed as raw bytes in the given order |
| `nbits`, `time` | normal big-endian hex of integers |
| `pow` result | **little-endian** 32 bytes — comparison via little-endian integer (§11); for display, reverse to big-endian hex |

---

## 8. Submit params and nonce reconstruction

Canonical submit: `params = [worker, jobId, solutionNonce, (time)?]`.

**Solution nonce = 32 hex chars (16 bytes):**

```
| first 16 hex = extranonce1 issued at subscribe | last 16 hex = miner-rolled nonce |
```

### inibox (and 4-param gpu): full nonce

`params[2]` is already the full 32-hex solution nonce. Normalize: lowercase, strip optional `0x`, left-pad to 32.

### gpu 5-param variants

Some firmware splits the nonce: `params[2]` = prefix `p2`, `params[4]` = miner nonce `p4` (8 bytes / 16 hex). Three known layouts — try all, pick the candidate with the **lowest pow value** (wrong layouts hash to random ~1-gate values; the true nonce always wins):

```
C1:  p2 ‖ p4
C2:  p2[0..8] ‖ "00000000" ‖ p4
C3:  extranonce1(16 hex full) ‖ p4
```

---

## 9. Rejection semantics and error codes

**GPU reject format:** `{"id":…,"result":null,"error":[<code>,"<message>",null]}`

| Code | Meaning |
|---|---|
| 20 | unknown method |
| 21 | job not found |
| 22 | duplicate share |
| 23 | low difficulty |
| 24 | unauthorized worker |
| 25 | not subscribed |
| 26 | rate limited |
| 27 | invalid params |

**INIBOX dialect:** any reject = `{"result":false,"error":null}` (never error arrays).

**Silent-accept rule (inibox dialect, firmware stability):** these conditions are answered `result:true` but **not recorded**:

- job not found / stale job (job archive miss)
- duplicate nonce for the same job
- malformed/invalid nonce string

Legacy firmware reacts badly (stalls/crashes) to rejects for these conditions. **Soft shares** (valid PoW below the gate) are accepted and may be logged/recorded as soft. GPU sessions should be rejected strictly by the table above.

---

## 10. Keepalives

| Dialect | Mechanism |
|---|---|
| `gpu` | re-send the **current notify every 15 s** (same jobId) — rental proxies idle-drop connections without fresh work |
| `inibox` | `mining.ping` every 15 s → expect `mining.pong` — stays ahead of NAT/firewall ~60 s idle kills |

**Do NOT periodically re-send jobs to inibox sessions** — the firmware restarts work on every notify, causing hashrate dips. Ping only.

---

## 11. PoW verification (legacy pre-HF2 NexaPoW)

Inputs: `headerCommitment` (64 hex, wire form), `solutionNonce` (32 hex), `target` (BigInt).

```
1.  miningHash = SHA256( SHA256( bytes(hc) ‖ 0x10 ‖ bytes(nonce) ) )
                                   ^ tag byte 0x10 between hc and nonce
2.  h1 = SHA256( miningHash )
3.  sig = schnorr_sign( msg = h1, key = miningHash ):
        d    = intBigEndian(miningHash) mod n            (n = secp256k1 order)
        k    = rfc6979(hmac-sha256, algo tag ASCII "Schnorr+SHA256  ") mod n
        R    = k·G ; if Y(R) odd → k ← n−k              (even-Y normalization)
        Pub  = compress(d·G)                             (33 bytes, 02/03 prefix)
        e    = intBigEndian( SHA256( Rx(32) ‖ Pub(33) ‖ h1 ) ) mod n
        s    = (k + e·d) mod n
        sig  = Rx(32) ‖ toB32(s)                         (64 bytes)
        if d, k or e is zero → invalid-key (rare; share unusable)
4.  pow = SHA256( sig )                                  (32 bytes, LITTLE-ENDIAN)
5.  valid  ⇔  leToBig(pow) ≤ target
       where leToBig(x) = Σ x[i] · 256^i   (x[0] = least significant byte)
```

**nbits (compact target) decode:**

```
e = (nbits >>> 24) & 0xff ;  s = nbits & 0x7fffff
target = s · 256^(e−3)   if e > 3
target = s >> (8·(3−e))   if e ≤ 3
s == 0 → invalid target
```

---

## 12. Difficulty and targets

```
DIFF1 = 0x0000FFFF × 256^26 = 0xffff0000000000000000000000000000000000000000000000000000   (224 bits)

shareTarget = DIFF1 / shareDifficulty            (shareDifficulty = fixed difficulty of the session)
share valid  ⇔  leToBig(pow) ≤ shareTarget
```

- **Block check is independent of the share gate:** a share below the gate ("soft") can still satisfy `leToBig(pow) ≤ nbitsTarget`. Run the block check on **every** accepted share.
- `shareDifficulty(pow) = DIFF1 / leToBig(pow)` — display metric only.
- Estimated hashrate ≈ `Σ shareGateDiff × 2^32 / seconds`.
- Use **fixed difficulty per port** for this hardware class (§14 quirk 1).

### The setting that decides acceptance

Difficulty is assigned **once per session, per stratum port**, immediately after `mining.authorize` succeeds:

```
<-- {"id":1,"result":true,"error":null}           authorize accepted
<-- {"id":null,"method":"mining.set_difficulty","params":[2]}
<-- {"id":null,"method":"mining.notify",...}      first job
```

- INIBOX ports use **fixed** difficulty (typically a low value such as 1–2); it never changes mid-session (§14 quirk 1). Auto-adjusting vardiff is a GPU-port feature.
- Two derived numbers control everything below:
  - **share gate:** `shareTarget = DIFF1 / diff` (constants above)
  - **expected share interval:** `interval ≈ diff × 2³² / hashrate`

| fixed diff | expected interval @ 850 MH/s |
|---|---|
| 1 | ≈ 5 s |
| 2 | ≈ 10 s |
| 5 | ≈ 25 s |
| 100 | ≈ 8.4 min |
| 1000 | ≈ 84 min |

**Acceptance check:** with a sane fixed difficulty the **first counted share arrives within seconds** — one or two jobs. If the device is connected but nothing is accepted for minutes, either the difficulty is far too high for its hashrate (table above) or the shares are being silently dropped — see the subsections below.

### Jobs, rotation and work validity

New work is **not on a fixed timer**: the pool polls its node (typically every second) and broadcasts a fresh `mining.notify` whenever the node's candidate actually changes (new transactions / timestamp). What the device sees:

- **New job** — a new `jobId` whenever the candidate changes; `cleanJobs = true` only when the height changed (§6). Keep mining the current job until the next notify arrives.
- **Refresh ≠ new work** — a re-sent notify with the **same `jobId`** (e.g. the 15 s GPU keepalive, §10) is a keepalive, not new work. Never send periodic job re-sends to inibox sessions (§10).
- **Work validity window** — a jobId stays submittable for ~2 minutes while current, then ~10 more minutes in the job archive (§6, §14 quirk 6). Submits against archived jobs are validated and counted normally — and may still be a block. After the archive expires → job not found (§9).
- **Operational rule** — the device must mine the **latest** jobId it received. Hashing an evicted jobId produces uncredited work: the link stays alive, replies stay `result:true`, nothing is counted (see below).

### What the pool counts as accepted

**The reply shown in the miner log and the share recorded by the pool are not always the same thing:**

| Outcome | PoW path run? | INIBOX reply | GPU reply | Counted by pool |
|---|---|---|---|---|
| full share (`leToBig(pow) ≤ shareTarget`) | yes | `result:true` | `result:true` | **yes** — credited |
| soft share (valid PoW, below gate) | yes | `result:true` | `result:true` | **yes** — credited at gate diff |
| stale job (unknown `jobId`) | no | `result:true` — *silent* | `error [21]` | **no** |
| duplicate nonce (same session) | no | `result:true` — *silent* | `error [22]` | **no** |
| duplicate nonce (another session) | yes | `result:false` | `error [22]` | no |
| malformed nonce (§8) | no | `result:true` — *silent* | `error [27]` | **no** |
| PoW path broken (§7 class) | attempted, no PoW | `result:true` — *silent* | `result:true` — *silent* | **no** |
| validation exception | no | `result:false` | `error [20]` | no |

> **Why a log full of "accepted" can still mean zero:** the INIBOX dialect answers stale / duplicate / malformed submits with `result:true` on purpose (§9 — firmware stability), but a pool can only persist shares it could actually verify (full or soft with PoW). **Miner says accepted, pool shows no shares → every submit is landing stale/invalid — fix job handling (§6), nonce reconstruction (§8) or byte order (§7).**

Soft shares are still counted (credit = gate difficulty), so a device sitting below the gate keeps moving the pool's counters — the hashrate estimate is `Σ shareGateDiff × 2³² / seconds` (see above).

### Alive vs counted

Worker liveness on a pool is judged by the **last counted (recorded) share**, not the TCP connection. Typical thresholds:

| last counted share | status |
|---|---|
| < 60 s | live |
| < 5 min | connected |
| < 2 h | idle |
| older | offline |

**Device "alive" but never accepted — order of checks:**

1. **Socket up, miner log shows accepted, pool counts nothing** → submits are going stale / invalid / unrecorded: job handling (`clean_jobs`, §6), nonce reconstruction (§8), **byte order (§7 — the classic zero-count bug)**.
2. **Shares counted, but far fewer than expected** → compare the actual rate against the interval table above; difficulty is likely far above the device's demonstrated hashrate.
3. **Explicit `result:false` on an INIBOX device** → cross-session duplicate (two machines submitting the same worker + job + nonce) or a validation exception — check for duplicate worker names.

---

## 13. Test vectors (synthetic)

Generated with the algorithm in §11. These verify byte-exact computation of `miningHash` and `pow`:

| # | headerCommitment | nonce |
|---|---|---|
| 1 | `a1b2c3d4e5f60718293a4b5c6d7e8f90112233445566778899aabbccddeeff00` | `00112233445566778899aabbccddeeff` |
| 2 | `0f1e2d3c4b5a69788796a5b4c3d2e1f00123456789abcdefedcba9876543210` | `ffeeddccbbaa99887766554433221100` |
| 3 | `deadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef` | `0123456789abcdef0123456789abcdef` |

| # | miningHash | pow (LE bytes) | leToBig(pow) = pow reversed | valid vs DIFF1 |
|---|---|---|---|---|
| 1 | `3824a7bbf3cb4ccff7e10f4efe5db9a34b1c348beaa0eadd6110a3b86ecd1a5a` | `2ddcae5e661569bf1741d017f7be4ca1b8aa75487851647fdb3a9cd2e04d9981` | `81994de0d29c3adb7f6451784875aab8a14cbef717d04117bf6915665eaedc2d` | false (ratio ≈ 4.599e-10) |
| 2 | `8b2b42614a65916e7562151e8c1b7f6e1363aa9b43073ecccce1317a6022fc61` | `5d9f525ce377655c3ed19cd59b126f97adffddc31a5feadd2cf3fa934f459c9c` | `9c9c454f93faf32cddea5f1ac3ddffad976f129bd59cd13e5c6577e35c529f5d` | false (ratio ≈ 3.806e-10) |
| 3 | `38746ed8c933ad027063f7cf8c7a3dee65216795698431d614c4d2a5069d1ceb` | `8e8598f1007f02099afbbd2e6dabaeb8233457a047b940790276555a247b180b` | `0b187b245a5576027940b947a0573423b8aeab6d2ebdfb9a09027f00f198858e` | false (ratio ≈ 5.372e-9) |

Notes:
- The three samples are random values — they exercise the **hashing/signing path byte-exactly**; all fall above DIFF1 (a real diff-1 share needs `leToBig ≤ DIFF1`, odds 1 in 2^32), so `valid = false` against the diff-1 gate is the **expected** result.
- Constants for your regression tests:
  - `DIFF1 = 0xffff0000000000000000000000000000000000000000000000000000`
  - `nbitsToTarget(0x1b0404cb) = 0x404cb000000000000000000000000000000000000000000000000`
- To create a guaranteed-valid vector at a chosen difficulty `D`, grind a nonce until `leToBig(pow) ≤ DIFF1/D` and add it to your own suite.

---

## 14. Field quirks (observed on hardware)

1. **Firmware ignores `mining.set_difficulty` changes after session start.** Send the final difficulty once, immediately after authorize; use fixed-difficulty ports. A session that keeps submitting at the old difficulty produces soft shares.
2. **Framing:** multi-frame/no-newline flushes and bare submits are routine (§2), not error cases.
3. **Rejects for stale/duplicate/invalid must be silent accepts** for the inibox dialect (§9) — visible rejects destabilize firmware.
4. **No periodic job re-notify** to inibox sessions — ping instead (§10).
5. **Unknown-method probing** at startup — respond success in the inibox dialect (§5).
6. **Stale ≠ worthless:** keep a job archive ≥10 min; a share for an evicted job can still be a block (run the block check anyway).
7. Fixed extranonce1 of 8 bytes (16 hex) with extranonce2-size field = 8 is known to work across firmware builds.

---

## 15. Out of scope / getting started

**Not in this document:** payouts and fees, PPLNS/SOLO reward accounting, vardiff algorithms, database schema, dashboards/APIs, reverse-proxy and host security. Design those yourself — this document intentionally stops at the wire.

**Two situations:** you already run a stratum listener → apply the integration delta below. Starting from nothing → use the bring-up order below it.

### Already have a pool (integration delta)

Running an existing NEXA stratum listener (Echelon-style or custom) and only adding INIBOX support? You do not rebuild — the delta is dialect-side and small:

| Area | What to change / add |
|---|---|
| Dialect detection | classify `inibox` vs `gpu` at subscribe/authorize from user agent / worker name (§4) |
| Response envelopes | inibox replies carry `"jsonrpc":"2.0"` and never error arrays — `result:true` / `result:false` with `error:null` (§5) |
| Notify | inibox gets the **5-param** form with `time` + `cleanJobs` (§6) |
| Submit parsing | inibox sends the **full 32-hex solution nonce** in `params[2]` — no extranonce assembly (§8) |
| Reject path | **silent-accept** stale / duplicate / malformed for inibox sessions only (§9); keep strict error codes for GPU |
| Keepalives | inibox: `mining.ping` → `mining.pong` only; **no periodic job re-sends** (§10) |
| Unknown methods | answer `result:true` for inibox feature probes (§5) |
| Difficulty | fixed difficulty per port; `set_difficulty` once, right after authorize (§12, §14 quirk 1) |
| Job archive | keep ≥10 min; validate archived-job shares and run the block check on every accepted share (§6, §12) |
| Byte order / PoW | already correct if your pool mines NEXA today — otherwise §7 + §11 are mandatory |

Everything else (nbits decode, job broadcast, GPU vardiff, payouts) is unchanged.

### New pool: bring-up order

1. Implement §11 and validate against §13 vectors — this must pass first.
2. Stand up a private listener (one TCP port, fixed difficulty, not published anywhere).
3. Connect real hardware; confirm the handshake (§3), notify (§6), and share acceptance (§9).
4. Run the block check (§12) against live jobs; only then consider wider exposure.

**Questions:** open an issue on this repository.
