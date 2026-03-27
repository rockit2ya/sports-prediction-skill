# Sports Prediction Engine Framework

## Skill Purpose

Guide the creation and tuning of a **spread/line prediction engine** for any sport (NBA, NFL, MLB, NHL, Soccer). This skill captures the architecture, methodology, and tuning discipline proven in the NBA Prediction Engine (v3.60, 60.5% ATS).

**USE FOR:** building a new prediction engine for a sport, adding guard rails, implementing a fade evaluator, setting up market anchoring, creating a bet tracker, backtesting parameter changes, calibrating Judge rates.

**DO NOT USE FOR:** live API integration details (sport-specific), odds scraping, bankroll management math, or UI/frontend design.

---

## 1. Architecture Pattern (Sport-Agnostic)

Every prediction engine follows this pipeline:

```
Fair Line (N components) → Market Anchor → Edge → Guard Rails → Fade Evaluator → Judge → Bet / Shadow / Fade
```

### 1.1 Fair Line Calculation

Sum N sport-specific components to produce a **projected spread** (or projected total, moneyline implied probability).

| Component | NBA Example | NFL Example | MLB Example | NHL Example | Soccer Example |
|---|---|---|---|---|---|
| Efficiency gap | OFF/DEF rating (pace-adjusted) | DVOA / EPA per play | wOBA / FIP differential | xGF / xGA per 60 | xG / xGA per 90 |
| Home advantage | HCA (location splits or flat) | HFA (~2.5 pts) | HFA (~0.5 runs) | HIA (~0.15 goals) | HFA (~0.3 goals) |
| Rest / fatigue | B2B penalty, schedule density | Short week / bye week | Day game after night, doubleheaders | B2B, 3-in-4 nights | Midweek fixture congestion |
| Key player impact | Star tax (on/off +/-) | QB injury model, skill position impact | Starting pitcher (ERA/FIP) + lineup WAR | Goalie strength, top-line injuries | Key player absence (xG contribution) |
| Situational | Motivation (tanking), SOS | Divisional rivalry, weather, travel | Park factors, platoon splits, bullpen usage | Travel, altitude, rivalry | Derby matches, European competition fatigue |

**Principles:**
- **Regression to the mean** — Never use raw stats. Regress toward league average (typically 50–65% shrinkage) to prevent overfitting to small samples.
- **Recency weighting** — Blend recent form (last 15–30 games) with season averages. Use adaptive weights: increase recent weight when deviation from season is large.
- **Component logging** — Log every component value per game to a `fair_line_log` CSV. This is your backtesting lifeline.

### 1.2 Market Anchor

**Critical: Raw model predictions are not profitable on their own.** The market is highly efficient. You must blend your model's fair line with the market consensus:

```
anchored_line = (MARKET_ANCHOR_WEIGHT × market_line) + ((1 - MARKET_ANCHOR_WEIGHT) × model_line)
```

- Start at **0.65 market / 0.35 model** and tune from there.
- Post-anchor edge = `(1 - MARKET_ANCHOR_WEIGHT) × |model_line - market_line|`
- This means a small config change (e.g., dampener ±0.03) has **large downstream effects** on edge distribution.

### 1.3 Edge & Confidence

```
edge = |anchored_line - market_line|
```

**Edge Confidence Score (ECS):** A 0–100 composite from multiple signals:
- Agreement between model direction and market movement
- Injury data completeness / recency
- Number of data sources corroborating the pick
- Historical accuracy at this edge magnitude
- Swing-spot bonus (games where your model has highest conviction)

**ECS Tiers:** HIGH (80+), MODERATE (60–79), LOW (40–59), NO TRUST (<40).

### 1.4 Guard Rails

Guard rails are **shadow tags** that block bets from becoming real. Shadow bets are logged at $0 for backtesting — this is how you discover which guard rails are accurate.

**Universal guard rails** (apply to any sport):
| Tag | Trigger | Purpose |
|---|---|---|
| `LOW_EDGE` | edge < min_edge | Not enough signal to overcome vig |
| `TEAM_BLACKLIST` | Model historically wrong on this team | Remove persistent losers |
| `INJURY_BREAKER` | N+ key players OUT | Too much uncertainty |
| `HOME_PICK_PENALTY` | Picking home favorite | Market prices HFA efficiently; raise min_edge |
| `PICK_BOY` | Model picks weaker team in lopsided matchup | Historically terrible WR |
| `CONFIDENCE_CONFLICT` | High edge but low ECS | Signals disagree |

**Sport-specific examples:**
- **NFL:** `WEATHER_RISK` (wind >25mph kills passing), `SHORT_WEEK` (Thursday games)
- **MLB:** `BULLPEN_DEPLETED` (3+ innings last 2 days), `PARK_FACTOR_EXTREME`
- **NHL:** `GOALIE_UNCERTAINTY` (starter unconfirmed), `SCHEDULE_CRUSH` (4 games in 6 nights)
- **Soccer:** `ROTATION_RISK` (Champions League midweek), `MANAGER_CHANGE` (first 3 games)

