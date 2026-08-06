# Two projects, built to be questioned

**→ [nimb-ou.github.io](https://nimb-ou.github.io)** — the interactive writeup. Public, works
anywhere, no login.

Both repos exist to back specific claims on a CV. The design goal throughout was not to
produce impressive numbers — it was to produce numbers that survive being asked about.

| | [loan-underwriting-copilot](https://github.com/nimb-ou/loan-underwriting-copilot) | [power-price-alpha](https://github.com/nimb-ou/power-price-alpha) |
|---|---|---|
| **What** | Credit scoring + SHAP + reason codes + RAG over precedents + a grounded LLM agent | GB half-hourly price forecasting + conformal intervals + battery-arbitrage backtest |
| **Stack** | XGBoost, ChromaDB, Gemini 2.5 Flash, SHAP, FastAPI, Docker | XGBoost, statsmodels, pandas, three public APIs |
| **Tests** | 226 (5 skip without an LLM key) | 201 |
| **Course** | 10 notebooks + 5 markdown lessons + solutions | 14 notebooks + 2 markdown lessons + solutions |
| **Writeup** | [nimb-ou.github.io/credit.html](https://nimb-ou.github.io/credit.html) | [nimb-ou.github.io/power.html](https://nimb-ou.github.io/power.html) |
| **One command** | `make all` | `make all` |

## What each one measured

**Credit.** ROC-AUC **0.79** (CV 0.7883 ± 0.0332, held-out 0.7858, 95% CI [0.721, 0.845]).
The resume said 0.81; that figure came from an untuned configuration lucky on one split and
does not reproduce.

The better result is not the AUC at all. At scikit-learn's default 0.5 threshold the model
costs **0.77 per applicant** — *worse than declining every single applicant* (0.70). At the
cost-optimal threshold, **0.49**. The threshold is worth more than the modelling, and that
generalises far beyond this dataset.

**Power.** MAE improvement over a seasonal-naive baseline: **22%**.

The declared baseline gives 29.9%, but it is not the hardest one available, and the weather
features come from an archive rather than a real forecast. Against the strongest baseline:
22.1%. With every weather feature removed: 22.1%. Two independent stress tests converging is
why 22% is the number to quote — and it still exceeds the 18% originally claimed.

## The four things worth showing

1. **`data/calendar.py` and `tests/test_calendar.py`** (power). A GB day has 46, 48 or 50
   settlement periods. Two sources express time differently. Joining them naively corrupts two
   days a year and every lag that crosses them, silently. 32 tests pin it.

2. **`agent/guards.py` and `tests/test_guards.py`** (credit). Prompting an LLM for
   faithfulness cannot be verified; catching unfaithfulness can. Every number in an answer must
   appear in its context, every case citation must have been retrieved, protected attributes
   cannot be cited as reasons. Half the tests check the *opposite* direction, because a guard
   that rejects correct answers gets switched off.

3. **`tests/test_no_leakage.py`** (power). Day-ahead means the forecast is made at 11:00 on
   T−1, so `price_lag_1d` is not knowable. Eighteen tests assert no feature's source timestamp
   reaches past its decision time.

4. **Both `RESUME_CLAIMS.md` files.** Every claim maps to a command, a file and a number that
   regenerates on every push.

## Things that went wrong, and are in the git history

- **A Sharpe of 50.** The first strategy design traded `forecast − naive` against
  `realised − naive`. No instrument has that payoff, so the P&L was synthetic — and the
  annualisation compounded it by treating 48 half-hours as independent bets. Replaced with a
  battery, where every leg is a real trade.
- **`model_loaded: false`.** The Docker container exposed a path-resolution bug: config
  resolved via `Path(__file__).parents[2]`, correct under an editable install and wrong inside
  a wheel. The same bug then bit the second repo, twice, by writing a test run's output into
  the real checkout.
- **A wrong claim in my own claims file.** It said one baseline beat the declared one; the
  wrong baseline was named. Fixing it revealed the declared baseline was not the hardest, which
  is why the headline moved from 30% to 22%.
- **Two CI failures on first push.** A missing module the Makefile had always referenced, and a
  hardcoded `.venv/bin/python` that only existed on this machine.

None of these were caught by reading the code. They were caught by running it somewhere else.

## Running them

```bash
cd ~/Desktop/First_project/loan-underwriting-copilot && make venv && make all
cd ~/Desktop/First_project/power-price-alpha     && make venv && make all
```

Both repos are private on GitHub. Make them public when you want to share them —
`gh repo edit --visibility public`.

## Where they would go next

**Credit:** build the case store from data the model was *not* trained on, which is the
experiment that would turn the RAG layer from explanatory to predictive. Then counterfactuals,
then reject inference.

**Power:** charge battery degradation per cycle, relax the one-cycle-a-day constraint, and
replace the point forecast with a bid curve.


---

## Added since the first pass

**Credit — fairness, measured rather than assumed.** The system already hid protected
attributes from the officer view and blocked the agent from citing one as a reason. Neither is
a fairness result. `models/fairness.py` now measures per-group approval rates, bad rates and
AUC at the served threshold, and tests them by permutation.

The first version used a bootstrap on the min/max approval-rate ratio and reported all three
protected attributes as conclusively disparate. That was the *test* being wrong: `min` over a
set of noisy group rates is biased downward, and a bootstrap centred on a biased estimate
stays biased. After the fix: one attribute untestable (8 people), one gap explained by
realised default rates, and **one real finding** — the age gap survives restricting to
applicants who did not default. The ablation prices the fix at 0.008 AUC, inside the
confidence interval, and the disparity does not survive the removal.

**Credit — adverse-action reason codes.** A SHAP table is not the notice a declined applicant
is owed. `explain/reason_codes.py` produces the regulated artifact: risk-increasing factors
only, capped at four, each quoting the applicant's own value, never a protected attribute.
Served on `/explain` and shown beside the SHAP table in the UI, because confusing the two
means mailing someone log-odds.

**Power — the interval that did not mean what it said.** Quantile forecasts fitted under
pinball loss cover **50.2%** of outturns against a nominal 80%. That is not a broken
estimator: on the window it was fitted on, coverage is 0.810, and across the same split the
standard deviation of prices rose from 14.8 to 21.3 GBP/MWh. The market moved.

Conformal calibration takes coverage to **75.6%** and — the number that matters — Winkler from
208.1 to 169.5. It does not reach nominal, because exchangeability fails on a non-stationary
series, and that shortfall is reported rather than hidden.

**Power — forecast results by regime.** Answers the obvious challenge. The improvement is
**16.0%** calm, **27.6%** crisis, **36.6%** post-crisis — largest in the most recent regime,
so it is not an artifact of the gas crisis.

**Both — the course became usable alone.** A syllabus with prerequisites and reading order,
and worked solutions for every exercise. Where an exercise is cheap to run, the solution
carries a measured number: the single-rule credit policy costs 0.912 per applicant against
0.700 for declining everyone, and reconstructing `foreign_worker` from the other attributes
scores 96.0% against a 96.3% majority-class baseline — worse than a constant, which is the
trap the exercise is built around.
