# CLAUDE.md

Guidance for working in this repository: **Nyao Scalper v51.0**, a MetaTrader 5 Expert Advisor (MQL5) plus a Python backtest runner.

## Repository layout

| Path | What it is |
|---|---|
| `nyao_scalper.mq5` | The whole EA, one file of about 8,400 lines. No local `.mqh` includes. |
| `settings/default.set` | Default profile. **Must equal the EA's built-in `input` defaults** (README says so, and users rely on it). |
| `settings/high-risk.set`, `settings/usdjpy.set` | Variants of default: 2.5% risk, and USDJPY with every distance scaled. |
| `tools/backtest/nyao_backtest.py` | Runs Strategy Tester passes from the CLI and builds `reports/<name>/`. Windows only, Python 3.8+ stdlib only. |
| `reports/<name>/` | Generated: `summary.md`, `index.html`, `results.csv`, `plan.json`, `graphs/`. `raw/` is git-ignored. |
| `README.md` | User-facing docs for every feature and input. Update it when behaviour or defaults change. |

Only standard MQL5 libraries are used: `<Generic\HashMap.mqh>` and `<Controls\Dialog.mqh|Edit|Button|Label>`. No DLLs. The EA **requires a hedging account** (`InitializeEA` fails on netting).

## Compiling (MetaEditor CLI)

MetaEditor is not installed on this machine. The command below is the one `compile_ea()` in the backtest tool uses:

```powershell
& "C:\Program Files\MetaTrader 5\MetaEditor64.exe" /compile:"<path>\nyao_scalper.mq5" /log:"<path>\nyao_scalper.log" /inc:"%APPDATA%\MetaQuotes\Terminal\<id>\MQL5"
```

- Put the quotes around the value only: `/compile:"C:\path with spaces"`.
- `/inc` must point to a terminal's `MQL5` folder so that `<Generic\...>` and `<Controls\...>` resolve. Compiling a copy under `MQL5\Experts\` works too.
- **Don't trust the exit code.** Read the `.log` (it may be UTF-16) and check that its `Result:` line contains `0 errors` and that a fresh `.ex5` exists. Delete the old `.ex5`/`.log` before you compile.
- `*.ex5` and `*.log` are git-ignored. Don't commit binaries.
- Other ways to compile: F7 in MetaEditor, or the VS Code "MQL5/MQL4" (Buraq) extension (see README).

## Running backtests

```bash
python tools/backtest/nyao_backtest.py run --name default-2026-real-ticks --periods M1,M5 --from 2026-01 --to 2026-09
python tools/backtest/nyao_backtest.py run --name my-test --set settings/my.set --periods M5 --from 2026-03 --to 2026-06 --model ohlc
python tools/backtest/nyao_backtest.py report --name default-2026-real-ticks   # rebuild report from existing tester output
```

What `run` does:
1. Finds `terminal64.exe` under Program Files, then its data folder from `%APPDATA%\MetaQuotes\Terminal\*\origin.txt`.
2. Compiles the EA into `MQL5\Experts\NyaoBacktest\`, so the user's own build is never touched.
3. Writes the `.set` file as a UTF-16 tester profile, `MQL5\Profiles\Tester\nyao_bt_<name>.set`.
4. Writes one `[Tester]` `.ini` per timeframe × month to `%TEMP%\nyao_backtest\<name>\`. Each run gets a fresh deposit, `ShutdownTerminal=1` and local agents only.
5. **Closes MT5 gracefully** (`CloseMainWindow`, never a kill), then runs `terminal64.exe /config:<ini>` for each pass at below-normal priority with a CPU affinity mask.
6. Reopens MT5 and parses each `<report>.htm`. Parsing needs **English terminal labels** ("Total Net Profit:" and so on).

Options: `--split monthly|whole`, `--model real|every|ohlc|open` (default `real`), `--symbol XAUUSDm`, `--deposit 2000`, `--leverage 500`, `--cpu-mask`, `--timeout-min 30`, `--keep-raw`, `--no-reopen`, `--terminal`, `--data-dir`.

Notes:
- **Ask before you run `run`.** It closes the user's MT5 terminal.
- Only real-tick results mean anything for this EA. Position management runs every tick, while entries run once per bar.
- The tester has no news calendar, and state persistence is off in the tester.
- `ea_version()` reads `#property version` for the report title.

## Architecture overview

All logic lives in global functions and global state in one file. The rough order in the file:

