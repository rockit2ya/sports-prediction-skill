# Sports Prediction Engine Framework

## Skill Purpose

Guide the creation and tuning of a **spread/line prediction engine** for any sport (NBA, NFL, MLB, NHL, Soccer). This skill captures the architecture, methodology, and tuning discipline proven in the NBA Prediction Engine (v4.8.0, modular architecture with ONE source of truth, 14 forced pick optimization mechanisms — 13 active, analytical confidence modifiers bridging parallel decision pipelines, injury mismatch cap preventing signal double-counting).

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
| Head-to-head | Season series record (±0.5–2.0 pts) | Divisional H2H (6 games/yr) | Season series, pitcher vs lineup | Season series | Recent meetings, home/away H2H |

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

### 1.7 Two Paths — Discipline vs. Conviction

The engine serves two fundamentally different modes of operation. Every architectural decision must account for both.

#### The Safe Path (non-forced)

The default mode. Guard rails shadow-block uncertain games. The Judge skips low-conviction matchups. Discipline preserves bankroll and win rate.

**Philosophy:** Not every game deserves your money. When the model is unsure, the fade evaluator is split, or the signals conflict — the right answer is to walk away. Shadow the bet at $0, log it for backtesting, and move on. A 58% win rate on 8 games beats a 52% win rate on 14. This path exists to protect you from the games the engine doesn't understand well enough to have an edge.

**The safe path accepts these costs:**
- Missed winners that guard rails blocked (the price of avoiding the losers those rails also caught)
- Lower volume (fewer bets per slate)
- Occasional frustration when a shadowed game would have covered

These costs are acceptable because the alternative — betting every game at reduced accuracy — is how bankrolls die.

#### The Gun-to-Head Path (forced / `--force`)

When you MUST bet every game on the slate — no skipping, no shadows, no walking away. This is the risky path where the engine has no choice but to find the best possible side for every matchup, including the ones it would normally refuse to touch.

**Philosophy:** When retreat is not an option, you attack with everything you have. The forced path activates every analytical signal the engine possesses — defense quality, ATS tiers, historical team WR, injury asymmetry, head-to-head matchup history, market alignment — and synthesizes them into a conviction score. Games the safe path would skip become puzzles the forced path must solve.

**The forced path demands:**
- **Signal depth parity with the Judge** — The forced pick pipeline must see every signal class the Judge uses. If the Judge evaluates 16 modifier classes, the forced confidence scorer must evaluate equivalent information, expressed as confidence adjustments rather than fade-rate modifiers. A forced path with fewer signals than the Judge is making blind decisions.
- **Triple consensus protection** — When model, fade, and market all agree, that alignment is sacred. No flip mechanism overrides it.
- **Injury-aware tier logic** — "Strong team" labels from season-long NET ratings are stale when key players sit. Gate strength-based auto-flips on current injury burden.
- **Last-resort market fallback** — When the model has zero directional signal (edge < 2), defer to the market favorite. The market is never perfect, but it's better than a coin flip.

#### The Judge's Ultimate Responsibility

The Judge is the final arbiter of truth in both paths. The model proposes. The fade evaluator challenges. **The Judge decides.**

In the safe path, the Judge resolves model–fade disagreements: ALIGNED (trust the model), FADE (trust the fade), or SKIP (trust neither). In the forced path, the Judge's labels, rates, and verdicts become the foundation for forced pick side selection — `tier_flip` labels lock picks, `no_trust`/`injury_mismatch`/`multi_tag` labels trigger flips, and the rate cascade determines which direction has more empirical support.

**The Judge must be scrupulous.** Every pick from the model and every recommendation from the fade evaluator passes through the Judge's scrutiny. The Judge doesn't rubber-stamp — it re-evaluates. It applies 16+ modifier classes: direction gates, ECS floors, tight-line boosts, away-dog penalties, defense quality, H2H matchup history, ATS tier classifications, final-pick team WR, NET gap analysis, fade destination health, injury asymmetry, large-spread adjustments, and blacklist coherence checks. Each modifier is empirically calibrated from historical outcome data — not intuition, not theory, not vibes.

