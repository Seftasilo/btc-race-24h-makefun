# cbBTC 1-minute trading bot — make.fun bounty entry

Bounty: **"Race a BTC trading bot live for 24 hours"**
(`btc-race-24h-20261112-1439`, $250 total: 1st $60, 2nd $40, 3rd $30,
4th $25, 5th $20, 6th–10th $15 each)
— https://make.fun/bounty/btc-race-24h-20261112-1439

- **Deadline:** Thu 2026-11-12 20:39 UTC = Fri 2026-11-13 03:39 WIB.
- **Race:** from 15 minutes after the deadline, qualifying bots trade the
  next 24 hours live, bar by bar. Entries were open for 6 hours.
- **Status:** I/O fields **verified against the official kit** and the bot
  rewritten to the exact protocol; local dev grade **+$3.14 — dev gate
  PASSED**. **Not yet submitted** — no GitHub repo created, nothing sent
  to make.fun.

## What the bot does

A long/flat-only spot bot for Coinbase Wrapped BTC (cbBTC), paper money
($1,000 start), trading 1-minute bars. One trend ride per window:

- **Entry:** EMA(8) crosses above EMA(21) — the kit's own ema-cross
  baseline entry (freqtrade AverageStrategy rules). EMAs are seeded as
  the mean of their first span of closes, exactly like the baseline, so
  the bot can trade from bar 21. Full equity ($1,000) in one position.
- **Exit (sell everything):** take-profit +1.50%, trailing stop 0.30%
  below the peak once +0.40% in profit, or a 480-bar (8 h) time stop.
  No fixed stop-loss and no cross-down exit.
- **One round trip per window, then stop.** Every strategy family tested
  on the dev window loses on its 2nd+ trips (see "Why one trip" below);
  the one-trip cap is a hard trade limit, the same kind of risk control
  as a daily loss limit.
