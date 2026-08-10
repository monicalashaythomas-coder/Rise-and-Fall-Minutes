# RISEFALL bot — Railway deployment

This folder is one Railway service: the always-on live trading bot
(`risefall_bot_v4_hmm_gbm.py`). It runs continuously (background worker,
no HTTP port needed) and never exits on its own — Railway's restart policy
(`railway.json`) brings it back up if it ever crashes, and its own internal
watchdog re-execs the process in place if it goes idle for 5+ minutes.

`risefall_lstm_model.py` **must** ship in this same folder — it's the
shared model-architecture module the bot imports directly (`RiseFallWinClassifier`,
`lstm_duration_scan`, etc.) to run Gate 6, the LSTM ensemble second-opinion
check described below. It has to be byte-for-byte the same class definition
the trainer service used, or a state_dict trained by one won't load in the
other.

## Files

| File | Purpose |
|---|---|
| `risefall_bot_v4_hmm_gbm.py` | The bot. Entry point. |
| `risefall_lstm_model.py` | Shared LSTM architecture — same file as in the trainer repo. |
| `requirements.txt` | Python deps (CPU-only torch). |
| `.python-version` | Pins Python 3.11 for the Railpack build. |
| `railway.json` | Start command + restart policy. |
| `.env.example` | Every environment variable this service reads. |

## One-time Supabase setup

Run this **once**, in the Supabase SQL editor, before your first deploy —
covers both the bot's own persisted state and the LSTM table the trainer
service (deployed separately) writes into:

```sql
CREATE TABLE IF NOT EXISTS bot_trade_log (
    id          BIGSERIAL PRIMARY KEY,
    ts          TIMESTAMPTZ DEFAULT now(),
    symbol      TEXT,
    direction   INTEGER,
    step        INTEGER,
    stake       REAL,
    won         BOOLEAN,
    profit      REAL,
    p_up        REAL,
    confidence  REAL,
    duration    INTEGER,
    layer_votes JSONB,
    n_agree     INTEGER,
    n_disagree  INTEGER
);

CREATE TABLE IF NOT EXISTS bot_symbol_state (
    symbol         TEXT PRIMARY KEY,
    reliability    REAL,
    threshold      REAL,
    step0_wins     INTEGER DEFAULT 0,
    step0_total    INTEGER DEFAULT 0,
    layer_weights  JSONB  DEFAULT '{}',
    payout_history JSONB  DEFAULT '[]',
    updated_at     TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS bot_global_state (
    key        TEXT PRIMARY KEY,
    value      JSONB,
    updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS bot_gate_config (
    key        TEXT PRIMARY KEY,
    value      REAL,
    updated_at TIMESTAMPTZ DEFAULT now()
);

-- v7: written by the risefall-trainer service, read-only from the bot.
CREATE TABLE IF NOT EXISTS bot_risefall_lstm_model (
    key                TEXT PRIMARY KEY,   -- "current_tick" / "current_minute"
    kind               TEXT,
    state_dict_b64     TEXT,
    window_size        INTEGER,
    hidden_size        INTEGER,
    num_layers         INTEGER,
    n_heads            INTEGER,
    trained_at         TIMESTAMPTZ,
    symbol             TEXT,
    n_ticks_used       INTEGER,
    n_train_examples   INTEGER,
    n_val_examples     INTEGER,
    val_loss           REAL,
    val_accuracy       REAL,
    baseline_comparison JSONB,   -- persistence/AR(1)/GBM/GRU/CNN comparison
                                  -- report, see risefall_lstm_train.py
    updated_at         TIMESTAMPTZ DEFAULT now()
);

-- v8: written AND read by risefall-trainer only (the bot never touches
-- this table) -- persistent minute-bar archive that works around Deriv's
-- ~24h ticks_history retention limit. Every minute-mode cron run upserts
-- its freshly resampled bars here (deduped on symbol+epoch) and trains on
-- the full accumulated history instead of just that run's fresh fetch.
-- See risefall-trainer/README.md.
CREATE TABLE IF NOT EXISTS bot_risefall_minute_bars (
    symbol TEXT NOT NULL,
    epoch  BIGINT NOT NULL,
    price  REAL NOT NULL,
    PRIMARY KEY (symbol, epoch)
);
CREATE INDEX IF NOT EXISTS idx_risefall_minute_bars_symbol_epoch
    ON bot_risefall_minute_bars (symbol, epoch);
```

## Deploy