**The Judge's wisdom compounds.** Rate modifiers that individually shift a fade rate by ±3–5 points can stack to swing a verdict by 20+ points. A game that starts at base rate 52 (coin flip territory) might land at 78 after the cascade reveals: fading toward an elite defense (+4), elite ATS tier team (+3), historically hot final-pick WR (+4), opponent has high injury burden (+5), NET gap confirms fade direction (+5), and the model opposes market direction (+5). That's not one signal — it's six independent signals converging on the same truth. The Judge sees all of them simultaneously.

**The Judge owns the outcome.** When a forced bet loses, the question is always: did the Judge have enough information? If a signal existed that the Judge didn't evaluate, that's an architecture gap to close (v4.6.1 closed four such gaps). If the Judge saw everything and still picked wrong, that's variance — irreducible noise in a probabilistic game. The goal is never 100% accuracy; it's ensuring every decision was made with maximum available information.

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

**Bet tracker columns:** Date, Matchup, Pick, MarketLine, FairLine, Edge, ECS, Stake, Notes (SHADOW:TAG), Result, ModelVersion. Add variant-specific columns (e.g., `Risky_Bet`, `Risky_ToWin` for alternative sizing strategies) rather than creating separate tracker files — see lesson #16.

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
11. **Never reimplement decision logic in diagnostic/replay tools.** Extract pure functions and call them from all consumers. Reimplementing the same computation in a test or replay script creates a parallel copy that silently diverges as the engine evolves — producing false confidence. Discovered in NBA v4.0: diagnostic was missing 6 rate modifiers and produced different verdicts than the engine on identical inputs. Fix: extract `check_guard_rails()`, `judge_verdict()`, `apply_shadow_override()` as shared functions.
12. **Audit blacklists regularly with hard data.** Blacklists decay. Teams improve mid-season, or your model learns to price them correctly. Run a per-team evidence table (model-pick WR%, fade-from WR%, fade-to WR%, shadow WR%) at least every 2 weeks. Remove teams where fading is losing money; add teams where the model consistently picks wrong AND fading from them is profitable. Discovered in NBA v3.67: PHX and MIL had been blacklisted but fading from them was 17–25% WR — the blacklist was actively destroying value.
13. **Promote display-only indicators to guard rails.** If you built a scoring system that displays risk but doesn't block bets, you're wasting the signal. Wire it into the decision layer with a configurable threshold. Shadow-test at the threshold for 20+ bets before making it a hard block. Discovered in NBA v3.67: blowout risk scoring existed for months as display-only — games could score 85 blowout risk and still place real bets. Extracting it as a pure function and wiring `SHADOW:BLOWOUT_RISK` into guard rails turned dead information into actionable protection.
14. **Build batch mode from day one using extracted functions.** Once your decision logic is extracted into pure functions (Section 8.3), building a batch/non-interactive runner is trivial — it just loops through all games and calls the same functions. This validates the architecture (if batch mode requires reimplementation, your extraction is incomplete). Use distinct entry ID prefixes (`B` for batch, `G` for interactive) to track provenance in bet trackers. Batch mode eliminates session drift (early vs late picks using different data freshness) and reduces a 20-30 minute interactive session to seconds. Discovered in NBA v4.1.1: batch generator calls are identical pipeline — zero logic divergence confirmed via replay.
15. **Momentum/form tracking adds no value when the market anchor is strong.** With market anchor weight ≥ 0.55, Vegas lines already capture streaks, momentum, and recent performance far more comprehensively than any simple ATS streak counter. Sequential same-team bet outcomes show 50/50 continuation rates (tested on 84 pairs). Engineering time is better spent on guard rail precision and fade calibration. Discovered in NBA v3.67 analysis: zero residual form signal after market anchor.
16. **Never create parallel tracker files for variant strategies — use columns.** If you add an alternative bet-sizing mode (e.g., quarter-Kelly on every pick), store variant data as extra columns in the existing tracker (`Risky_Bet`, `Risky_ToWin`) rather than a separate file. Parallel files create sync bugs: any code that updates one file must remember the other, and report logic must merge two sources. Columns in one file mean one write path, one update path, one report source. Empty columns for non-applicable rows are cheap; paired-file sync bugs are not. Discovered in NBA v4.3.1: dual tracker caused interactive updater to forget the paired risky file, and post-mortem shadow mask applied differently to each file.
17. **Calibrate confidence scores with historical outcome feedback.** Static confidence formulas (blending ECS, edge, action type) produce plausible-looking scores but drift from reality. Add a feedback loop: scan all historical trackers, compute actual WR by action type, and apply a ±adjustment to the confidence formula. This catches systematic biases — e.g., shadow forced picks at 43.8% WR should get −9 confidence vs breakeven, not the same score as clean bets at 56.8%. The adjustment function should be simple (each 1% above/below breakeven = ±1 point, capped) and the scan should require a minimum sample size (≥10 games) to avoid noise. Discovered in NBA v4.4.0: forced pick confidence was identical across action types despite 13-point WR spread between best (SHADOW_OVERRIDE 60.3%) and worst (SHADOW 43.8%).
18. **Every promotion rule needs a confidence floor.** Any rule that promotes a blocked pick to an actionable bet (SKIP→ALIGNED, SHADOW→REAL, DISAGREE→AGREE) must gate on the model's own confidence score (ECS or equivalent). Without it, rules designed for "model is right but edge is small" also fire for "model has no idea" — promoting garbage picks alongside good ones. The floor should match your NO_TRUST tier cutoff (e.g., ECS < 40). Similarly, forced-pick auto-flips must respect structural signals (lopsided detection identifying the strong team) over generic confidence thresholds. Discovered in NBA v4.4.2: LOW_EDGE auto-flip and LOW_EDGE-only promote had no ECS floor, promoting a NO_TRUST pick (ECS 35, edge 2.9) to a real bet that lost by 13. Adding min_ecs: 40 improved actionable WR by +1.5pp with no regression.

