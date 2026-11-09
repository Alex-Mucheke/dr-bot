# Strategy Explanation — Deriv Adaptive Digit Bot

## 0. Read this first

- **No guaranteed profit.** The bot cannot predict future digits.
- **Expected value is probably negative.** Deriv's synthetic-index digits are designed to be independent and uniform, and Deriv's payouts include a margin. If both are true, every Over/Under contract has negative EV for any barrier, and the bot's "edge detector" can only fire on statistical noise. This is stated as an assumption: it comes from how these indices are documented to work, and the real payout margin was not measured here.
- The bot's honest behaviour is therefore: **trade rarely or never.** A session with zero trades is the correct outcome when no validated advantage exists.
- **Not tested on real Deriv.** It was validated structurally and run against a mock (section 8).

## 1. Entry conditions

A trade is opened only if **all** are true:

1. `TradingEnabled = 1` (configuration valid, no stop triggered).
2. Risk check passes (section 5).
3. At least `ExtendedWindow` (500) ticks have been observed.
4. For Over *or* Under at the configured `Barrier`: the contract's real payout was read from the platform and is > 0, estimated win probability exceeds break-even by at least `RequiredZ` standard errors, and `ExpectedValue > 0`.
5. No contract is currently open (Deriv Bot runs one contract at a time here; `trade again` is only called after settlement).

If payout is unavailable (0), the candidate is skipped — the bot does not invent a payout or fall back to random entry.

## 2. Probability calculations

For each window (last 500 ticks and last 100 ticks):

- Over win probability = count(digit > Barrier) / N
- Under win probability = count(digit < Barrier) / N

**Conservative estimate:** the bot uses the *lower* of the 100-tick and 500-tick estimates for each side. A signal that appears in only one window is therefore discounted. This is the "signal stability across windows" check.

Break-even probability = `Stake / Total Payout` (using the payout read from the platform).