1. Push this folder as its own GitHub repo (or point Railway at a
   subfolder) — the root Railway builds from must contain
   `requirements.txt`, `railway.json`, and both `.py` files.
2. Railway → New Project → Deploy from GitHub repo → select it.
3. Add every variable from `.env.example` under Service → Variables.
   At minimum: `DERIV_APP_ID`, `DERIV_API_TOKEN`, `SUPABASE_URL`,
   `SUPABASE_KEY`. Leave `DERIV_ACCOUNT_TYPE=demo` until you've watched it
   trade successfully on demo.
4. Deploy. Watch the logs — it bootstraps ~10k ticks per symbol, runs a
   full deep calibration pass, then starts scanning. First trades usually
   land within a few minutes once calibration completes.
5. Also deploy the `risefall-trainer` service (separate Railway service,
   its own repo/folder) so `bot_risefall_lstm_model` actually gets
   populated — until then Gate 6 just logs "unavailable" and the bot
   trades exactly as it did before the LSTM was wired in (`LSTM_ENABLED`
   can also be set to `false` to disable Gate 6 outright).

## v9: the entire signal stack now runs on minutes, not just the LSTM

Previously, only Gate 6 (the trained LSTM) actually understood minute-bar
data — the ~16-18 layer intelligence stack (Markov, HMM regime, Hawkes,
OU, Hurst, ARFIMA, Kalman, copula, vol_trust, entropy, etc., all feeding
`bayesian_fusion`) and the Monte Carlo (`hmm_gbm_scan`,
`monte_carlo_duration`) only ever computed on raw tick data. Even with
"minute-first" candidate selection, whenever the LSTM didn't clear its
own bar, the fallback pipeline that fired was 100% tick-native — there
was no minute-scale version of it to fall back *into*.

That's fixed now. `MinuteBarView` is a thin adapter that presents a
symbol's resampled minute bars through the exact same interface
`SymbolData` itself exposes (`.symbol`, `.prices()`, `.epochs()`,
`.returns()`, `.mean_tick_dt()`). Every function in the tick-gate
pipeline was audited (see conversation history) and confirmed to only
touch its input data through that interface, or — for `entropy_gate_passes`,
`multi_timeframe_confluence`, `hmm_gbm_scan`, `monte_carlo_duration`,
`meta_ensemble_agrees` — to not take a `SymbolData`-like object at all,
just plain `prices`/`returns` arrays. That means the entire existing,
tested analytical stack runs unmodified against minute-bar data, just by
feeding it through this adapter instead of ticks — no rewrite of the
actual statistics.

**New per-cycle priority order** (both the normal scan loop and the
martingale recovery path):

1. **`try_minute_gates_candidate()`** — the full Gates 1-6 + Monte Carlo
   pipeline, running on minute bars via `MinuteBarView` and a parallel
   `state.minute_model_cache` (HMM/GARCH/OU/Hawkes fit on minute bars,
   via `fit_minute_models_for_symbol()`). If this qualifies, that trade
   fires directly — **nothing else below it even runs** this cycle.
2. **LSTM standalone** (`lstm_evaluate`, unchanged from v8) — only
   reached if step 1 didn't qualify.
3. **Tick-gate pipeline** (the original, all-ticks Gates 1-6) — the true
   fallback now, only reached if neither of the above produced anything.

**What this does NOT include (scoped down deliberately, see below)**:
the tick path's adaptive per-symbol thresholds and walk-forward-learned
per-layer weights come from `expanding_window_walk_forward()` — a
multi-fold backtesting routine that's expensive and needs a lot of
history. The minute path's models are fit directly (no walk-forward OOS
validation, no learned layer weights) and use `per_layer_weights=None`
(static defaults) plus the SAME `state.adaptive_threshold` /
`state.per_symbol_threshold` / `state.reliability` tracking the tick path
already maintains per symbol — shared, not duplicated, since both paths
trade the same underlying symbol just resampled differently. If the
minute path's win rate ends up systematically different from the tick
path's, those shared adaptive mechanisms will still drift toward
whatever's actually working, just without the head start a full
walk-forward fit would give it. Building a full parallel walk-forward
validator for minute bars would be a further, separate undertaking — say
the word if you want that too.

**A real practical constraint worth knowing**: `MinuteBarView` resamples
from the bot's own in-memory tick buffer (`SymbolData`, `maxlen`-bounded),
not from `risefall-trainer`'s persistent Supabase minute-bar archive. For
a 1HZ symbol at the default buffer size that's roughly ~200 minutes of
history — workable for Gates 1-5's shorter internal windows and for
fitting HMM/GARCH/OU/Hawkes, but thinner than what the trainer
accumulates over days via its archive. If richer live minute history
turns out to matter, the same archive pattern could be added to the bot
too.