19. **Gate auto-flip rules on objective independent signals.** If you have a rule that reverses the model's pick (auto-flip), don't let it fire on a single soft signal (e.g., "confidence is low, so flip"). The flip target might still be wrong. Add a second, independent, objective gate — e.g., only flip when the picked team's power rating is objectively much worse than the opponent's. The gate prevents flips where the model picked correctly despite low confidence. Discovered in NBA v4.4.7: NO_TRUST auto-flip fired on every ECS < 40 game (45.8% WR). Adding a NET rating gap gate (< -3) to confirm the model pick was actually weak raised flip WR to 62.5% and games blocked by the gate hit 76.9%.
20. **Use dynamic classification from live data instead of static lists.** Any time you maintain a manual list of teams/players/situations (e.g., "strong teams" vs "weak teams"), automate it from the underlying rating data with configurable thresholds. Static lists get stale immediately — mid-season form changes make them wrong within weeks. Dynamic classification reads live data, uses threshold parameters from config, and falls back to static lists when data is unavailable. Add a preflight check that diffs live classification vs the static snapshot to surface movements. Discovered in NBA v4.4.7: static MEN/BOYS tiers had 5 top-10 NET teams classified as BOYS and 4 bottom-10 as MEN. Dynamic classification from live NET ratings with configurable thresholds (MEN ≥ 1.8, BOYS ≤ -4.0) corrected this automatically.
21. **Use head-to-head matchup history as a multi-layer signal.** Season series records between specific teams capture intangible dynamics (matchup advantages, coaching adjustments, defensive schemes) that pure efficiency metrics miss. Integrate H2H at multiple layers: (a) fair line component (small adjustment, e.g. ±0.5–2.0 pts), (b) ECS signal (penalty when picking against dominant team), (c) auto-flip gate (block reversals that oppose matchup history). The fair line effect alone is marginal, but the interaction with other systems (ECS dropping below a gate threshold, blocking an auto-flip) creates cascading value. Require minimum sample (≥2 games) to prevent noise. Discovered in NBA v4.5.0: H2H integration converted 2 losses to wins on April 9 via two different mechanisms — ECS penalty caused a gate failure (BOS@NYK), and auto-flip gate blocked a bad reversal (LAL@GSW). Combined: 4W-2L → 6W-0L.
22. **Keep parallel decision pipelines at signal parity.** If your engine has two pipelines making decisions about the same games (e.g., one for "should we bet?" and another for "which side forced?"), the second pipeline must evaluate the same signal classes as the first — otherwise it makes uninformed decisions the primary pipeline would have caught. Express the signals differently (confidence adjustments vs rate modifiers) but ensure the same information flows into both paths. Config-gate each independently for surgical rollback. Discovered in NBA v4.6.1: forced pick confidence used ~11 signals while Judge had 35+; April 14 playoffs 0-2 forced bets where model was right. Injecting 4 missing signal classes closed the gap without breaking existing protections (676-game replay: zero regressions).
23. **Data-reject fixes that harm profitable guardrails.** When a proposed fix looks logically correct but backtesting shows it inverts a profitable existing pattern, reject it — the existing mechanism may capture the same information through a different pathway (e.g., market alignment embeds team strength that a model-pick gate would try to replicate). Always validate against the full historical dataset. Discovered in NBA v4.6.1: AGREE+WITH market protection was 21W-9L (70.0%); proposed model-pick gate would have inverted it to 9W-21L (30.0%).
24. **Prevent signal double-counting across layers in multi-layer cascades.** When a decision cascade has a base rate category (e.g., "injury mismatch" at rate 42) AND tag-based overrides (e.g., "INJURY_QUALITY" at rate 60), the override can contradict the base rate if both encode the same underlying signal. The base rate says "fading injury-mismatched games is sub-breakeven" — but the injury-tagged override boosts the rate upward, effectively double-counting the injury signal. Then stacking modifiers (away-dog +4, final-pick WR +4/+5) push past the decision threshold. Fix: when a base rate category already prices in a signal class (injuries, matchup quality, etc.), block tag overrides from the SAME signal class from overriding upward. Non-overlapping tag overrides (confidence conflicts, daily caps, blacklist) remain unaffected. Config-gate with an independent toggle so it can be disabled if it regresses. Discovered in NBA v4.8.0: injury_mismatch base rate 42 + INJURY_QUALITY override 60 + modifiers → rate 73 (above threshold 70) → wrong FADE on GSW +5.5 (GSW won outright by 10.5). Adding injury_mismatch_cap blocked IQ override → rate 45 → correct ALIGNED verdict. Zero regressions across full corpus.