| Lines (approx.) | Section |
|---|---|
| 36–80 | Enums (`ENUM_INPUT_TYPE`, `ENUM_SCORE_MODEL`, `ENUM_CHECKPOINT_HOLD`, ...) |
| 82–317 | `input` groups: indicators, score weights, orders, limit entry, dampening, loss management, hedge chain, lot sizing, equity, TP/SL, R:R, checkpoint, trailing, robot |
| 319–538 | Globals; structs `SignalStrength`, `PositionHealth`, `ManagedPosition`, `TradeStats` |
| 572–1021 | `OnInit` → `InitializeEA` (deferred until connected), `OnDeinit` |
| 1026–1246 | `OnTick` → `ProcessTick` |
| 1300–1426 | `OnTradeTransaction`: fill registration and close accounting |
| 1443–1648 | Floating P/L helpers, own-drawdown, `CheckBasketStop` |
| 1653–1966 | Entry pipeline: `CheckForTradingSignal`, `ComputeEntryScore`, `DirectionSignal`, `EntryConditionsBlock`, `HTFTrendAllows` |
| 1972–2015 | `ManagePositions` (management order) |
| 2022–2486 | Scoring: `ComputeRawScore` (legacy), `ReadTrend` / `ComputeTrendScore`, `GetSignalStrength` |
| 2493–3340 | Position health, `ManageLosingPositions`, partial close, virtual-SL re-entry |
| 3347–3660 | `ManageTrailingTPSL` (adaptive trailing, off by default) |
| 3665–4190 | Managed-position registry, sync, loss-streak paper trading, `ProcessClosedPosition`, `OpenPosition` |
| 4212–4980 | Hedge chain (lot sizing, open, graduate, release, reseed, `ManageHedgeChains`, `ProtectGraduatedHedges`) |
| 4998–5230 | Checkpoint exit and trend-flip exit |
| 5285–5610 | Pending limit entries |
| 5604–5785 | Close and modify helpers (`ClosePositionAndAccount`, `CloseAllPositions`, `ModifyPosition`) |
| 5787–6140 | Entry window and gates, spread filter, risk-based lot sizing, volume helpers |
| 6148–6530 | Dynamic lots, break-even price, `PointScale`, `ConvertToPoints`, SL/TP/R:R distances |
| 6532–7280 | News, trading hours, pause state, peak / target / min-equity / drawdown, market close, leverage |
| 7285–7575 | Trade reports and statistics |
| 7641–7985 | Persistence (save, restore, adopt, balance operations) |
| 7997–8113 | Discord alert queue (`WebRequest`) |
| 8119–end | Dashboard |

## Event flow

**`OnInit`**: shows a password dialog if `EA_PASSWORD` is set (empty by default). Outside the tester, if `AccountReady()` is false it sets `initDeferred` and polls with a 1 s `OnTimer`. Otherwise it calls **`InitializeEA`**, which:
1. Validates inputs.
2. Creates indicator handles: fast/slow EMA, RSI, ATR, and the trend EMA/ATR when `ScoreModel=Trend`. Adds the HTF EMA when the HTF filter is on.
3. Resets globals.
4. Scans open positions (magic + symbol) into `managedPositions[]`. A comment starting with `"Hedge L"` marks a former hedge leg (`noRehedge`, `hedgeGraduated`).
5. Calls `RestoreState()`, `AdoptUnrestoredPositions()` and `SaveState(true)`.

**`OnTick`**: runs `ProcessTick()`, then `SaveState()` (writes only if something changed), then `FlushDiscordAlerts()` (at most one webhook per tick).

**`ProcessTick`** steps, in order:
1. Invalidates the per-tick signal cache (`_strengthValid[4]`).
2. If the balance changed, runs `CatchUpBalanceOperations` (deposits and withdrawals shift the baselines). Retries pending adoptions.
3. Runs `CheckAlgoTradingStatus`, `CheckPeakEquity`, `CheckTargetEquity`, `CheckMinTradeableEquity`, `CheckEquityDrawdawn` (sic) and `CheckBasketStop`.
4. **Terminal stop.** If target, min-equity or max-triggers is hit: closes all positions, deletes pending orders, deletes the state file once flat, calls `ExpertRemove()`, then returns.
5. Runs `CheckTradingHours`, `CheckLeverageChange` and `CheckMarketClose`.
6. On a new bar: updates the velocity baselines and the consecutive-trade-candle counters, and resets the per-bar buy/sell counts.
7. Runs `CheckHighImpactNews` (it always runs, so the news window is known).
8. **Blocked** (outside hours, leverage changed, near close): `ManagePositions()` only, plus `MarkEntryBarSeen()`. Returns.
9. **Paused** (`IsPaused()`: separate equity, basket and news timers): manage only. Returns.
10. Resume alert if it was paused.
11. Runs **`ManagePositions()`**, then **`CheckForTradingSignal()`**, then the report and dashboard.

