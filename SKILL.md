---
name: dlt-skill
description: |
  This skill should be used whenever the user works with the 大乐透 (Super Lotto)
  multi-strategy prediction tool at C:\project\dlt\. It covers running predictions,
  feeding back actual draw results, adding or modifying prediction strategies,
  analyzing backtest performance, and managing the feedback JSON data.
  Use this skill when the user mentions 大乐透, DLT, lottery prediction, dlt_predict.py,
  彩票预测, 添加策略, 回测, 开奖反馈, 策略排名, 杀号, 选号, or any
  lottery-related data analysis — even if they don't say the word "skill."
---

# 大乐透多策略预测技能

## Before Implementation

When asked to create or modify anything in this project, gather context before writing code:

| Source | What to gather |
|--------|---------------|
| **Codebase** | Read the target insertion area in `dlt_predict.py` — line numbers in this skill are approximate, verify against actual code. Check existing strategy patterns nearby. |
| **Conversation** | Extract requirements the user stated: strategy name, core idea, window size, any special constraints. |
| **Skill references** | Read `references/strategies.md` for strategy contract and existing tool functions. Read `references/confidence-system.md` for scoring mechanics if the change affects confidence. Read `references/external-sources.md` for research sources when designing new strategies. |
| **Data files** | Read `dlt_feedback.json` to understand current feedback state. Read `dlt_100.csv` header if CSV format is relevant. |

## Required Clarifications

When adding a new strategy, ask the user if these are unclear:

1. Strategy name and core insight — what pattern does it exploit?
2. Lookback window — how many periods of history to use?
3. How to generate 3 distinct bets from the same logic?