---

## 8. Clean Architecture Pattern (ONE Source of Truth)

Proven in NBA v4.0. Apply this pattern to every new prediction engine from day one.

### 8.1 Core Principle

**Every decision computation must exist in exactly one place.** The interactive engine, diagnostic tools, replay harnesses, and any future consumer all call the same extracted functions. Zero reimplementation, zero drift.

### 8.2 Module Separation

Organize code into three layers with strict dependency rules:

```
┌─────────────────────────────────────────────────┐
│  ANALYTICS (pure logic, no I/O)                 │
│  - Fair line calculation                        │
│  - Guard rail evaluation                        │
│  - Fade evaluator / Judge arbitration           │
│  - Shadow override logic                        │
│  - Edge & ECS scoring                           │
│  All functions: data in → verdict out            │
│  No print(), no input(), no file writes          │
├─────────────────────────────────────────────────┤
│  ENGINE UI (interactive I/O layer)              │
│  - User prompts, display, menu navigation       │
│  - Market line input, bet confirmation           │
│  - Bet tracker CSV writes                        │
│  - Calls analytics functions, formats output     │
├─────────────────────────────────────────────────┤
│  TOOLS (diagnostic, replay, tuning)             │
│  - fade_judge_diagnostic: calls judge_verdict()  │
│  - replay harness: calls all extracted functions │
│  - auto_tune: reads fair_line_logs              │
│  - post_mortem: reads bet_trackers              │
│  All tools call analytics — never reimplement    │
└─────────────────────────────────────────────────┘
```

**Dependency rule:** Tools → Analytics ✅ | Tools → UI ❌ | UI → Analytics ✅ | Analytics → UI ❌

### 8.3 Function Extraction Pattern

Every decision point in the pipeline should be an extracted function with this shape:

```python
def check_guard_rails(edge, ecs, team, opponent, star_tax, ..., model_params, guard_rails_config):
    """Pure function. Returns dict with tags, shadow reasons, verdict."""
    tags = []
    # ... all logic here ...
    return {
        "tags": tags,
        "is_shadow": len(tags) > 0,
        "shadow_reasons": tags,
        "details": {...}
    }
```