**`ManagePositions()`** runs in this order on every tick. The order matters, because chain legs must be frozen before trailing and loss management see them:
`SyncManagedPositions` → `ManageHedgeChains` → `ProtectGraduatedHedges` → `ManageCheckpoints` → `ManageTrendFlipExits` → `ManagePaperTrade` → `ManageTrailingTPSL` (per position) → `ManageLosingPositions` → `ManagePendingOrders`.

**`OnTradeTransaction`** (`DEAL_ADD` only) is the **primary** close-accounting path:
- Balance and credit deals go to `ApplyBalanceOperation`.
- A `DEAL_ENTRY_IN` that isn't registered yet (a filled limit) is registered and gets `InitCheckpoint`. The entry score comes from the order comment.
- A full close calls `ProcessClosedPosition`. A partial close is ignored, because the position still exists.

`SyncManagedPositions` is the per-tick fallback. `ProcessClosedPosition` is **idempotent**: it returns early if the ticket is no longer managed.

## Logic modules

### Signal scoring
`GetSignalStrength(orderType, entryMode, forceClosedOnly)` returns a `SignalStrength` with `finalScore` from 0 to 10.
- The result is cached per tick in 4 slots: buy, sell, buy-closed-only and sell-closed-only.
- It takes a linear-weighted average of `SignalSmoothingCandles` closed bars (bar 1 has the largest weight). Unless closed-only, it blends in the forming bar with weight `CurrentCandleBlend`.
- Closed-only applies when `entryMode && EnableNewBarEntryOnly`, or when `forceClosedOnly` is set. Checkpoint and trend-flip use `forceClosedOnly`.

`ComputeRawScore(type, idx)` dispatches on `ScoreModel`:
- **`SCORE_TREND`** (the default), via `ReadTrend` / `ComputeTrendScore`:
  - Four parts, each in [-1, +1]: range position, EMA distance in ATRs (tanh), directional efficiency (tanh) and ROC in ATRs (tanh).
  - `s` is their average. Buy = `5+5s`, sell = `5−5s`, so the two always sum to 10.
  - Readings for closed bars are cached in a ring of 16 (`trendCache`).
- **`SCORE_LEGACY`**: trend (EMA alignment + slope, max 3) + momentum (RSI, body and impulse, max 3) + chop (ATR expansion or ER, max 2) + peak breakout + volatility − wick rejection.

`deadMarket` (ATR/AvgATR < `MinVolRatioToTrade`) is an **entry-only** gate. It is deliberately not folded into `finalScore`, because management compares against that score.

Velocity is informational only, for the dashboard.

### Entry gate
1. `CheckForTradingSignal()` runs only when the new-bar gate is open: once per bar, after a normal spread is seen in that bar. A requote or price-change retcode allows up to 3 retries within the bar.
2. `BuySignal()` / `SellSignal()` call `DirectionSignal(isBuy)`, which runs:
   - `EntryConditionsBlock()`. It returns a reason string, or `""` when the direction may enter. Checks, in order:
     - per-candle limit
     - opposite trade on the same bar
     - max losing positions in the same direction
     - loss cooldown
     - range filter (`DirectionalEfficiency`)
     - `HTFTrendAllows`: last closed HTF bar; fails closed when data is missing
     - duplicate-distance filter (`ZonePoints × PointScale × multiplier`)
   - `ComputeEntryScore()`: the closed-bar score minus `LosingPosScorePenalty` per losing same-direction position. The threshold is the min score plus the consecutive-candle boost plus the drawdown boost. With `DD_SCOPE_OWN` the drawdown comes from this EA's own P/L.
   - The dead-market block.
3. The higher qualifying side calls `OpenPosition()`, or `PlaceLimitEntry()` when `EnableLimitEntry` is on.
4. `OpenPosition()` runs `IsAllowedToOpenPosition()`: `GetEntryBlockReason()` (stopped, paused, streak pause, hours, leverage, market close), max holding-loss positions, `MaxOpenOrders`, the order lock and spread. It then computes:
   - the lot: `CalculateDynamicLotSize`, then `ApplyRiskSizing` against the worst-case SL `(1+skips)×SL`
   - the SL and TP distances
   - stop validity
   - the broker SL at the hard cap
   - **no broker TP in checkpoint mode**
   It then sends `OrderSend`, registers the position (`result.order` doubles as the position ticket on hedging accounts) and calls `InitCheckpoint`.