Do not ask about: file locations (they're fixed), existing tool functions (in references), or numbering conventions (follow existing patterns).

## Optional Clarifications

- Whether to add a custom confidence signal-strength branch (otherwise falls back to generic 100-period frequency)
- Whether the strategy needs a fixed random seed for reproducibility

## Core Operations

### Run prediction

```bash
cd C:\project\dlt && python dlt_predict.py
```

Pipelines: load CSV → 30-period backtest all 15 strategies → rank → predict 45 bets (15×3) with confidence → save to `dlt_feedback.json` → show historical stats and trend matrix.

### Feed back draw results

```bash
python dlt_predict.py --feedback <period> <f1> <f2> <f3> <f4> <f5> <b1> <b2>
```

Computes per-strategy hits, shows star-rating (* hit / o miss), names best strategy, persists to `dlt_feedback.json`.

### Install optional ML dependencies

```bash
pip install -r requirements.txt
```

Only needed for the AI集成学习 strategy (XGBoost + RandomForest). Other 14 strategies are pure stdlib.

## Strategy System

Every strategy follows this contract (details in `references/strategies.md`):

```python
def strategy_xxx(rows, **kwargs) -> list[tuple[list[int], list[int]]]:
    """rows[0] is latest period. Returns exactly 3 bets of (sorted_front_5, sorted_back_2)."""
```

Existing 15 strategies (full details, including window sizes and algorithm descriptions, in `references/strategies.md`):

| Batch | Strategies |
|-------|-----------|
| Original (3) | 热号, 冷号, 均衡 |
| Second (5) | 012路, 尾号, 分区, 回补, 综合 |
| Third (7) | 马尔科夫链, 遗传算法, AI集成学习, 杀号, 黄金斐波那契, AC值同期对比, 六行六列图 |

Tool functions available for reuse: `freq`, `missing`, `last_appear`, `route_012_counts`, `tail_counts`, `zone_counts`, `is_prime`, `has_consecutive`, `count_repeats`, `get_zone`, `_pick_sliding_bets`. Signatures in `references/strategies.md`.

## Adding a New Strategy

### Must Follow

- [ ] Strategy returns exactly 3 bets, each `(sorted(front_5), sorted(back_2))`
- [ ] Front numbers in 1-35, back numbers in 1-12, no duplicates within a bet
- [ ] Historical data is the primary signal source — not pure randomness
- [ ] 3 bets have distinct selection logic (different allocations/weights/thresholds, not just re-seeding)
- [ ] If using randomness, seed with `int(PREDICT_PERIOD) + offset` for reproducibility
- [ ] Complex combinatorial searches have a candidate limit (≤20000) to keep runtime reasonable
- [ ] Registered in all 4 required locations (see below)
- [ ] Run `python dlt_predict.py` after all changes to verify no errors

### Must Avoid

- Pure random selection without any historical signal
- Duplicate bets (3 identical front+back combos)
- Hardcoding a specific period's data or numbers
- Modifying the strategy contract (return format, input assumptions)
- Changing `FRONT_POOL`, `BACK_POOL`, or shared constants without understanding impact

### Registration Checklist

After writing the strategy function (insert before `calc_hits`), register it in exactly these 4 places:

1. [ ] **Backtest loop** (~L1464): add `("策略名", strategy_xxx, {'window': NN}),`
2. [ ] **Prediction configs** (~L1634): add same entry
3. [ ] **Trend table** (~L1792): add strategy name to `all_strategy_names`
4. [ ] **Confidence branch** (~L1518, ~L1695, optional): add `elif` in `compute_confidence` and `_get_num_confidence` if custom signal logic needed

Line numbers are approximate — verify against actual code before inserting. Read the area around the target line to find the exact insertion point.

## Confidence & Backtest System

Detailed formulas and data structures in `references/confidence-system.md`. Quick reference:

**Confidence** (6 sub-scores): Backtest avg (40%) + Signal strength (25%) + Structure check (15%) + Repeat match (10%) + Position deviation (5%) + Consecutive bonus (5%)

**Backtest ranking**: Composite = mean×50% + stability×30% + high-score-rate×20%, over 30-period sliding window.

## Error Handling

| Scenario | Action |
|----------|--------|
| CSV not found | Check `CSV_PATH` at top of `dlt_predict.py` — it's hardcoded absolute path |
| `dlt_feedback.json` corrupted | Delete it and re-run prediction to regenerate; no migration needed |
| ML deps not installed | AI集成学习 strategy silently falls back to `strategy_hot` — no crash |
| Invalid period in --feedback | Script prints error and exits with code 1 |
| No prediction record for period | Script prints "无预测记录" and exits |
| Strategy returns <3 bets | Prediction loop may crash; always verify 3 bets in your strategy |
| Invalid --feedback arguments | Script expects exactly 9 numbers after period; front 1-35 each, back 1-12 each. Validate before running: `len(sys.argv) == 10`, `all(1 <= n <= 35 for n in front)`, `all(1 <= n <= 12 for n in back)` |

## Automation Workflows

### Before each draw

1. Update CSV if new data available
2. Update `PREDICT_PERIOD` to next period number
3. Run `python dlt_predict.py`
4. Review backtest rankings for strategy momentum
5. Note top-confidence bets

### After each draw

1. Obtain official draw numbers
2. Run `python dlt_predict.py --feedback <period> <9 numbers>`
3. Review per-strategy hit rates
4. Check trend matrix for long-term patterns

## Data Reference

| When you need | Read |
|---------------|------|
| CSV format | `C:\project\dlt\dlt_100.csv` (first 3 lines for columns) |
| Feedback history | `C:\project\dlt\dlt_feedback.json` |
| Strategy research sources | `C:\project\dlt\strategies_research.md` or `references/external-sources.md` (17 methods with URLs, implementation status) |
| Design specs | `C:\project\dlt\docs\superpowers\specs\` |

## External Resources

| Resource | URL | Notes |
|----------|-----|-------|
| 大乐透官方规则 | https://www.lottery.gov.cn/dlt/ | Official game rules |
| 历史开奖数据 | https://www.lottery.gov.cn/kj/kjlb.html | Official draw results |

## Quick Reference

| Task | Command |
|------|---------|
| Predict | `python dlt_predict.py` |
| Feedback | `python dlt_predict.py --feedback <period> <9 nums>` |
| Install ML deps | `pip install -r requirements.txt` |
| View data | Read `dlt_100.csv` |
| View history | Read `dlt_feedback.json` |
| View research | Read `strategies_research.md` |
| Strategy details | Read `references/strategies.md` |
| Scoring details | Read `references/confidence-system.md` |
| Research sources | Read `references/external-sources.md` |