## v10: minutes only — no tick fallback anywhere, and two real MC bugs fixed

**No more tick trades, ever, by design.** Every remaining tick-fallback
path has been removed, not just deprioritized:
- `try_tick_candidate()` and `try_tick_recovery_candidate()` (the old
  tick-based Gates 1-6 pipelines) are deleted entirely, not just
  unreachable dead code.
- The standalone LSTM path only ever considers a candidate if its own
  pick is minute-duration (`duration_unit == "m"`) — a tick-duration LSTM
  read is no longer a fallback, it's simply not a qualifying candidate.
- `execute_single_step()`'s old behavior of silently downgrading to a
  tick contract when Deriv rejected `duration_unit="m"` is gone. A
  rejected minute contract now fails that trade attempt cleanly (logged,
  no contract placed) instead of substituting a tick trade.
- If nothing confidently qualifies in minutes on a given cycle — normal
  scan or recovery — the bot **waits**. It does not trade ticks as a
  last resort. "No trade" is always an acceptable outcome; "wrong
  contract type" isn't, per explicit instruction.
- Because the tick pipeline was also the only thing feeding
  `record_gate_vote()`/`maybe_recalibrate_gate()` (the system that adapts
  `MIN_LAYER_AGREE`/`MAX_LAYER_DISAGREE` over time), that call moved into
  `try_minute_gates_candidate()`'s Gate 1 check — otherwise those
  thresholds would have frozen at whatever they were the moment tick
  trading stopped and never adapted again.

**Two real, verified duration-selection bugs fixed in the Monte Carlo
layer** — both mechanically biased duration selection toward the
*longest* candidate duration regardless of whether longer durations were
actually more predictable, which is exactly the kind of thing that
quietly erodes an account over time:

1. **`monte_carlo_duration()`'s parametric half** projected a drift
   *point estimate* (`np.mean` of the last 50 returns) forward by
   `dur`, but only scaled *diffusion* noise by `sqrt(dur)` — never the
   drift estimate's own uncertainty. Since the drift term grows as
   `O(dur)` while diffusion-only noise grows as `O(sqrt(dur))`, *any*
   nonzero drift estimate — including pure sampling noise with zero real
   predictive content — mechanically produces increasingly extreme
   confidence at longer durations. On top of that, `drift` was computed
   as `direction * abs(mean(returns))` — taking the *absolute value* of
   recent momentum and reapplying it in whatever direction was already
   chosen upstream, which forces `E[drift] > 0` in the trade's favor even
   on pure noise (by the folded-normal expectation), and incidentally
   defeats Gate 5's purpose as a check *independent* of the layer stack's
   own direction. Fixed both: drift is now the signed mean (a genuine
   headwind shows up as a headwind), and the terminal distribution's
   standard deviation now combines diffusion noise with drift-estimation
   uncertainty in quadrature (`sqrt((dur·SE)² + (vol·√dur)²)`), so
   confidence in a noisy estimate no longer runs away as duration grows.
2. **`hmm_gbm_terminal_log_returns()`** (the HMM regime-conditional half)
   had the identical structural issue: projecting a *fitted* HMM state
   mean forward by `dur` without accounting for that state mean's own
   estimation uncertainty. Same fix, same quadrature combination.
3. **`hmm_gbm_scan()`'s block-bootstrap half** had a *different* source
   of the same underlying problem: summing `dur` block-resampled returns
   from a fixed historical window mechanically amplifies whatever that
   window's own realized sample mean happens to be — not a genuine edge,
   just that specific finite sample's noise — by a factor of `dur`.
   Fixed by demeaning the resampling pool once before drawing blocks,
   which removes the window-level mean artifact while fully preserving
   the local autocorrelation/clustering structure block-bootstrapping
   exists to capture.

**Verified empirically, not just by code review** — reran each function
hundreds of times against pure noise (genuinely no real edge at any
duration) and checked whether the choice/estimate trended toward the
longest duration:
- `monte_carlo_duration`: mean win-rate estimate across durations
  1/2/3/5/10 went from a clear +0.093 upward trend (duration 10 vs
  duration 1) before the fix to +0.015 after — well within one standard
  error of the sampling noise (std ≈0.17 at n=60 trials).