- Deterministic, standard library only (via the kit's vendored shim),
  no network, no wall clock, no files, no randomness. Two runs on the
  same input give byte-identical replies.

## I/O protocol — verified against the official kit

Kit source (downloaded 2026-11-13 ~01:45 WIB from the official endpoint
only): `GET /api/competitions/btc-race-24h-20261112-1439/kit`
(bundle sha256 `3a2c0374c937d7fa2d6917e8469855866ec923633bf64552fe1a9783802bcb6d`,
5,849,659 bytes). Every file hash in `kit.json` verified OK (27 files)
before use. Kit safety-inspected first: no mnemonic/key exfiltration in
the CLI; the CLI was not run for submission.

Exact fields (protocol `agent-fun/bars-replay/v1`):

| message | fields |
|---|---|
| `hello` (once) | `{"type":"hello","protocol":"agent-fun/bars-replay/v1","seed":str,"symbols":[{"symbol":"CBBTC","mint":"cbbtcf3aa214zXHbiAZQwf4122FBYbraNdFqgw4iMij","price_scale":5,"qty_scale":8}],"capital":{"quote":"USDC","quote_scale":6,"quote_units":"1000000000"},"bar_interval":"1m","first_close_ms":...,"caps":{"max_open_positions":1,"max_orders_per_bar":16,"max_weight_bps":10000,"shorts":false,"leverage":false},"fees":{"fee_bps":4,"slippage_bps":2,"min_notional_quote_units":"1000000"},"latency_bars":1}` — no `warmup_bars` for this competition (defaults to 0). Source: `protocol.schema.json`, `competition.yaml`, `sdk/makefun_bars.py`. |
| `batch` (per minute) | `{"type":"batch","seq":int,"t":ms,"bars":[{"symbol":"CBBTC","t":ms,"o":str,"h":str,"l":str,"c":str,"v":str\|null,"o_depth":[str],"c_depth":[str]}],"fills":[...],"account":{"cash_quote_units":str,"positions":[{"symbol","qty","cost_quote_units"}],"equity_quote_units":str}}`. Bar prices are **integer decimal strings** at price_scale 5 ($72,000 → `"7200000000"`); qty at qty_scale 8; quote amounts at quote_scale 6 ($1 → `"1000000"`). `v` is null. A minute with no sample has **no bar** (CBBTC absent from that batch). |
| `reply` (exactly one per batch) | `{"type":"reply","seq":<same>,"orders":[...]}` — hold = `[]`. Buy: `{"side":"buy","symbol":"CBBTC","notional_quote_units":<int>,"max_slippage_bps":100}`. Sell: `{"side":"sell","symbol":"CBBTC","qty":<int>,"max_slippage_bps":100}`. `client_id` is omitted (the shim assigns `o1, o2, ...`). No `action`/`amount`/`hold` keys; `additionalProperties: false`. |
| `done` | `{"type":"done","seq":N,"reason":"end_of_data"\|"aborted","account":{...}}`. |

Fill model (from `harness/harness-src.tar.gz`, `sim/bars/engine.ts`):
orders decided on batch k fill at bar k+1's open — fee 4 bps +
slippage 2 bps + constant-product impact vs that bar's `o_depth`.
A full round trip costs ~26 bps at $1,000 size. `max_slippage_bps`
is measured from the latest close the bot was sent; a sell's proceeds
never fund a buy of the same reply; orders decided on the last batch
are rejected (`after_end`); everything still held at the end is
flattened at the last close (no max_slippage, no min notional).
Decision checks run in this order: max_orders_per_bar (16) →
duplicate_client_id → not_in_universe → zero_amount → warmup →
after_end → no_bar → buy: insufficient_cash / below_min_notional ($1)
 / max_weight / max_open_positions; sell: insufficient_position /
below_min_notional (unless emptying the position).
Grader runs the bot as `python3 -P -s /bot/main.py` with
`PYTHONHASHSEED=0`; `-P` leaves the script dir off `sys.path`, so the
vendored SDK is imported via a `__file__`-based path insert. Sandbox:
no network, read-only `/bot`, 256 MB `/tmp`, 1 CPU, 512 MB, 1,800 s.

Dev gate: *"only a final entry whose latest dev grade made a net PnL
above 0 USDC after every fee races"*. Race gate: *"at least 1 trade
and a maximum drawdown of 25% or less"*. Deliverable: *"A public
GitHub repository owned by your own GitHub account, created after the
competition opens, under an OSI-approved license, with your bot at
main.ts (Node 22) or main.py (Python 3.13) in the repository root"*
+ the AI coding-agent transcripts that built it. Max 10 submissions;
the last one before the deadline is final.

## Local dev grade — real dev window, exact engine port

`python3 tests/grade_local.py` (see "Files"): a faithful port of
`sim/bars/engine.ts` (BarsPool, barsBuyFill/barsSellFill, decision
checks in order, fill resolution, flatten at last close), feeding the
real dev dataset (`dataset/bars/CBBTC.ndjson.gz`, 1,439 bars over the
1,440-minute window, 1 gap minute; first close $73,739.38, last
$72,044.01, −2.30%), spawning the bot as `python3 -P -s main.py` with
`PYTHONHASHSEED=0`, and running it twice for byte-identical
determinism. Sanity check: buy-and-hold through the same grader =
−$25.97, matching a hand computation (~−$25.6).

| | net PnL | trades | round trips (won) | fees | max DD | rejections | dev gate |
|---|---|---|---|---|---|---|---|
| **final `main.py`** | **+$3.14** | 2 | 1 (1) | $0.80 | 0.48% | 0 | **PASS** |

The one trade: buy decided bar 34, filled $73,693.65 (full $1,000);
sell decided bar 60, filled $73,984.34 (trailing stop). Replies
byte-identical across 2 runs.

**Honest reading:** the margin is thin and the win is a single trade —
on the unseen race window this bot can lose. What the grade proves is
that it races (dev gate passed, 1 trade, 0.48% max drawdown, far under
the 25% race limit), not that it will profit there.

## Why one trip — what was tested and rejected

All graded on the same real dev window with the same grader
(~50 variants; the pattern is consistent, not a single bad draw):

| strategy | net PnL | trips (won) |
|---|---|---|
| Dip-bounce + momentum, 50% size, SL −0.20% (original bot) | −$10.33 | 8 (1) |
| Dip-bounce, deeper dip 1.0% + SL −0.40% (best of that family) | −$3.74 | 5 (2) |
| Momentum ignition, lower thresholds | −$19.72 to −$22.32 | 15–18 (1–2) |
| Mean reversion to SMA120 | −$44.74 | 40 (3) |
| EMA8/EMA21 cross, kit's own ema-cross baseline (full size) | −$96.02 | 34 (1) |
| Ride-the-trend (EMA12/EMA60 + ret60) | −$10.82 | 9 (1) |
| EMA cross entry + trail/TP exits, unlimited trips | −$40.83 | 5 (1) |
| EMA cross entry + trail/TP exits, max 2 trips | +$0.75 | 2 (1) |
| **EMA cross entry + trail/TP exits, max 1 trip (final)** | **+$3.14** | **1 (1)** |

The dev window is one slow rise (+1.29% over 7 h) then a 16 h grind
down, with 1-min moves ≤0.47% (typical ~0.07%). A −0.20% stop is hit
by noise; cross-based exits whipsaw; every re-entry pays ~26 bps of
costs against ~0.1%/h drift. The first ride banks the only clean move.
The public dev board's top entries (+$13.5, +$8.74, +$4.27) are
consistent with the same shape — selective entries, trailing exits —
fitted much harder to this published window than this bot is.

## Files

| file | what it is |
|---|---|
| `main.py` | The bot (Python 3.13-compatible, stdlib only + vendored kit shim). Run: `python3 -P -s main.py` (stdin → stdout). |
| `vendor/makefun_bars.py` | Byte-identical copy of the kit's `sdk/makefun_bars.py` (kit instructs this; excluded from authorship counts). |
| `LICENSE` | MIT (OSI-approved). Copyright holder is listed as `member-3` — confirm it matches the GitHub account owner at repo creation. |
| `README.md` | This file. |
| `tests/grade_local.py` | Exact local grader (engine port + determinism check). Run: `python3 tests/grade_local.py [--trades]`. |
| `tests/backtest.py` | Earlier synthetic-data replay harness (next-open fills, 6 bps/side, no price impact) + edge-case + determinism checks. Superseded for grading by `grade_local.py`. |
| `tests/gen_data.py` | Deterministic synthetic-bar generator (fixed seeds). |
| `tests/data/*.jsonl` | Generated synthetic bars (1440 bars = 24 h each). |
| `tests/BACKTEST.md` | Earlier synthetic results and caveats. |
| `tests/variants/` | Swept variants (parameter + structural), each graded on the real dev window; see the table above. |

## Still needed to submit (not done)

1. **GitHub repo:** public repo owned by the user's GitHub account —
   the account that logs in via Google **seftasilol@gmail.com**
   (note: different from the make.fun Google login, seftazii5@gmail.com,
   handle @member-3). Repo must be created **after** the competition
   opened (2026-11-12 14:39 UTC), contain `main.py` at the root plus
   `LICENSE` (+ this README), under an OSI-approved license.
2. **Transcripts:** the AI coding-agent session transcripts that built
   the bot (this session + any subagent sessions), bundled as make.fun
   requires, uploaded with the submission. Redact personal details only;
   do not redact code, commands, or reasoning.
3. **Submit:** make.fun bounty page → "Submit an entry" → repo URL +
   commit SHA + build transcripts. Max 10 submissions; the last one
   before the deadline is final. Deadline **Fri 2026-11-13 03:39 WIB**.
4. After submitting, poll the entry status; fix any rejected checks and
   resubmit before the deadline if there is time.
