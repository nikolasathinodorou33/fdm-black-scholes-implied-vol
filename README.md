# Pricing Options with Finite Differences: Where Historical Volatility Fails

A finite difference (FDM) solver for the Black-Scholes equation, validated against the exact formula and then tested against real S&P 500 option prices using implied volatility.

**Main finding:** pricing with historical volatility gets ordinary options roughly right, but values crash protection at almost nothing. A 29-day S&P 500 put with its strike 10% below the index costs **10.45 points** in the market; the FDM at historical volatility prices it at **0.06**. A seller using that model would be both underpaid and underhedged (holding about **2%** of the hedge the market's pricing implies) for exactly the event that hurts most.

---

## The question

Black-Scholes needs one input nobody knows: volatility. Part 1 estimates it from the past (historical volatility). Part 2 asks: **if you price options with historical volatility, how far are you from what the market actually charges, and where?**

## Part 1: The FDM pricer

The Black-Scholes PDE is solved with an explicit finite difference scheme on a grid of stock prices and times. Starting from the known payoff at expiry, the solver steps backwards in time; each grid value is a weighted average of three neighbouring values one time step later. Boundary conditions fix the value at S = 0 and at the top of the grid.

**Validation.** For a 1-year at-the-money call (S = 100, K = 100, r = 5%, σ = 20%):

| | Price |
|---|---|
| FDM (M = 100, N = 10,000) | 10.4658 |
| Black-Scholes formula | 10.4506 |
| Absolute error | 0.0152 |

**Convergence.** Halving the stock-price step reduces the error about four times, the second-order convergence expected from central differences:

| M | dS | Absolute error |
|---|---|---|
| 20 | 15.0 | 0.4308 |
| 50 | 6.0 | 0.0633 |
| 100 | 3.0 | 0.0152 |
| 200 | 1.5 | 0.0039 |

![Convergence](figures/convergence_loglog.png)

**Stability.** The explicit scheme is stable only if σ²M²dt < 1 (all three weights stay positive). The test case gives 0.04; the solver in Part 2 increases the number of time steps automatically when needed.

### The Greeks: delta and gamma

Both are read directly from the FDM grid using differences between neighbouring stock prices.

**Delta** is the rate of change of the option price with respect to the stock price: how much the option's value changes when the stock moves by 1. It is also the number of shares a seller holds to hedge one option.

![Delta](figures/delta.png)

With the strike fixed at 100, delta rises from 0 to 1 as the stock price rises relative to the strike:

- **Stock far below the strike (deep out of the money):** delta ≈ 0. The option will almost certainly expire worthless, so small stock moves barely change its value.
- **Stock far above the strike (deep in the money):** delta ≈ 1. The option will almost certainly be exercised, so it moves one-for-one with the stock.
- **Stock near the strike:** delta is a little above 0.5 (about 0.64 at S = 100). Interest makes the stock drift upwards in the model, so an at-the-money call is slightly more likely to finish in the money than not. The notebook reports delta = 0.617 at the nearest grid point, S = 99, matching Black-Scholes there (0.618).

**Gamma** is the rate of change of delta with respect to the stock price: the curvature of the option price. It measures how quickly the hedge goes out of date as the stock moves.

![Gamma](figures/gamma.png)

Gamma is the slope of the delta curve, so it peaks where delta changes fastest. Far from the strike, the outcome is almost decided either way and delta is flat, so gamma is near zero. Near the strike, the outcome is finely balanced: a small move in the stock noticeably changes how likely the option is to be exercised, so delta changes quickly.

The peak is at S ≈ 89.6, below the strike, for two reasons. Stock prices move in percentages, so a move of 1 is a larger percentage move at lower prices, which pushes gamma up on the left. And in the model the stock is expected to drift upwards over the year, so the point where the outcome is most balanced already sits below 100. Black-Scholes puts the peak at K·exp(−(r + 1.5σ²)T) = 100·exp(−0.11) ≈ 89.6, matching the FDM result. At S = 99, gamma = 0.0193 (Black-Scholes: 0.0193): delta rises by about 0.019 for each 1 increase in the stock price.

As expiry approaches, the delta curve becomes steeper and the gamma peak narrower and taller, moving towards the strike: on the final day, delta jumps from 0 to 1 exactly at the strike.

### Does the pricer behave as theory predicts?

Beyond matching the exact formula, each sensitivity behaves as expected:

- **Volatility.** The test call rises steadily from 6.83 at 10% volatility to 21.80 at 50%. More volatility means more chance of a large gain while the loss is capped at the premium. Because price always increases with volatility, each market price corresponds to exactly one implied volatility (used in Part 2). The FDM error falls as volatility rises (0.020 at 10%, 0.007 at 50%), because a smoother price curve is easier to approximate on a fixed grid.
- **Maturity.** A one-month at-the-money NVDA call (S = 230.86, K = 230, σ = 39.4%) costs 11.18; at 3, 6 and 12 months it costs 19.42, 27.78 and 40.30. Price grows roughly with √T (11.18 × √3 ≈ 19.4, × √6 ≈ 27.4), since uncertainty grows with the square root of time; longer maturities sit slightly above this because of the interest effect on the strike. The FDM error also falls with maturity (0.020 at 1 month, 0.006 at 1 year), for the same smoothness reason: what controls the difficulty is σ√T.
- **Across stocks.** For one-month at-the-money calls on five stocks, option price divided by stock price follows the rule of thumb 0.4 × σ × √T closely, from 2.3% for JPM (σ = 20.7%) to 6.1% for TSLA (σ = 52.9%). Small deviations are explained by strikes rounded slightly in or out of the money.
- **Historical volatility window.** For NVDA the 30-day, 90-day and 1-year estimates (38.4%, 39.4%, 37.7%) give option prices of 10.93, 11.18 and 10.73: the price depends on which past window is chosen, and nothing in the model says which is right. Part 2 addresses this with market prices.

## Part 2: Implied volatility

**Data.** S&P 500 index options (European, cash-settled) from Yahoo Finance for expiries of 29, 60, 91 and 181 days. Prices are bid-ask mids; options with no quotes, spreads above 50% of the mid, or strikes outside 70–130% of the index level are removed. Index level 7,666.54; 3-month T-bill rate 3.98%; 90-day historical volatility 12.45%.

**Method.**

1. The pricer is extended to puts and continuous dividend yield, vectorised, and re-checked: call − put on the FDM grid satisfies put-call parity exactly (4.8771).
2. The dividend yield for each expiry is backed out from put-call parity, so the forward price comes from the options themselves.
3. Implied volatility is found with Newton's method (using vega), with Brent's method as a fallback. Test: volatilities recovered with a maximum error of 5 × 10⁻⁸.
4. Data check: calls and puts at the same strike give implied volatilities within 0.26 vol points on average.
5. Every option is priced with the FDM at historical volatility and compared with its market price. As a check, the FDM at each option's own implied volatility reproduces market prices within 0.3%.

### Result 1: the volatility skew

If Black-Scholes were right, implied volatility would be the same at every strike. It isn't:

![Implied volatility skew](figures/iv_skew.png)

For 29-day options, implied volatility is about 48% for a strike 30% below the index, 23.8% at 10% below, and 13.7% at the money, against 12.45% historical. Crash protection is priced as if the index were several times more volatile than it has been. Shorter maturities show the steepest skew; at-the-money implied volatility rises with maturity (13.7% at 29 days to 15.7% at 181 days).

### Result 2: the cost of pricing with historical volatility

| Option | Strike / index | Market price | Implied vol | FDM at hist. vol | Gap |
|---|---|---|---|---|---|
| put | 0.85 | 5.30 | 29.8% | 0.0001 | −100% |
| put | 0.90 | 10.45 | 23.8% | 0.06 | −99% |
| put | 0.95 | 28.00 | 18.4% | 6.38 | −77% |
| put | 1.00 | 101.40 | 13.7% | 90.49 | −11% |
| call | 1.05 | 9.10 | 11.3% | 13.47 | +48% |

![Pricing gap](figures/pricing_gap.png)

The model is closest near the money, badly underprices crash protection, and overprices modest upside calls.

![Market vs FDM prices](figures/market_vs_fdm_prices.png)

### Result 3: the hedge is wrong too

For the put with strike 6,900 (10% below the index), delta is −0.0008 under historical volatility and −0.0477 under its implied volatility. A seller hedging with historical volatility holds about 2% of the hedge the market's own pricing implies.

## Conclusion

The failure is not only that historical volatility looks backwards (the last six weeks were unusually calm, with 30-day volatility at 9.8%). Even the market's own at-the-money volatility would still price crash puts near zero: a bell curve with any single realistic volatility treats a 15–30% fall within a month as essentially impossible, while the market prices it as rare but real. The problem is the assumption that one volatility describes all outcomes. In practice this is handled by using a different implied volatility for each strike and maturity, or models with jumps or stochastic volatility.

## Limitations

- **Snapshot data.** Quotes were downloaded on 1 October 2026 after the US close, so they are end-of-day rather than live. Results are a single snapshot.
- **Implied dividend yield.** The yield backed out from parity is negative for short expiries (−1.55% at 29 days). It absorbs a small timing mismatch between the index close and option quotes and a difference between the T-bill rate and the rate implied in option prices. The forward price, which is what the model uses, is still taken from the options themselves, so implied volatilities are unaffected (confirmed by the parity check).
- **Data noise.** A few isolated spikes in the longer-maturity skew curves come from duplicate or stale quotes, not real features.
- **Hedging comparison.** Implied-volatility deltas assume each option's implied volatility stays fixed as the index moves; in sell-offs implied volatility tends to rise, so the true hedge would likely be larger still.
- **Numerical method.** The explicit scheme is simple and transparent but needs small time steps for stability. Crank-Nicolson would allow larger steps.

## Possible extensions

- Repeat the analysis on a stressed date (e.g. March 2020) to see how the price of crash protection changes.
- Crank-Nicolson or implicit schemes.
- A model with jumps or stochastic volatility (e.g. Heston) calibrated to the skew.

## How to run

```bash
pip install -r requirements.txt
```

Open `fdm_black_scholes_implied_vol.ipynb` in Jupyter, VS Code or Google Colab and run all cells. For live option prices, run during US market hours (14:30–21:00 UK time). Results will differ from those above, which reflect the 1 October 2026 snapshot.

## Repository contents

- `fdm_black_scholes_implied_vol.ipynb`: the full notebook, with outputs
- `figures/`: plots used in this README
- `requirements.txt`: Python libraries needed