- `hmm_gbm_scan`: which duration got picked as "best" across 600
  pure-noise trials went from duration=10 winning 33.5% of the time
  (fair share is 20%) to 18.5–20.8% across all five durations — flat,
  no detectable bias.

## What Gate 6 (the LSTM ensemble) actually changes

- **v10: minutes only.** Every scan cycle: `try_minute_gates_candidate()`
  (the full Gates 1-6 + Monte Carlo pipeline, minute-native) runs first.
  If it qualifies, that trade fires directly. If not, the standalone LSTM
  gets one more shot (`LSTM_MIN_EDGE_STANDALONE`, default 0.12) — but
  only counts if its own pick is minute-duration; a tick-duration LSTM
  read is not a fallback anymore, it simply isn't a qualifying candidate.
  If neither qualifies, the bot **waits** — it does not trade ticks under
  any circumstance, including if Deriv rejects a minute-duration buy
  request (that fails the trade attempt cleanly instead of downgrading).
  This applies identically to the martingale recovery path.
- Within `lstm_duration_scan()` itself (`risefall_lstm_model.py`), the
  ensemble still internally prefers its minute model over its tick model
  whenever both have a usable read — relevant mainly for reporting/
  diagnostics now, since a tick-duration result from that function is
  filtered out before it can become a trade regardless.
- Gate 6 is still a **hard veto** inside `try_minute_gates_candidate()` —
  if the LSTM ensemble disagrees with the minute layer stack's direction
  there, the trade is skipped. This is deliberate: the LSTM is the model
  `risefall-trainer` actually optimizes and re-uploads every cron cycle
  (unlike its five comparison baselines — persistence, AR(1)/Hurst, a
  GBM, a GRU, a dilated CNN — which stay diagnostic-only and never
  influence a live trade), so it gets real veto power to match.
  **Practical consequence**: this can meaningfully cut trade frequency,
  and the quality of your trades now depends on the LSTM actually being
  good, not just present. Consider leaving `LSTM_ENABLED=false` until
  `risefall-trainer`'s `baseline_comparison` logs show the LSTM reliably
  beating its persistence baseline (and ideally the richer AR(1)/GBM/GRU/
  CNN ones too) over a few cron cycles, then flip it on.
- Until the trainer has completed at least one successful run, the LSTM
  model is simply absent and the standalone LSTM path is a permanent
  no-op — the minute-native Gates 1-6 pipeline (which doesn't depend on
  the LSTM at all, only on `state.minute_model_cache`'s HMM/GARCH/OU/
  Hawkes fits) is what carries the bot in that case, still minutes-only.
- The trainer trains **one shared model pooled across a whole basket of
  symbols** (see its README), not just `1HZ10V` — normalization is
  per-window/local (`local_normalize()` in `risefall_lstm_model.py`), so
  the same model applies sanely to any symbol regardless of that symbol's
  native volatility scale, including ones outside the trainer's basket.
- Trade summaries (`emit_sequence_summary`/`log_trade_summary`) correctly
  report `minutes` — previously this was hardcoded to always print
  "ticks" regardless of `duration_unit`.

## Two other real bugs fixed in this pass

**1. Crash loop on every drift-triggered recalibration.**
`check_calibration_triggers()` returns `("drift", [list of flagged
symbols])`, but `run_calibration()`'s startup print assumed the second
element was always a plain string (`':' + loss_symbol`), which raised
`TypeError: can only concatenate str (not "list") to str` on every single
drift event. The bot's watchdog catches this and restarts the process in
place, so it wasn't fatal, but it meant the bot could spend most of its
time crash-looping through `deep_startup_calibration()` (meant to run
ONCE per process lifetime) instead of ever settling into steady-state
trading. Fixed to handle every shape `trigger_reason`'s second element
can actually take (list, string, or `None`).