**Key discipline:** Promote/demote guard rails based on backtested WR data, not intuition. A guard rail with 45% WR on shadowed bets is **saving you money**. One at 55% should be removed.

### 1.5 Fade Evaluator

The fade is an **independent evaluator**, not a mechanical inverter. When guard rails fire, the fade examines the matchup on its own terms:

1. **Blacklist check** — Is the model picking a blacklisted team? If so, DISAGREE.
2. **Lopsided detection** — Is there a massive quality gap (e.g., NET rating gap ≥ 5)? Strong teams cover.
3. **Injury mismatch** — Does the model's pick have significantly more injury burden?
4. **ECS trust** — Does the model even trust its own pick? Low ECS = more likely to fade.
5. **Multi-tag red flags** — Multiple shadow tags = compounding risk.
6. **Fade destination health** — Don't fade INTO a crippled team (check destination injuries).

The fade outputs: **AGREE** (align with model) or **DISAGREE** (recommend opposite side), with stated reasoning.

### 1.6 Judge (Arbitration Layer)

The Judge resolves model–fade disagreements using a **rate cascade**:

```
Base rate selected by tag priority:
  lopsided (78) → injury_mismatch (62) → no_trust (45) → multi_tag (56) → low_edge_only (52) → home_fade (53) → away_fade (47)

Then apply modifiers:
  ± direction penalty (model opposes market)
  ± ECS floor penalty (low confidence)
  ± tight line boost (weak market consensus)
  ± fade suppress penalty (historically bad fade targets)
  ± NET gap boost (fading toward stronger team)

Final rate ≥ fade_threshold → FADE
Final rate < fade_threshold → ALIGNED (go with model)
```

The rates are **empirically calibrated** from your shadow bet history. Start with 50/50 and adjust based on actual WR data per tag.

---

## 2. Config-Driven Design

**Never hardcode tunable values.** All parameters live in a single `model_config.json`:

```json
{
  "version": "1.0",
  "last_updated": "YYYY-MM-DD",
  "model_params": {
    "MARKET_ANCHOR_WEIGHT": 0.65,
    "EFFICIENCY_REGRESSION": 0.65,
    "RECENCY_WEIGHT": 0.40,
    "KEY_PLAYER_DAMPENER": 0.13,
    "HOME_ADVANTAGE_FLAT": 2.5,
    "RESIDUAL_HCA": 0.0,
    "min_edge": 5,
    "edge_cap": 10,
    "edge_ceiling": 12
  },
  "guard_rails": {
    "team_blacklist": [],
    "injury_breaker_threshold": 5,
    "home_pick_penalty": 2,
    "fade_threshold": 50,
    "judge_rates": {
      "lopsided": 78,
      "injury_mismatch": 62,
      "no_trust": 45,
      "multi_tag": 56,
      "low_edge_only": 52,
      "home_fade": 53,
      "away_fade": 47
    },
    "fade_suppress": {
      "enabled": true,
      "teams": [],
      "penalty": 15
    }
  }
}
```

**Access pattern:** Use a helper function with fallback defaults:
```python
def _p(key, default=None):
    return model_params.get(key, default)
```

---

## 3. Data Architecture

All data is **cached locally** (JSON/CSV). No live database required.

| File Pattern | Purpose |
|---|---|
| `{sport}_stats_cache.json` | Season-level team ratings |
| `{sport}_stats_recent_cache.json` | Recent form (last N games) |
| `{sport}_injuries.csv` | Current injury/absence report |
| `{sport}_player_impact_cache.json` | Player on/off or WAR-style impact data |
| `odds_cache.json` | Multi-book lines and consensus |
| `fair_line_log_YYYY-MM-DD.csv` | Daily component breakdown per matchup |
| `bet_tracker_YYYY-MM-DD.csv` | Daily bet log (real, shadow, fade, override) |
| `game_results_cache.json` | Actual results for backtesting |

**Bet tracker columns:** Date, Matchup, Pick, MarketLine, FairLine, Edge, ECS, Stake, Notes (SHADOW:TAG), Result, ModelVersion.

---

## 4. Tuning & Backtesting Methodology

### 4.1 Change Validation Process

**Every parameter change must be backtested before shipping:**

1. **Identify signal** — Find a pattern in shadow bet data (e.g., "away fades win only 46%").
2. **Quantify scope** — How many bets does this affect? Changes affecting <5 bets are noise.
3. **Backtest** — Replay historical bets with the proposed change. Count wins flipped and wins broken.
4. **Net impact** — Only ship if `wins_gained - wins_lost > 0` AND the improvement isn't fragile (e.g., all from one week).
5. **Update config** — Change `model_config.json`, bump version.
6. **Document** — CHANGELOG entry with: Problem → Root Cause → Evidence → Change → Validation.

### 4.2 Key Metrics