5. **Loss-streak pause**: after `LossStreakPauseCount` losses in a row, signals open *paper* trades instead (`OpenPaperTrade` / `ManagePaperTrade`). The first win, paper or real, resumes real trading.
6. `IsNewRiskAllowed()` is the entry window plus the order lock plus spread. It is shared by hedge-chain starts, rolls and reseeds.
7. The dashboard's `DashboardEntryGate()` re-runs the **same gates in the same order**, with `quietLog=true`. Keep the two in sync.

### Checkpoint exit (on by default)
The TP is never sent to the broker: `cpPrice` is a virtual checkpoint. `ManageCheckpoints()` handles every non-chain leg:
- **A, loss side**: when `CheckpointMaxSkips > 0`, `virtSL` is an EA-managed inner stop. It can be pushed further while `CheckpointScoreAgrees` holds. The broker SL stays at the hard cap.
- **B, checkpoint reached**: hold or close per `CheckpointHoldMode`:
  - `TREND` = `HTFTrendAllows`
  - `SCORE` = own side's closed score > the opposite side's
  - `BOTH`
  On a hold, the SL locks `CheckpointLockPct%` of the move and `cpPrice` advances by `cpStep`. Several steps can roll in one tick.
- **C, trailing**: trails at `CheckpointTrailPct%` of `cpStep` once profit ≥ `CheckpointTrailStartPct%`. It moves in steps of at least 5%.
- **C2, break-even**: `CheckpointBreakEvenPct` puts the SL at `MinProfitLockPrice` (commission + negative swap + 1 tick).
- **D, owed lock**: `cpLock` is retried until the broker SL reaches it. If price falls back to the lock first, the EA closes the trade itself.
- A failed modify sets `cpRetryAfter` to 1 s. Stops only ever move forward.

`ManageTrendFlipExits()` closes a losing non-chain position when the opposite side's closed score is ≥ `TrendFlipExitScore`.

### Hedge chain (off by default; a martingale)
`ManageHedgeChains()`:
- **Phase A**, for each `chainId`: find the older leg (lowest `hedgeLevel`) and the hedge (highest). Then decide:
  - **orphan**: one leg left → `GraduateChainLeg`
  - **stopping**: `CloseChain` until flat
  - **STOP** (checked first): open + booked loss ≤ `−ChainLossStopThreshold()`, the tightest of $, % of equity and ×SL
  - **COVERED**: hedge P/L ≥ `HedgeRecoveryPct%` of the older leg's loss → close the older leg and graduate the hedge. `hedgeLockProfit` becomes the recovery floor.
  - **ROLL**: hedge losing and the older leg recovered → close the older leg and `OpenChainHedge` a bigger reverse leg. Needs level < `HedgeCycleLevels`, a lot that grows, and total lots within `HedgeMaxTotalLots`.
  - **RESEED**: `ReseedCycle` partial-closes the deepest hedge and starts a new cycle.
  - **EXHAUSTED**: `ReleaseChainToLossMgmt` (`noRehedge`, not closed).
- **Phase B**: start a chain on a losing standalone leg. Needs:
  - adverse move ≥ `HedgeTriggerATR×ATR`
  - reverse score ≥ `HedgeMinSignalScore`
  - a hedge lot > the position lot
  - room under the caps and `HedgeMaxActiveChains`
  The root's `ManagedPosition` is backed up and restored if the hedge order fails. The root SL is cleared only after the hedge exists.

Anything that adds exposure obeys `IsNewRiskAllowed`. Closing actions always run.

Active chain legs (`chainId != 0`) are **skipped** by trailing, loss management, checkpoints, trend-flip and the basket stop.

`ProtectGraduatedHedges()` guarantees a stop on legs that left a chain.