**Rules for extracted functions:**
1. **Pure** — No side effects. No file I/O, no global state mutation, no print().
2. **Config as parameter** — Accept `model_params` and `guard_rails_config` as arguments. Never read config files inside the function.
3. **Rich return dicts** — Return everything a consumer might need: verdict, reasons, intermediate values, rate breakdowns. Transparency enables debugging without reading source code.
4. **Fallback defaults** — Every config read uses `config.get(key, default)` so the function works even with incomplete configs.

### 8.4 Logging Discipline

Replace all `print()` in analytics with Python's `logging` module:

```python
import logging
logger = logging.getLogger(__name__)

# In functions:
logger.info("Guard rail fired: %s (edge=%.1f, ecs=%d)", tag, edge, ecs)
logger.debug("Judge rate cascade: base=%d, modifiers=%s, final=%d", base, mods, final)
```

**Why:**
- `print()` in the analytics layer couples logic to the interactive console. Replay tools and diagnostics don't want console spam.
- `logging` lets each consumer control verbosity: `logging.DEBUG` for diagnostics, `logging.WARNING` for production, `logging.CRITICAL` for silent replay.
- Log messages become a debugging trail without touching the code.

### 8.5 Replay Harness Design

Every engine should have a **non-interactive replay harness** that:

1. **Reads fair_line_log CSVs** — component-level data already logged during live sessions.
2. **Calls the same extracted functions** — `check_guard_rails()` → `judge_verdict()` → `apply_shadow_override()`.
3. **Resolves W/L** from `game_results_cache.json` — compares pick vs actual result.
4. **Diffs against bet_tracker CSVs** — flags any mismatch between replay verdict and what was logged live.
5. **Reports layer-by-layer WR** — Clean picks, SHADOW, FADE, OVERRIDE, each with W/L and WR%.

```
Replay pipeline:

fair_line_log_*.csv          bet_tracker_*.csv
       │                            │
       ▼                            │
  Reconstruct inputs                │
       │                            │
       ▼                            │
  check_guard_rails()               │
  judge_verdict()                   │
  apply_shadow_override()           │
       │                            │
       ▼                            ▼
  Replay verdicts  ──── DIFF ──── Logged verdicts
       │
       ▼
  game_results_cache.json
       │
       ▼
  W/L resolution → layer WR% report
```

**Why this matters:** When you change a parameter (e.g., raise `fade_threshold` by 5), you can replay the entire history through the REAL engine code and see exactly which bets flip and what the new WR would be — before shipping the change.

### 8.6 Starter File Template

For a new sport, create these files on day one:

| File | Layer | Contents |
|---|---|---|
| `{sport}_analytics.py` | Analytics | Fair line calc, guard rails, fade evaluator, Judge, shadow override — all pure functions |
| `{sport}_engine_ui.py` | UI | Interactive CLI, display, bet logging — calls analytics |
| `{sport}_batch_picks.py` | UI | Batch pick generator — runs full pipeline for every game on a slate, pre-fills bet tracker. Same extracted functions as interactive engine, zero reimplementation |
| `{sport}_engine_replay.py` | Tools | Non-interactive replay harness — calls analytics |
| `{sport}_diagnostic.py` | Tools | Fade/Judge calibration replay — calls analytics |
| `model_config.json` | Config | All tunable parameters, guard rail thresholds, version |
| `auto_tune.py` | Tools | Ridge regression parameter optimizer |
| `post_mortem.py` | Tools | End-of-session performance report |
| `preflight_check.py` | Tools | Pre-session data quality validation |

### 8.7 Architecture Validation Checklist

Before shipping any engine, verify:

- [ ] **Zero decision logic in UI** — UI only formats and displays; all computation in analytics
- [ ] **Zero decision logic in tools** — Diagnostic and replay call analytics functions; never reimplement
- [ ] **Zero print() in analytics** — All output uses `logging`
- [ ] **Config as parameter** — No function reads config files directly; config passed in
- [ ] **Rich return dicts** — Every function returns enough data to debug without reading source
- [ ] **Replay produces identical verdicts** — Run replay against bet_tracker; zero unexplained diffs
- [ ] **Batch mode produces identical verdicts** — Batch runner calls same functions as interactive; zero logic divergence
- [ ] **Fair line log captures all components** — Cannot backtest what you didn't log
- [ ] **Shadow bets logged at $0** — Guard rail accuracy requires tracking what was blocked

### 8.8 Forced Bet Optimization Pattern (v4.5.3)