- **ATS Win Rate** — Primary metric. Track by edge bucket, tag, home/away, team.
- **Bias (Mean Signed Error)** — Should be near 0. Positive = model favors home too much.
- **MAE (Mean Absolute Error)** — Lower is better, but doesn't directly drive WR.
- **ROI** — Revenue per dollar wagered. Accounts for juice (-110 standard).
- **Shadow Tag Accuracy** — WR of shadowed bets. Below 45% = guard rail is saving money.

### 4.3 Auto-Tune

Use ridge regression against `fair_line_log` + `game_results_cache.json` to find optimal component weights. Constrain coefficients to prevent sign flips (e.g., HCA should never be negative).

---

## 5. Version & Release Discipline

1. Update parameters in `model_config.json` (bump version and last_updated)
2. Update CHANGELOG.md: Problem → Root Cause → Evidence → Changes → Validation
3. Update documentation (README, glossary, guides)
4. Commit with descriptive message referencing the change
5. Create and close GitHub issue for traceability
6. Delete temporary analysis scripts (prefix with `_` for easy cleanup)

---

## 6. Sport-Specific Starter Guides

### NFL
- **Efficiency:** DVOA (Football Outsiders) or EPA/play (nflfastR)
- **Key player:** QB is ~60% of team value. Build a robust QB adjustment model.
- **HFA:** ~2.5 points, declining over time. Consider altitude (Denver +1.0).
- **Situational:** Weather (wind, cold, rain), travel distance, divisional familiarity, bye weeks.
- **Sample size:** 17 games/season is tiny. Regression must be aggressive (70%+).
- **Guard rails:** `PRIMETIME_TRAP` (public overbet), `LOOKAHEAD` (team looking past weak opponent).

### MLB
- **Efficiency:** Run differential, wOBA/FIP differential, park-adjusted.
- **Key player:** Starting pitcher is dominant. Use xFIP or SIERA, not ERA.
- **HFA:** ~0.5 runs (~54% home win rate). Smaller than other sports.
- **Situational:** Park factors, platoon splits (L/R), bullpen freshness, travel.
- **Sample size:** 162 games = large sample. Less regression needed (50%).
- **Guard rails:** `BULLPEN_DEPLETED`, `CALL_UP_UNCERTAINTY`, `PARK_EXTREME`.

### NHL
- **Efficiency:** xGF% (expected goals for percentage), Corsi/Fenwick at 5v5.
- **Key player:** Goaltender is ~40% of the variance. Track save% and xGA.
- **HFA:** ~0.15 goals (~55% home win rate for puck line).
- **Situational:** B2B games, travel, altitude, goalie confirmation timing.
- **Sample size:** 82 games. Moderate regression (60%).
- **Guard rails:** `GOALIE_UNCONFIRMED`, `SCHEDULE_CRUSH`, `BACKUP_START`.

### Soccer (European Leagues)
- **Efficiency:** xG (expected goals), xGA, xPts.
- **Key player:** Distribute impact more evenly. Focus on creative midfielders and strikers.
- **HFA:** ~0.3 goals (~46% home win rate post-COVID, was ~49% pre-COVID).
- **Situational:** European competition fatigue, derby matches, manager changes, relegation pressure.
- **Sample size:** 38 league games. Heavy regression (65%).
- **Guard rails:** `ROTATION_RISK` (Champions League midweek), `EARLY_SEASON` (first 5 rounds), `MANAGER_BOUNCE`.

---

## 7. Anti-Patterns (Lessons Learned)

1. **Don't skip market anchoring.** Raw model edges are unprofitable in every sport tested.
2. **Don't optimize for CLV.** Closing line value has zero/negative correlation with actual WR in this framework. Optimize for ATS wins.
3. **Don't trust small samples.** A 3W-0L signal is not a signal. Wait for 15+ bets before acting.
4. **Don't add guard rails without shadow-testing.** Let them run as shadows for 20+ bets, then check WR.
5. **Don't fade into injured teams.** Always check fade destination health before recommending the opposite side.
6. **Don't hardcode.** Every number that might change goes in config with a fallback default.
7. **Don't skip component logging.** Without `fair_line_log`, you cannot backtest. Period.
8. **Don't over-engineer the model.** The market anchor does the heavy lifting. Your model's job is to find the 35% the market misses.
9. **Verify safety gates actually block.** When implementing a gate that should prevent an action (e.g., AGREE gate blocking low-conviction picks), verify the blocked path produces a *different* output than the passed path. A gate that sets a flag but doesn't change the downstream recommendation is dead code. Discovered in NBA v3.65: gate-passed = 100% WR, gate-blocked = 40% WR, but blocked path still recommended BET.
10. **Check fade destination injury burden.** Beyond checking if the fade destination is a historically bad team (#5), also check if that team is severely injured. Fading INTO a crippled team (star tax ≥ 8.0) is catastrophic (~20% WR) regardless of other signals. This catches edge cases where other guards (like lopsided detection) are suppressed.