**2. Directional bias from `ConfidenceCalibrator`.** This one was a real
find, not something I noticed on my own — full credit for tracing it to
`expanding_window_walk_forward()`'s outcome label. The walk-forward
report was labeling each training example `won = (predicted_dir ==
actual_dir)` — a symmetric "was the prediction correct" label — and
feeding that into a calibrator whose whole job is producing a
*directional* `P(up)` estimate. A confident, genuinely-correct PUT call
and a confident, genuinely-correct CALL call both score `won=1`
identically, so the fitted isotonic table ends up mapping "this
confidence level" to a *win rate*, not to `P(up)` — and `calibrate()`
blends that win rate straight into a probability of the wrong thing.
Reproducing the exact scenario confirmed it's worse than it first looks:
the temperature-scaling stage collapses almost *all* directional signal
(a synthetic model with genuine, symmetric 70% accuracy calibrated a
confident PUT and a confident CALL to nearly the same ~0.63, since a
direction-blind label gives the temperature fit nothing informative to
distinguish them), and the isotonic stage on top of that further dragged
confident PUTs toward CALL specifically. Fixed by changing the label fed
into calibration to "did price actually go up" — symmetric with what
`p_up` itself means — while leaving the separate, legitimately-symmetric
hit-rate/accuracy reporting (`per_duration_outcomes`, `hits_fold`, the
per-fold hit rate you see in calibration logs) untouched, since "how
often is the model right" is a different, correctly-symmetric question
from "what does this confidence level imply about direction".
**This bug would have applied to both tick and minute trading equally**
— it's upstream of the duration/unit decision entirely, in the
directional confidence signal every gate consumes.

## New: Support/Resistance layer (18th layer in the stack)

`compute_support_resistance()` — genuinely new, level-based signal, not
another rolling-statistic indicator like RSI/Bollinger/Z-score above
(which the stack already had). Detects swing highs/lows over a lookback
window, clusters nearby ones into levels weighted by how many times
price has touched near them, then produces a proximity- and strength-
weighted signal from the nearest support/resistance. Ranging mode fades
into a level (near resistance → bearish, near support → bullish);
momentum mode adds a continuation bonus on a genuine breakout beyond all
known levels (the classic "broken resistance becomes support" polarity
flip). Wired into all three places a layer actually needs to reach to
matter: `_layer_votes` (gate agreement counting), `bayesian_fusion`
(actual evidence contributing to `p_up`), and `explain_signal` (trade
logging) — verified directly that all three actually see it, not just
the vote count.

Also worth knowing: `risefall-trainer`'s defaults got meaningfully
deeper this pass — `LSTM_EPOCHS` 15→30, pooled example caps 20k/4k→50k/
10k train/val — now that it's fully minutes-only, there's no tick
training to split cron-window budget with. Expect ~60-90 min per run at
these defaults rather than the previous well-under-20-minutes; lower
`LSTM_MAX_TRAIN_EXAMPLES`/`LSTM_MAX_VAL_EXAMPLES` first if a run risks
overrunning the 5h cron window.

**Layer-agreement thresholds updated for the new layer count.**
`MIN_LAYER_AGREE`/`MAX_LAYER_DISAGREE` (Gate 1) were derived and tuned
against a 16-layer stack (see the "56% supermajority" comment above
their definition). Adding Support/Resistance as a 17th layer shifted
what the same raw integers meant as a fraction of the vote — kept
proportionally equivalent instead: `MIN_LAYER_AGREE` 9→10 (56.25%→58.8%,
close to the original design intent), `MAX_LAYER_DISAGREE` unchanged at
4 (23.5%, close to the original 25%). This is a modest adjustment, not a
return to the 12/16=75% supermajority that caused a documented
near-zero-trade-frequency incident referenced in that same comment block.
**`GATE_SCHEMA_VERSION` was bumped 4→5 alongside this** specifically so
any value already persisted to Supabase from before this change (tuned
against the old 16-layer stack) gets automatically invalidated on next
load instead of silently overriding the new default — `load_gates()`
already had this exact mechanism built in for exactly this situation, it
just needed the version bump to fire. If you're running this on a bot
that's already accumulated live gate history, you don't need to do
anything manually; the schema-mismatch fallback handles it on the next
restart.

## Real gap found and fixed: no floor on confidence magnitude