### Risk and equity guards
- **Lot**: `CalculateDynamicLotSize` (base lot + optional drawdown steps). Then `ApplyRiskSizing` / `RiskSizedLot`: equity × `RiskPerTradePct` / money lost per lot at the SL. If the minimum lot risks more than that, it is still traded up to `MaxRiskPerTradePct`; above that cap the entry is skipped. Risk sizing applies only to Points/ATR stops (`RiskSizingApplies`).
- **Point inputs**: `PointScale()` rescales values written for `PointInputDigits`-decimal quotes. This covers SL/TP/trailing/RR as Points, `ZonePoints` and `MaxSpreadPoints`. `ConvertToPoints()` handles $, % of equity, Points and ATR.
- **Basket stop** (`CheckBasketStop`):
  - The `MaxTotalLossPct` hard stop counts everything, including chains. It closes all positions and pauses.
  - The `MaxBasketLossPct` stop counts non-chain legs only (`GetBasketFloatingPL`). It closes non-chain positions and pauses.
- **Drawdown pause** (`CheckEquityDrawdawn`): triggers when equity < `lastPeakEquity × MinEquityPercent%`. The pause grows with `PauseMinutesMultiplier`, and `lastPeakEquity` resets to balance. `MaxMinEquityTriggers` turns repeated triggers into a full stop. A new peak lifts the equity pause.
- **Hard stops**: `TargetEquity` and `MinimumEquity` close everything and call `ExpertRemove()`.
- **Pause timers** are independent (`equityPauseUntil`, `basketPauseUntil`, `newsPauseUntil`, all in `TimeTradeServer()` time), so one never shortens another.
- **Other filters**: spread (fixed, or a fraction of ATR), news (`CalendarValueHistory`, throttled), trading hours, market close (`GetCurrentSessionEnd` follows seamless sessions), and leverage change (optionally closes all positions).
- **Account vs own trades**: account-wide checks (min-equity %, `MinimumEquity`, `TargetEquity`, leverage) watch the whole account. Basket, total-loss and per-trade logic use only this symbol + magic.

### Persistence
- State lives in `MQL5\Files\NyaoScalper\state_<login>_<symbol>_<magic>.bin`. It is inactive in the tester and in optimization (`IsPersistenceActive`).
- Format: `PersistHeader` (account guards such as peak equity, streak, cooldown, pauses, paper trade, leverage, `lastBalanceDeal`) followed by a raw `FileWriteArray` of `ManagedPosition[]`.
- `RestoreState` validates `version == STATE_FILE_VERSION`, `recordSize == sizeof(ManagedPosition)`, login and magic. It overlays saved records onto the freshly scanned positions and replays balance operations made while the EA was offline.
- `AdoptUnrestoredPositions` gives uncovered positions a stop and a checkpoint, retried every tick through `adoptPending`.
- `SaveState()` diffs against the last snapshot (header fields + `SameManagedPosition`) and writes only on change.
- `ResetSavedState=true` deletes the file at init. The file is also deleted after an equity stop once flat.

## Conventions and invariants

- **Changing `ManagedPosition`**: update `RegisterManagedPosition` (initialise the field), `SameManagedPosition` (diff) and **bump `STATE_FILE_VERSION`**. Old files are then ignored, not misread.
- **Changing the version**: change `#property version`, `#define EA_TITLE`, the header comment, the `input group " Nyao Scalper vXX"` line and the header line in the `.set` files.
- **Changing an input default**: update `settings/default.set` (and the derived profiles if they apply) and the README tables and text.
- **Logging**: use `LogPrint(...)`. It is gated by `EnableLogging` and muted by `quietLog` while the dashboard evaluates gates. Use plain `Print` only for messages that must always show (`[STATE]`, `[RISK SIZING]`, `[POINTS]`, init errors).
- **Positions are filtered by `MagicNumber` and `_Symbol`** everywhere. Closes made manually or by close-by still count for managed tickets.
- **Closing on the same tick**: if code decides something on the same tick after a close, use `ClosePositionAndAccount()`, which does the accounting immediately. That accounting is idempotent.
- **Prices**: normalise with `NormalizePrice()` (tick-size aware). Check stops with `IsSLValid` / `IsStopDistanceValid` before you modify.
- **Loops**: loops that may close positions iterate **backwards** and re-check `i >= managedPositionCount`.
- **Comment style**: `// +---...---+` box headers for sections. Explanatory comments describe *why*. The code is wrapped at about 120–130 columns.
- **Keep shared gates aligned**: entry gates are shared by fresh entries, re-entries, limit housekeeping, hedge starts and rolls, and the dashboard. Change them in one place (`GetEntryBlockReason`, `EntryConditionsBlock`, `IsNewRiskAllowed`) rather than duplicating them.
- **Validating a change**: a build that compiles is not enough. Run a backtest, ideally with real ticks, and compare against the existing `reports/` baselines.
