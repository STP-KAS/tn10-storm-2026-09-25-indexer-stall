> **Part of [tn-master](https://github.com/STP-KAS/tn-master)**, the map of every TN10 storm. Renamed on 11 Oct 2026 from `tn10-indexer-stall-2026-09`; old links redirect here.
>
> ← previous storm: [tn10-storm-2026-09-25-build](https://github.com/STP-KAS/tn10-storm-2026-09-25-build) | next storm: [tn10-storm-2026-09-25-r1](https://github.com/STP-KAS/tn10-storm-2026-09-25-r1) →

# TN10 indexer stall (api-tn10.kaspa.org), Sep 2026

Private working note. Checked on 2026-09-29 at 10:55–10:57Z (12:55–12:57 CEST).
All times are given in UTC (Z) and CEST (UTC+2).

## What
The public TN10 REST API `api-tn10.kaspa.org` (a kaspa-rest-server and indexer DB behind Cloudflare) has a database whose
**accepted-transaction processing has not moved since 2026-09-25 19:55:38Z (21:55:38 CEST)**. `/info/health` returns
**HTTP 503** with `database.isSynced:false`. Node-backed endpoints still return 200 with live data.

## Why (question asked)
Did our TN10 stress test on 25–26 Sep 2026 break it? **Unproven.** The stall started while our storm was running
(≈6k net TPS). We have no indexer-side logs or error output, so we can't tell whether the storm caused it.
**Time correlation is not causation.**

## When
| Event | UTC | CEST | Source |
|---|---|---|---|
| kascov post about the 17 Sep 907-block reorg | 25 Sep 15:21:10Z | 17:21:10 | `evidence/kascov-x-post.txt` |
| Our storm supervisor starts P0–P7 workers (FEE_MULT 1.2) | 25 Sep 19:22:15Z | 21:22:15 | `evidence/storm-start.txt` |
| Last health response Cloudflare still serves as stale 200 (`acceptedTxBlockTime`) | 25 Sep 19:45:05Z | 21:45:05 | `evidence/api-health-2026-09-29.txt` |
| Overload probe P1 (fee tiers × 100 sompi/g) starts | 25 Sep 19:48:41Z | 21:48:41 | HANDOFF.md l.24 |
| **Indexer `acceptedTxBlockTime` freezes** | **25 Sep 19:55:38.262Z** | **21:55:38** | `evidence/api-health-samples-2026-09-29.txt` |
| Storage-mass-heavy blocks start showing up on our node (net TPS dips to 1.4–3k) | 25 Sep 19:56:46Z | 21:56:46 | `evidence/nettps-...txt` |
| Storm fee goes to 2x | 25 Sep 20:47Z | 22:47 | HANDOFF.md l.46 |
| Storm fee goes to 10x | 25 Sep 22:14Z | 26 Sep 00:14 | HANDOFF.md l.70 |
| 6x | 26 Sep 05:46Z | 07:46 | HANDOFF.md l.103 |
| 150x attempt, then back to 10x | 26 Sep 07:22:10Z / 07:51Z | 09:22:10 / 09:51 | HANDOFF.md l.112, 121–122 |
| Fee rule set to 2x node estimate (`max(2×normalBuckets[0],200)`) | 26 Sep 10:22Z | 12:22 | HANDOFF.md l.141 |
| Checked again: still frozen, 503 | 29 Sep 10:56Z | 12:56 | `evidence/api-health-*.txt` |

## Facts (with evidence)
1. **Health is 503 when the Cloudflare cache is bypassed.** At `2026-09-29T10:56:22Z`,
   `curl -D - "https://api-tn10.kaspa.org/info/health?nc=<ts>"` returned `HTTP/2 503`, `cf-cache-status: MISS`, body:
   `"database":{"isSynced":false,"blueScore":571964386,"blueScoreDiff":10,"acceptedTxBlockTime":1790366138262,"acceptedTxBlockTimeDiff":313243.74}`.
   `1790366138262` ms = **2026-09-25 19:55:38.262Z**. The diff of 313,244 s is about 3 d 15 h.
   Six more requests between 10:56:30Z and 10:56:37Z all returned 503 with the same `acceptedTxBlockTime`. They hit three different backend
   kaspads (p2pId 965d43fe v2.0.1, b079c555 v2.1.0, 28916cd5 v2.0.0), so all of them read the same frozen DB.
2. **A plain `/info/health` request (no query string) returns a misleading 200.** At 10:55:33Z and 10:56:23Z it returned `HTTP 200` with
   `cf-cache-status: STALE`, `isSynced:true` and `acceptedTxBlockTime 1790365505207` (= **2026-09-25 19:45:05Z**), with kaspad blueScore 568825293.
   Today's live blueScore is 571964429, about 3.14M higher (roughly 3.6 days at 10 BPS). So this is an old Cloudflare copy from about 19:45–19:49Z on 25 Sep.
   That explains the two claimed freeze times: **19:45:05Z is the stale cached health response and 19:55:38Z is the real freeze point.**
3. **Block ingestion is not frozen.** The uncached health shows `database.blueScore` only 10–43 behind the node
   (e.g. 571964493 vs 571964524). `/info/blockdag`, `/info/virtual-chain-blue-score` (blueScore 571964372 at 10:56:17Z) and
   `/blocks-from-bluescore?blueScore=571963000` return **200** with current data. Only accepted-tx / virtual-chain processing is stuck.
4. **Our load at the freeze.** Our node n0 logged at 2026-09-25 21:55:27/21:55:37 CEST (19:55:27/37Z)
   `Tx throughput stats: 6280.48 u-tps` / `5937.89 u-tps` with 61–74k tx/10 s coming in via RPC (`evidence/node-n0-log-...txt`).
   5-min average net TPS was 6,261 for 21:50–21:55 CEST and 4,796 for 21:55–22:00 CEST (`logs/tps12h/nettps.jsonl`).
   Storm fee at that time was FEE_MULT 1.2 (×100 sompi/g), before any 2x/6x/10x/150x escalation.
5. **Node logs cover the freeze but show no reorg.** `rusty-kaspa.log.1.gz` covers 2026-09-24 17:56 to 2026-09-29 03:47 CEST.
   The 26 Sep wipe did not remove this log. Searching it and the current logs for `reorg|reorganiz` finds **0** matches, and there are 0 `virtual` lines on 25 Sep 21:00–22:59 CEST.
   At the exact second (21:55:38.537 CEST) the only non-routine line is `Orphaned 1 block(s) ...1dade3ad... / Unorphaned block 1dade3ad...`.
   That is routine: there are 42 such orphan events between 21:40 and 22:10 CEST.
   Note: default kaspad INFO logging may not print virtual-chain changes, so "no reorg logged" does **not** prove no reorg happened.
6. At 21:56:46 CEST (68 s after the freeze), blocks seen by n0 shift to high storage mass (`mass_storage` 176k–395k vs ~10k before)
   and net TPS drops to 1.4–3k in some 10 s windows. Cause **unproven**: it could be our probe/storm transactions, and it happened after the freeze, not before.
7. The overload probe seeded 3,584 UTXOs "from public API" at 21:47 CEST / 19:47Z, 8 min before the freeze (HANDOFF.md l.23). Our scripts also call `api-tn10.kaspa.org` (e.g. `rebuild-storm-state.py`, `recover-utxos.py`, which last changed 25 Sep 20:51/21:00 CEST, plus later r5–r8 sweepers).
   We don't know if our API usage around 19:55Z mattered (**unproven**).

## Claims
| Claim | Status |
|---|---|
| api-tn10 froze at about 25 Sep 19:55:38Z | **Verified** for the accepted-tx pointer (`acceptedTxBlockTime` = 19:55:38.262Z). Block ingestion keeps running. |
| Another bot saw `acceptedTxBlockTime` frozen at 19:45:05Z | **Explained**: that value comes from the stale Cloudflare copy of `/info/health`. It is not the real freeze point. |
| Returns HTTP 503 (as of 28 Sep) | **Verified still true on 29 Sep 10:56Z** for uncached `/info/health`. The cached path shows 200 STALE. Other endpoints return 200. |
| Our stress test broke the indexer | **Unproven.** It lines up in time (storm ≈6k TPS since 19:22Z, probe since 19:48:41Z), but we have no indexer logs showing a cause. |
| kascov: 907-block TN10 reorg on 17 Sep (x.com/0xKnitser/status/2103505094532866269) | **Unverified.** WebFetch of x.com returned 403. The public mirror api.fxtwitter.com shows the post (25 Sep 15:21:10Z): "A 907-block one on TN10 took us down Sep 17: the watchdog restarted a 19-min rollback 44 times. that's fixed now". This is a self-report. Our node logs start 24 Sep, so we can't check 17 Sep. It predates our test either way. |

## Sources
- Live API: `https://api-tn10.kaspa.org/info/health`, `/info/blockdag`, `/info/virtual-chain-blue-score`, `/info/network`,
  `/blocks-from-bluescore`, queried 2026-09-29 10:55–10:57Z. Raw output in `evidence/api-health-*.txt`.
- Node n0 log `/tmp/kaspa-logs-tn10-n0/rusty-kaspa.log.1.gz` (box), excerpt in `evidence/node-n0-log-2026-09-25T2155CEST.txt`, coverage in `evidence/node-log-coverage.txt`.
- Stress-test workspace `tn10-break-test-2026-09-25/`: `HANDOFF.md` (excerpt `evidence/handoff-fee-timeline-excerpt.txt`, times in CEST),
  `logs/tps12h/nettps.jsonl`, `logs/tps12h/supervisor.jsonl`, `scripts/tps-supervisor.py.bak-2245`.
- kascov post: https://x.com/0xKnitser/status/2103505094532866269 (read through https://api.fxtwitter.com/0xKnitser/status/2103505094532866269), saved in `evidence/kascov-x-post.txt`.

## Conclusion
- **Proven:** api-tn10's indexer DB stopped advancing `acceptedTxBlockTime` at 2026-09-25 19:55:38Z (21:55:38 CEST).
  Uncached `/info/health` still returns 503 on 29 Sep. Block ingestion and node-backed endpoints are live.
  The "19:45:05Z" value is a stale Cloudflare cache.
- **Proven:** our storm was running at about 6k TPS (fee 1.2x) when it happened. Fee escalation (2x → 10x → 6x → 150x → 2x node estimate) all came *after* the freeze.
- **Unproven:** that our test caused it. **Unproven:** any reorg on 25 Sep (none is logged on n0, but INFO logs may not show one).
  **Unverified:** kascov's 17 Sep 907-block reorg. To settle the cause we would need the indexer operator's logs from about 19:55Z.
  Time correlation is not causation.
