---
title: "The Coin Flip Exorcism: Teaching a Trading Bot to Stop Gambling"
date: 2026-04-18 21:00:00 +0900
categories:
  - Trading
  - Infrastructure
tags:
  - finint
  - polymarket
  - market-microstructure
  - kubernetes
  - signal-processing
  - debugging
  - claude-code
layout: post
math: true
image:
  path: /assets/img/posts/coin-flip-exorcism/pnl-breakdown.png
  alt: "P&L breakdown by strategy — every single one in the red. Negrisk Arb: -$294. Updown Capture: -$157. Total carnage."
---

87% directional accuracy is a lie. Tor is a dumpster fire for trading execution. And the difference between a coin flip and an actual edge is about 1.5 standard deviations.

The bot had been running for eight days. On paper, it looked like this: 87% win rate, signals firing every few seconds, six strategies running in parallel across crypto prediction markets. The dashboard was full of green. The wallet was down $325.

That contradiction — green metrics, evaporating money — is a specific failure mode in automated trading. It's not a bug. It's not bad luck. It's a system that learned to measure the wrong thing. We spent twelve hours tearing the pipeline apart to find out that 72% of our signals were indistinguishable from flipping a coin.

## The Reboot Loop from Hell

It started with the API being killed. The Kubernetes liveness probe was checking database connectivity every 30 seconds. When the DRBD storage layer had a cross-site replication hiccup — which it does, because physics is a bitch between Austin and Tokyo — PostgreSQL queries would timeout. The health endpoint returned 503. Three failures later, Kubernetes murdered the pod.

Eight times in eight hours. Each restart destroyed in-flight state: open price captures, signal tracking goroutines, position monitoring. The bot would come back, re-discover all the markets, fire duplicate signals for markets it had already seen, and start the cycle again.

**The fix:** Surgical separation. A new `/healthz` endpoint answers "is this process alive?" without touching the database. The readiness probe (`/ready`) still checks DB connectivity so traffic stops routing during outages, but the pod stays alive. Its state survives.

```
Before: DB hiccup → 503 → pod killed → state lost → restart → repeat
After:  DB hiccup → unready (no traffic) → pod alive → DB recovers → ready again
```

## The SKIP Graveyard

With the pod stable, we could finally look at the data. 92% of all signals were marked SKIP because the fill simulator's price cap was too tight. It used `maxOdds = 0.85` as the paper trading cap — if the market's best ask was above 85 cents, the simulator said "wouldn't fill." 

But the highest-confidence signals (90%+) are exactly the ones where the market has already moved past 85 cents. We were systematically discarding the highest-confidence signals. The SKIP rate dropped from 92% to under 20% overnight.

## The Phantom P&L

The P&L tab showed $0.00 for most trades. Not because trades weren't happening, but because the Go struct had `PnL` (capital P) while the JavaScript frontend was reading `pnl` (all lowercase). Go's default JSON serialization is case-sensitive. Every P&L value was silently `undefined`. 

One-line fix. Three days of invisible losses.

## The Negrisk Arb

We also found a collision in the risk management layer. Two different config files were trying to set the same bet sizing parameters, and the one that won was a hardcoded default from a three-month-old test. Every time we updated the "live" config, the bot ignored us and kept betting $50 on signals with 2% edges.

**The Rule:** If changing a trading parameter requires a redeploy, it's in the wrong place. Everything moved to a Postgres settings table, hot-reloadable via the UI.

The real ledger:

| Strategy | Trades | WR | P&L |
|----------|--------|-----|------|
| Up/Down Capture | 78 | 18.2% | -$152 |
| Follow Mirror | 3 | 0% | -$40 |
| News Trader | 3 | 0% | -$11 |
| **Total** | **84** | **17.1%** | **-$203** |

## The Coin Flip

The model was treating 0.01% price moves (0.3σ) as high-probability signals. Worse, 72% of all signals entered at **50 cents** — the exact point where the prediction market has no opinion. The model was screaming "BUY," but the market was literally a coin flip. We weren’t predicting. We were just sampling noise.

Two new gates:
- **Sigma gate** (≥1.5σ): Only fire when the price move is statistically significant.
- **Odds dead zone** (skip 42-58¢): Only fire when the market has actually moved away from 50/50.

## The Tor Tax

Every order was routing through Tor. In a market where the arbitrage window is ~2.7 seconds, we were burning 500ms on relay latency. We ditched Tor for Mullvad VPN proxies on our Austin and Tokyo nodes. Latency dropped to 170ms. The Binance WebSocket actually stays connected now.

---

Up to this point, we had fixed the plumbing. What we didn't have was a theory of why a signal deserved capital.

![The autotrade dashboard after the fixes — live scanner streaming signals, CALM regime, 611 markets tracked, circuit breaker inactive.](/assets/img/posts/coin-flip-exorcism/autotrade-dashboard.png){: .shadow }
_The dashboard after the exorcism. CALM regime, live signals streaming, and — for the first time — honest numbers._

---

## The Derivatives

A Polymarket contract isn't some exotic instrument. It's a cash-or-nothing binary call option. The price is the implied probability. Every piece of options pricing theory from the last fifty years applies.

Black-Scholes gives us the fair value:

$$
d_2 = \frac{\ln(S/K) - \frac{1}{2}\sigma^2 T}{\sigma\sqrt{T}}
$$

$$
\text{TheoreticalEdge} = N(d_2) - P_{\text{market}}
$$

Where $S$ is spot, $K$ is the strike (implied by the contract condition), and $T$ is time to expiry. 