Expected value = `Estimated Win Probability × Payout − Stake`. This is a **hypothetical** quantity; realised results are tracked separately in `SessionNetProfit` (from the platform's own profit total).

**Reliability** is measured as a z-score:
`Z = (EstimatedP − BreakEven) / sqrt(EstimatedP × (1 − EstimatedP) / 500)`.
`Z ≥ RequiredZ (3)` → signal strength 1 (moderate). `Z ≥ StrongZ (4)` → strength 2 (strong). Anything lower → no trade.

Limitations of this method: it assumes past frequency estimates the future, which is exactly what is **not** established for random digits; checking every tick also runs many overlapping tests, so false positives will occasionally occur even if Z=3 sounds strict. The estimate is treated as uncertain, hence the high threshold.

Other tracked statistics: `Digit0Count`…`Digit9Count` (over the last 500 ticks), `EvenCount`, `OddCount`, `TotalTicks`. They are informational. Even/odd and digit-frequency counts do not drive entry (the contract type is Over/Under).

## 3. Stake adjustment rules

- Base stake = `InitialStake`.
- Stake is **not** raised after a loss. After any trade the stake is recomputed from `InitialStake`, not from the previous stake.
- Boost: if session net profit ≥ `StakeBoostThreshold`, consecutive losses = 0, and drawdown < half the limit, stake = `InitialStake × StakeBoostFactor`, rounded down to cents.
- Caps, applied in order: `MaximumStake`, then `MaxRiskPercent` of current balance.
- A stake below `MinimumStake`, above the cap, or above the balance fails the risk check and blocks trading (stop reason 5).

Mapping to the requested risk tiers: no/weak evidence → no trade (the Z gate); moderate → base stake; strong → modest increase only when session conditions are healthy; "high uncertainty" → the Z gate makes it a no-trade rather than a reduced stake.

*Simplification:* the boost depends on session health, not directly on signal strength; `SignalStrength` currently gates entry (>0) but does not scale the stake further.

## 4. Loss handling

After each settled trade the bot: counts the result, updates `ConsecutiveLosses/Wins`, recalculates session profit/loss, refreshes balance and peak, recomputes the stake from base, runs the risk check. After a loss it does **nothing special** — the next entry still has to pass the full signal gate independently. At `MaxConsecutiveLosses` (3) it stops.

## 5. Profit protection

Checked before every purchase and after every settlement:

| Control | Rule | Stop reason |
|---|---|---|
| Take profit | session net ≥ `TakeProfit` | 1 |
| Stop loss | session net ≤ −`StopLoss` | 2 |
| Consecutive losses | losses in a row ≥ limit | 3 |
| Drawdown | peak balance − balance ≥ `MaximumDrawdown` | 4 |
| Stake protection | stake outside min/max/risk cap/balance | 5 |
| Config invalid / disabled | `TradingEnabled ≠ 1` | 9 |

Limits trigger **after** a trade settles, so a final loss can overshoot a limit by up to one stake. Nothing is reset automatically.

## 6. Unsupported or simplified features

| Requested | What was done |
|---|---|
| Validate available contracts and minimum stakes | Deriv Bot cannot query this. Config is validated internally; the platform rejects unsupported purchases (check market/barrier/minimum stake yourself). |
| Confirm account/market availability, subscribe to ticks | Handled implicitly by the platform when the bot runs. |
| Decimal-precision digit extraction | Uses Deriv Bot's own `lastDigitList` block, which applies the platform's digit extraction. Not independently verified. |
| True rolling distributions | Recomputed each tick from the last 500/100 digits (a sliding window, but the 500-tick loop runs every tick, which is slower than incremental updating). |
| Per-trade payout "proposal" | Uses the platform's `payout` block for the current stake. The real proposal API is not reachable from Blockly. |
| "Recent winning and losing sequences" | Consecutive win/loss counters only. |
| Total profit / total loss | `SessionProfit` = sum of winning-trade profits, `SessionLoss` = sum of losing-trade losses, `SessionNetProfit` from the platform total. |
| Stake scaled by signal strength | Strength gates entry only (see section 3). |
| Auto-stop when idle | No such block exists; the bot idles and you press Stop. |
| Highest-balance tracking | Peak is tracked during the session only. |
| Martingale | Not implemented. |

## 7. Known risks

- Negative expected value is likely (section 0).
- The platform's digit list may hold fewer than 500 ticks; then the bot never trades.
- 1-tick contracts settle fast; slippage and platform delays are possible.
- Noise-driven signals can still lose money, and stops trigger only after a trade settles.
- Deriv Bot versions change block names; an importer may reject the file.
- Using real money multiplies every one of these risks.

## 8. Testing performed (and not performed)

**Done:**
- XML well-formedness, unique block ids, variable declarations match usage, all 34 required variable names present, one `purchase` per branch (two in total, only in `before_purchase`), `trade_again` once in `after_purchase`.
- Block-type check against a **whitelist written from knowledge of Deriv Bot's blocks — not an official schema file.** Treat this as a consistency check, not proof of importer compatibility.
- The XML itself was executed by a small interpreter against a mock Deriv feed (mock payout = stake × 0.965 ÷ win probability, an assumption). Scenarios passed:
  1. Fair random digits, 3,000 ticks → 0 trades.
  2. Synthetic 80%-over history with matching outcomes → traded, take-profit stopped at +5.11.
  3. History looks strongly over but every trade loses → stopped after exactly 3 trades (consecutive-loss limit).
  4. Loss-heavy pattern with small `StopLoss` → stop-loss triggered.
  5. Drawdown limit lowered → drawdown stop triggered.
  6. `InitialStake` 0.9 with large boost → stake never above `MaximumStake` (1.00).
  7. Invalid config → trading disabled, zero trades.
  8. Balance 5.0 → risk cap blocked trading.
  9. Only 300 ticks → no trading.

**Not done:** import into real Deriv Bot, any run on a real or demo account, verification of real payout figures, real tick-list length, or real contract rejections.

The mock tests prove the *logic as written* behaves as intended under the mock's assumptions, nothing more.