Traced from a live trade log showing a signal fire at `p(UP)=0.5155,
confidence=0.0099` — essentially a coin flip. Root cause: nothing in the
gate stack checks confidence *magnitude*. Gate 1 (`passes_layer_gate`)
only checks how many of the 17 layers agree on *direction* — a count,
not a strength — so a comfortable majority weakly leaning the same way
can clear it while the fused confidence is indistinguishable from noise.
`MIN_EXP_WIN_RATE` gates a different quantity entirely (the Monte
Carlo's own win-rate estimate).

Added `MIN_CONFIDENCE` (default 0.03, env-overridable) as an explicit
floor, checked right after `fuse_signal()` in both the normal scan loop
and the recovery path's minute-native pipelines — independent of Gate
1's agreement count. Checked empirically against synthetic data before
picking the default: confidence values on this bot commonly sit in the
0.008-0.02 range even on otherwise-qualifying signals, so 0.03 filters
out roughly the weakest 60% (would have blocked the 0.0099 trade above)
while still leaving room to trade on stronger signals, rather than
setting a bar these models can't currently clear at all. Tune up as
model quality improves.

## Root cause found: calibration was starving the event loop, causing repeated watchdog restarts every ~65-70 minutes

Traced from a live log showing `deep_startup_calibration` (the full ~32-min,
all-8-symbols walk-forward validation) re-running every ~65-70 minutes all
night, with almost no trading in between -- and a direct smoking gun in a
follow-up screenshot: `[Watchdog] No activity for 2213s (limit 300s).
Restarting process in place now.`

Root cause: `deep_startup_calibration()` and `run_calibration()` were both
declared `async def` but contained **zero `await` statements anywhere in
their per-symbol loops** — `expanding_window_walk_forward()`,
`walk_forward_validate()`, and `fit_minute_models_for_symbol()` are all
synchronous, CPU-bound calls (numpy/statsmodels/hmmlearn/arch). Since
Python's asyncio event loop is single-threaded and cooperative, a
coroutine that never yields blocks *everything* else for as long as it
runs — including `tick_consumer` (so `state.last_activity` never gets
touched by real tick arrival) and `watchdog`'s own periodic check (so it
doesn't even get a chance to fire *on time* — it only runs once the event
loop is finally free again, by which point idle time is wildly overdue
and it immediately restarts the process). Across 8 symbols at ~4 minutes
each, that's a genuine ~32-minute unbroken block, comfortably explaining
the observed 2213s stall.

Fixed by moving every CPU-bound fitting call in both functions onto a
background thread via `asyncio.to_thread()`, which keeps the event loop
free to service ticks/watchdog/balance throughout — verified directly
with a synthetic reproduction of the exact starvation pattern: the old
direct-call approach left `tick_consumer` completely stalled for the
full duration of a simulated 1-second blocking call (1049ms idle);
`asyncio.to_thread()` left it essentially unaffected (52ms idle, in line
with its own poll interval). Also added a light `await asyncio.sleep(0)`
per symbol in the main scan loop as cheap defense-in-depth, since the
minute-native Gates 1-6 pipeline does real (if much smaller) synchronous
work per symbol per cycle too — though that loop was already bounded by
the outer `await asyncio.sleep(2)` and was never the primary cause here.

**This should stop the repeated full-recalibration cycles entirely** —
`deep_startup_calibration` should now only run once at genuine process
start, and scheduled/drift-triggered `run_calibration` should complete
without starving the watchdog into a false restart.

## Martingale: factor 1.45, max 4 steps, no balance cap (explicit instruction)

Changed per explicit instruction: `MARTINGALE_FACTOR` 1.24→1.45,
`MARTINGALE_MAX_STEPS` 2→4, and **both balance-based guards removed** —
the entry-point check that used to refuse to arm recovery if the next
stake would exceed `MAX_SEQUENCE_LOSS_PCT` of balance, and the
recovery-loop's own "SEQUENCE LOSS GUARD" that used to abort mid-sequence
for the same reason. `MAX_SEQUENCE_LOSS_PCT`/`max_allowed` are still
computed and logged on every step for visibility, they just no longer
block anything. **`MARTINGALE_MAX_STEPS` is now the only thing that
stops a losing sequence** — after step 4 loses, the code's existing
"exhausted all steps" path resets to step 0 and runs a fresh
recalibration, same as before.

Worth being concrete about what this actually risks, since the comment
this replaced documented that unchecked martingale escalation is exactly
what caused a prior account-destruction incident on this bot: at
`MIN_STAKE=0.35`, a full losing sequence (step 0 through step 4) is
`[0.35, 0.51, 0.74, 1.07, 1.55]` — **4.22 total, ~12x base stake**, with
nothing checking that against balance. On a small account this can
exceed the entire balance in one sequence. This is exactly the risk
profile you asked for — flagging the number, not second-guessing the
decision.

## Safety notes

- `DERIV_ACCOUNT_TYPE=real` trades real money. The bot prints a loud
  warning banner on startup if set, but does not block it — that decision
  is left to you.
- Railway's filesystem is ephemeral. Every piece of state this bot needs
  to survive a restart (thresholds, reliability scores, gate config,
  direction history, meta-learner weights) is persisted to Supabase — if
  `SUPABASE_URL`/`SUPABASE_KEY` are unset, the bot still runs but forgets
  everything on every restart.