Here’s the thing: **The Greeks flip sign for ITM binaries.** Regular options have positive Gamma (movement helps). But a binary option pinned at 90¢ has nowhere to go but down. The payoff is concave. Gamma inverts. Theta becomes positive — every second that passes without a reversal is a second closer to collecting your dollar.

This sign inversion cracked the strategy separation: Up/Down Capture wants volatility (positive Gamma); ITM Capture wants stillness (negative Gamma). We’d been feeding them the same parameters like a cat and a fish.

![The greeks analytics page — factor trend chart showing sigma depth and WR% climbing over time, with the gamma regime indicator reading "-Γ concave ITM" in red.](/assets/img/posts/coin-flip-exorcism/greeks-analytics.png){: .shadow }
_The greeks analytics page. Factor trend shows sigma depth (orange) climbing while WR% tracks it. Bottom right: the gamma regime detector reads "-Γ concave ITM" — the system knows which side of the payoff curve it's on._

Now, every signal carries its theoretical edge:

```
greeks-live sym=BTCUSDT Δ=0.714 Γ=-8.32 IV/RV=1.12 E=+0.043
```

Four cents of edge. On a binary that settles at $1, that’s a money printer — if the model is even approximately right.

## The Feedback Loop

We replaced static thresholds with four methods that learn from their own output:

**The vol gate learns where to stand.** Every 5 minutes, the system queries 14 days of signals. If winners cluster in a specific IV/RV band, the gate centers itself there. It literally watches what works and moves.

**Gamma gets a leash.** If losses cluster at high $\lvert\Gamma\rvert$ while wins stay moderate, cap entry at $1.5\times$ the winning centroid. Losers at $\lvert\Gamma\rvert=12$, winners at $\lvert\Gamma\rvert=5$? It stops going where it gets flipped.

**Delta becomes a Kelly multiplier.** We use a Bayesian prior on signal quality. If a signal's delta is closer to historical winners, we size up (1.5×). Closer to losers? We size down (0.5×).

**The flip-risk tax.** On top of everything, Kelly gets damped by the gamma norm:

$$
f_{\text{adj}} = f_{\text{Kelly}} \cdot \frac{1}{1 + \gamma_{\text{norm}}} \quad \text{where} \quad \gamma_{\text{norm}} = \min\!\left(\frac{|\Gamma|}{20},\; 2.0\right)
$$

High-gamma entries get smaller bets regardless. You might be right, but if the payoff is concave, you don't want to be right with your whole bankroll.

**The regime detector.** It measures the "flip rate" — orders that reverse by T+60s. Above 35%? The market is choppy. We tighten thresholds and halve position sizes.

The closed loop:

```
Signal → Greeks computed → Order fills → Outcome resolves → DB
    ↑                                                        |
    └──── Cache (5min) ← aggregates win/loss centroids ──────┘
```

No human in the loop. The static thresholds serve as floors — data can only tighten, never loosen below them — and they become irrelevant as the sample grows.

## The Nemotron Governor

Greeks handle the micro: *this signal, this size, right now*. The macro question is different: should this strategy even be running? With what baseline parameters?

The old Governor was a 6-hour grid search. Eight parameters, coarsely discretized, $O(n^8)$. Most cycles timed out. Even when it found a "best" set, it couldn't tell you *why* — just that it backtested well on the last 48 hours. Textbook overfitting.

So we gave the job to a 30-billion-parameter language model.

Nemotron-Nano-30B, running on our vLLM cluster — the same GPU infrastructure we built for malware analysis, repurposed for quantitative trading. Every 4 hours, it gets 7-day metrics across **all five strategies** with the current parameters, Greek aggregates, and a structured prompt. Hard bounds on every parameter. Maximum ±20% change per cycle. It can turn the dials — it cannot unplug anything. No live/dead flags. No dry_run_mode. Those are human decisions.

The thing that makes this genuinely interesting: **the model sees all five strategies at once.** A grid search optimizes in isolation. The LLM sees that Up/Down's $\bar{\Gamma}_{\text{loss}} = -12.3$ while ITM Capture's $\bar{\Gamma}_{\text{win}} = -11.8$ and thinks: "these aren't bad signals — they're in the wrong strategy." That's a cross-strategy correlation no single-strategy optimizer would ever reach.

The Discord notifications:

```
Greeks Analyzer — Parameter Update
"WR for high-sigma signals (68%) significantly exceeds low-sigma (41%).
 Raising updown_min_sigma 1.42 → 1.70 to filter noise-band entries."

• updown_min_sigma: 1.4200 → 1.7000
• updown_size_frac: 0.1500 → 0.1350
```

The model doesn't start smart. It just runs every 4 hours, never gets emotional about a losing streak, and the signals it tunes today become the data for its next cycle. The centroids sharpen. The IV/RV band narrows. Better data in, better suggestions out. Not a model improving its weights — a well-prompted model with an ever-improving feed.

## The Open Question, Revisited

Ten days ago: *signal or noise?*

Signal. Narrower than we thought — high sigma, mid-range odds, favorable IV/RV, moderate gamma — but it's there. Win rate in that corridor is meaningfully above 50%. Outside it, the system correctly declines to trade.

But the honest version is that we traded one open question for a better one. The old question was whether directional prediction has alpha. The new question is whether the *system* — self-calibrating Greeks, regime detector, LLM Governor — converges toward profitability or oscillates around breakeven.

A human wrote the math. A human set the bounds. From here, the system runs itself.

The instruments are correct. The data is clean. The loop is closed.

Now the market gets to decide if we were right.
