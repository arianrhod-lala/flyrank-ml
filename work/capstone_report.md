# Capstone Report — CTR / Engagement Opportunity Scoring

- **Author:** Ian Lesigues
- **Lane:** CTR / Engagement Opportunity Scoring
- **Repo:** https://github.com/arianrhod-lala/flyrank-ml
- **Date:** August 17, 2026

> Copy this file to `work/capstone_report.md` and fill in the four placeholders above.

## 1. Problem framing

Content and SEO teams maintain far more URLs than they have hours to refresh, and default to
a rank-based guess for where to spend that time. The unit of analysis here is the **page**
(one row per URL per client). The output is a **predicted CTR gap** — expected CTR minus
actual CTR, given the page's rank tier — which becomes a **rank-ordered list**. The action a
human takes from it is picking which pages a writer or SEO specialist works on this sprint.
A wrong call costs real hours: refreshing a page that was already at its ceiling, or skipping
one that would have converted visibility into clicks. ML helps here because a position-tier
rule treats every page in a tier as interchangeable; it can't separate a page underperforming
its tier badly from one already earning everything its position can give it. That's exactly
the non-linear signal a model trained on more than one feature can pick up and a single-rule
baseline can't.

## 2. Data safety

**Used:** `content_refresh_anonymized.csv`, the FlyRank anonymized starter release (a
mid-panel slice of the gated Hugging Face warehouse), filtered to pages with at least 500
impressions in the 90-day window and a valid (positive) average position.

**Feature columns:** `impressions_90d`, `avg_position`, `content_age_days`, `word_count`,
`sessions_90d`.

**Deliberately excluded:**
- `health_score`, `priority_score` — FlyRank's own product outputs. Training on these would
  be circular: reproducing a score meant to independently evaluate the same pages.
- `clicks_90d` — alongside `impressions_90d` it algebraically reconstructs the label (`ctr`).
  Including it would leak the target rather than predict it.

**Leakage risks considered:** checked for both component leakage (`clicks_90d` feeding a
click-rate label) and outcome-bucket leakage (trend-derived fields such as `trend_direction`
/ `trend_pct`, which encode the label's own movement). Neither made it into the feature set.
`client_id` is used once, for the grouped split — never as a model feature.

**Confirmation:** no client names, domains, raw URLs, or query strings appear anywhere in
`work/` — the source release is pre-pseudonymized and pre-aggregated.

## 3. Baseline

A dynamic median: for each position tier, the median `ctr` computed on the **training**
split, mapped onto held-out pages by their tier. This is a fair comparison because it's
scored with the identical metric (RMSE) on the identical held-out split as the model — it's
also literally the heuristic content teams already use informally when they eyeball rank and
guess. Baseline RMSE: **0.521008**.

## 4. Model / analysis

Random Forest Regressor (`n_estimators=50`, `max_depth=7`, `random_state=42`). Chosen because
it can pick up non-linear, non-additive interactions between features (e.g. a page can be
under-visited *and* stale in a way that compounds) without needing those interactions
hand-specified, and it's robust to the outliers a search-performance dataset always has.

**Features:** `impressions_90d`, `avg_position`, `content_age_days`, `word_count`,
`sessions_90d` — left out `health_score`, `priority_score`, and `clicks_90d` on purpose (§2).

**Target:** `ctr` — observed click-through rate over the trailing 90-day window.

## 5. Evaluation

**Split:** client-grouped 80/20 holdout (`GroupShuffleSplit`, `random_state=42`), grouped by
`client_id`. Grouped rather than random because a random split would let the model see pages
from the same domain in both train and test, so it could memorize a site's architecture
instead of learning a transferable signal — grouping forces it to generalize to domains it's
never seen.

**Metrics, model vs. baseline, same split:**

| Method | RMSE |
|---|---|
| Position-tier baseline | 0.521008 |
| Random Forest Regressor | 0.419816 |

The model's error is 19.4% lower than the baseline's on held-out domains.

**Error analysis:** the largest predicted gaps concentrate in positions 1–20 — high-visibility
pages underperforming their tier by the widest margin. Past position ~60 the predicted gap
flattens toward zero for almost every page, which matches the intuition that low-rank pages
have little CTR headroom left to recover regardless of content quality.

## 6. Interpretation

*Not yet run.* The current notebook doesn't print `rf_model.feature_importances_`, so this
report can't honestly claim which feature is driving the model's predictions yet — only that
the model as a whole outperforms the baseline. Add this before submitting:

```python
importances = pd.Series(rf_model.feature_importances_, index=features).sort_values(ascending=False)
print(importances)
```

Fill this section in with the printed ranking and one plain-language sentence per top feature
once you've run it. Surprises or a feature landing lower than expected (a genuine "no effect")
are worth reporting as-is rather than leaving out.

## 7. Recommendation

Ranked by priority, in the order a FlyRank editor would work them tomorrow:

1. **High visibility, severe underperformance** — >5,000 impressions, missing expected CTR by
   >5%. Immediate sprint priority: title tag, meta description, intent review.
2. **Page-one click bleed** — positions 1–10, gap >3%. Position is already earned; the loss
   is in the snippet, not the ranking.
3. **Moderate opportunity** — a real but smaller gap. Standard monthly refresh backlog.
4. **No-go list** — don't automate metadata deployment on model output alone; move carefully
   (or not at all) on pages holding position 1 for core branded terms.

**Confidence and limits, stated plainly:** this is an observational study. A predicted gap is
a correlation between search signals and CTR, not evidence that refreshing a specific page
will close that exact gap — treat the ranking as a prioritization signal, not a forecast. The
model also can't see zero-click SERP features (AI overviews, instant answers), which can
suppress CTR regardless of page quality and will still show up as a false "opportunity."

## 8. Reproducibility

From a fresh clone:

```bash
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute work/notebooks/capstone.ipynb --output capstone_executed.ipynb
```

**Seeds:** `random_state=42` everywhere a seed is used — the `GroupShuffleSplit` and the
`RandomForestRegressor` both. A fresh run should reproduce the RMSE values in §5 exactly.

**Environment:** <paste `pip freeze` highlights or your `requirements.txt` deltas here —
at minimum `pandas`, `numpy`, `scikit-learn`>.

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
>
> **Metrics vs. base rate:** this task is regression (RMSE), not classification, so there's no
> majority-class base rate to report — the position-tier baseline *is* the equivalent
> reference point (§3), and it's reported alongside the model on every table in this report
> and on the deployed paper.
