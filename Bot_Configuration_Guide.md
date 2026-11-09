# Bot Configuration Guide — Deriv Adaptive Digit Bot

> **Status of this file:** It was checked for well-formed XML and executed against a *mock* Deriv environment (see `Strategy_Explanation.md`, section 8). It has **not** been imported into or run on the real Deriv Bot platform. If the importer rejects a block, nothing here has been verified against that failure — report the error message so it can be fixed.
>
> Nothing in this bot guarantees profit. Use a **demo account** first.

## 1. Import the bot

1. Log in at Deriv and open **Deriv Bot**.
2. Switch to a **demo (virtual) account** using the account switcher at the top right **before** doing anything else. The bot trades whichever account is active when you press Run.
3. Open the **Bot Builder** tab.
4. Click **Import** (folder icon), choose **Local**, select `Deriv_Adaptive_Digit_Bot.xml`, then **Open**.
5. The blocks load into the workspace. If you see an error instead, copy it exactly — it identifies which block the importer disliked.

(Menu labels vary slightly between Deriv Bot versions.)

## 2. Select the market

The bot ships on **Volatility 100 (1s) Index** (`1HZ100V`), contract type Digits → Over/Under, duration 1 tick.

To change the market, edit the first block ("Trade parameters"):
- Market → Derived → Continuous Indices → choose another Volatility index.
- Keep Trade type = *Digits → Over/Under* and Contract type = *Both*.

Not every index offers every barrier or minimum stake. Deriv Bot cannot query this in advance. If Deriv rejects a purchase (for example stake below the minimum), the platform shows an error. Adjust `MinimumStake` / `InitialStake` accordingly.

## 3. Configure stakes and limits

Open the **Trade parameters → Initialization** section. Every setting is a variable assigned there. Edit the number blocks.

| Variable | Default | Meaning |
|---|---|---|
| `InitialStake` | 0.35 | Base stake per trade |
| `MinimumStake` | 0.35 | Smallest stake the platform accepts for your market (check it) |
| `MaximumStake` | 1 | Hard ceiling; stake is never above this |
| `MaxRiskPercent` | 5 | A single stake may not exceed this % of current balance |
| `TakeProfit` | 5 | Stop opening trades when session net profit ≥ this |
| `StopLoss` | 3 | Stop opening trades when session net loss ≥ this (positive number) |
| `MaxConsecutiveLosses` | 3 | Stop after this many losses in a row |
| `MaximumDrawdown` | 2 | Stop when balance falls this far below its session peak |
| `StakeBoostThreshold` | 1 | Session net profit required before stake can rise |
| `StakeBoostFactor` | 1.5 | Stake multiplier once the threshold is met (still capped by `MaximumStake`) |
| `Barrier` | 4 | Digit barrier (1–8 allowed). Over wins on digits > barrier; Under on digits < barrier |
| `InitialWindow` | 100 | Short analysis window (ticks) |
| `ExtendedWindow` | 500 | Long analysis window (ticks); no trading until this many ticks exist |
| `RequiredZ` | 3 | Evidence needed for a "moderate" signal (standard errors above break-even) |
| `StrongZ` | 4 | Evidence needed for a "strong" signal |

Rules the bot applies at start-up: if the stake is outside min/max, a limit is ≤ 0, the barrier is outside 1–8, or the windows are inconsistent, `TradingEnabled` is set to 0 and an error notification appears. It then never trades.

Stake never increases after a loss. Stake only rises after the session is net profitable, with no losing streak, and drawdown below half the limit.

## 4. Test on demo

1. Confirm the account badge shows **Demo**.
2. Press **Run**. The bot needs `ExtendedWindow` ticks (500) of history before it can analyse. On a 1-second index that is under ~10 minutes if the platform supplies that many ticks (see limitation below).
3. Open the **Summary / Transactions / Journal** panels. Use **Journal** to read the bot's notifications.
4. Run a long demo session. **Expect many idle periods and often zero trades** — see the strategy document for why.

Variables you can watch in the Run panel (add them to the *Variables* view): `TotalTicks`, `SignalStrength` (0 none, 1 moderate, 2 strong), `EstimatedWinProbability`, `BreakEvenProbability`, `ExpectedValue`, `SessionNetProfit`, `ConsecutiveLosses`, `StopReason`, `TradingEnabled`.

`StopReason` codes: 0 running · 1 take-profit · 2 stop-loss · 3 consecutive losses · 4 drawdown · 5 stake/risk rule · 9 trading disabled (configuration).

**Known platform limitation to check first:** the bot reads the platform's digit list and needs ≥ 500 entries. If your Deriv Bot version only exposes fewer ticks, `TotalTicks` will never reach 500 and the bot will correctly *never trade*. If that happens, lower `ExtendedWindow` (and keep `InitialWindow` smaller), accepting weaker statistical evidence.

## 5. Start and stop

- **Start:** Run button (bottom of Bot Builder). 
- **Stop manually:** Stop button at any time. An already-open contract settles normally.
- **Automatic stop behaviour:** 
  - After a trade settles, if any limit is hit the bot shows a warning and ends the run.
  - If a limit is hit while waiting for a signal, the bot stops opening trades, shows one warning, and sits idle — **press Stop yourself**. (Deriv Bot has no "stop the bot" block, so an idle bot keeps listening to ticks without trading.)
- **Limits never reset automatically.** To trade again, press Run (a fresh session starts with fresh counters) after deciding that doing so is sensible.

## 6. Before using a real account

Don't, until you have read the risks in `Strategy_Explanation.md`. In particular: the digits of Deriv synthetic indices are designed to be random, so the bot's statistical edge detection is not expected to find a real edge. Real-money use is at your own risk.