When your engine has a `--force` mode (confirm ALL games including shadows), the forced pick path needs its own optimization layer. The Judge/fade evaluator is designed for BET/SHADOW contexts — it doesn't know about the "gun-to-head" constraint where you MUST bet every game.

**Key principles:**
1. **Triple consensus protection** — When model, fade evaluator, AND market all agree on the same side, skip all flip mechanisms. This is your highest-WR signal (78%+ in NBA). Don't second-guess it.
2. **Injury-aware auto-flips** — Auto-flip rules that bypass rate modifiers (e.g., MAN→BOY tier flips) must check whether "strength" is current. A team's season NET/ATS record is stale when key players are OUT. Gate auto-flips with star tax thresholds.
3. **Edge-based safety nets** — When model edge is near zero (< 2 pts), the model has no directional signal. A last-resort flip to the market favorite is better than trusting a coin-flip model pick. BUT require minimum market spread (|mkt| ≥ 3) to filter pick'em noise.
4. **Blacklist coherence** — If the fade evaluator independently determined a team is unreliable ATS (blacklisted), don't let forced-pick flip mechanisms send picks TO that team.
5. **SKIP side selection** — When the Judge produces SKIP (low conviction), don't blindly default to the model pick. Use signals: if NET_gap_suppress fired, flip to the NET-stronger side. If confidence is near zero, use market favorite as tiebreaker.
6. **Config-gate everything** — Every forced pick mechanism gets an independent `enabled` toggle and tunable thresholds. This allows surgical rollback if a mechanism regresses.
7. **Track `_original_forced`** — Before any flip mechanism fires, save the original forced pick. Use this to detect whether ANY prior flip already acted — prevents cascading double-flips.

**Mechanism ordering matters:**
- **Protection checks first** (triple consensus) — prevent good picks from being wrongly flipped
- **Context-aware flips in the middle** (NT flip, market alignment, low-confidence flip)
- **Safety nets last** (edge<2 flip) — only fires when nothing else did

**v4.5.3 lessons (replay-validated, 333 games across 43 dates):**
8. **When the Judge says model is wrong, trust it.** For FADE games (Judge faded the model), forced pick MUST use the fade direction, not model direction. NBA evidence: model-dir = 30.2% vs fade-dir = 69.8% (n=43). The original n=13 sample was misleading — always validate with full replay before shipping.
9. **Judge labels reveal forced-pick failure modes.** When the Judge assigns low-confidence labels (no_trust, injury_mismatch, multi_tag) but produces SKIP, the model pick is empirically wrong. Flip forced pick to the opponent for these labels. NBA evidence: combined 42W-20L (67.7%) after flipping.
10. **Small samples kill.** v4.5.2 shipped fade_forced_reversal based on n=13 (30.8%). Full pipeline replay (n=43) proved the OPPOSITE direction was correct. Always build cache-faithful replay infrastructure (archive all daily caches) and validate against the full dataset before committing directional changes.
11. **Bridge signal depth between parallel decision pipelines.** When two pipelines make decisions about the same games (e.g., Judge picks vs forced picks), keep their analytical depth in parity. If the primary pipeline evaluates 16 modifier classes (defense quality, ATS tiers, final-pick WR, injury asymmetry), the secondary pipeline must see the same signals — even if expressed differently (confidence adjustments instead of rate modifiers). Without parity, the secondary pipeline makes blind decisions that the primary would have caught. Config-gate each modifier class independently (`enabled` toggle + thresholds) so you can disable any that regress. NBA evidence (v4.6.1): forced picks lacked 4 signal classes the Judge used; injecting them as `_forced_analytical_modifiers()` into `compute_forced_confidence()` closed the gap. Data-rejected one proposed fix (AGREE+WITH gate: 21W-9L → 9W-21L) because the existing protection captured the same information through market pricing.
12. **Data-reject proposals that harm working protections.** When a fix looks logically sound but backtesting shows it destroys a profitable existing pattern, reject it — the existing pattern may capture the same information through a different mechanism (e.g., market alignment captures team strength that a model-pick gate would duplicate). Always test proposed changes against the full historical dataset before shipping. NBA evidence: AGREE+WITH market protection was 21W-9L (70.0%); proposed model-pick gate would have inverted it to 9W-21L (30.0%).
