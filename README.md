# Sports Prediction Engine — Copilot Skill

A reusable [VS Code Copilot Skill](https://code.visualstudio.com/docs/copilot/copilot-customization) that captures the architecture, methodology, and tuning discipline for building **spread/line prediction engines** across any sport.

## What's Inside

**[SKILL.md](SKILL.md)** — 275-line reference covering:

1. **Architecture Pattern** — The 6-stage pipeline: Fair Line → Market Anchor → Edge → Guard Rails → Fade Evaluator → Judge
2. **Component Mapping** — How each stage maps across NBA, NFL, MLB, NHL, and Soccer
3. **Config-Driven Design** — `model_config.json` template with helper function pattern
4. **Data Architecture** — File naming conventions, bet tracker schema, fair line log structure
5. **Tuning Methodology** — 5-step backtest validation process, key metrics (ATS WR, bias, MAE, ROI)
6. **Sport-Specific Starter Guides** — NFL, MLB, NHL, Soccer with efficiency metrics, key player models, HFA magnitudes, situational factors, and recommended guard rails
7. **Anti-Patterns** — 8 lessons learned (market anchor is mandatory, don't optimize CLV, don't skip component logging, etc.)

## Origin

Distilled from the [NBA Prediction Engine](https://github.com/rockit2ya/nba_prediction_engine) (v3.60, 60.5% ATS over a full season). Every pattern in this skill was validated through backtesting against 371+ real bets.

## Installation

This skill is designed to live at `~/.agents/skills/sports-prediction-engine/` so Copilot auto-discovers it in every workspace.

**If cloned elsewhere**, symlink it:

```bash
ln -sfn ~/Documents/GitHub/sports-prediction-skill ~/.agents/skills/sports-prediction-engine
```

## Usage

Copilot will automatically invoke this skill when you ask about:
- Building a prediction engine for any sport
- Setting up guard rails, fade evaluators, or market anchoring
- Backtesting parameter changes
- Calibrating Judge rates

## License

MIT
