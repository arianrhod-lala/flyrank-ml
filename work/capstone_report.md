# Capstone Report — CTR / Engagement Opportunity Scoring

- **Author:** <your name>
- **Lane:** CTR / Engagement Opportunity Scoring
- **Repo:** <your repo URL>
- **Date:** <submission date>

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

Feature importance and residual error were computed in `work/notebooks/w05_model.ipynb`
(same features, target, filter, and split as the capstone) via
`rf_model.feature_importances_`:

| Feature | Importance |
|---|---|
| `sessions_90d` | 0.384408 |
| `avg_position` | 0.253978 |
| `impressions_90d` | 0.164532 |
| `word_count` | 0.136093 |
| `content_age_days` | 0.060989 |

**In plain words:** actual user interaction (`sessions_90d`) drives the model's expectation
more than rank does. `avg_position` alone — the entire basis of the position-tier baseline —
is only the second-most-important signal, which is consistent with the model beating that
baseline: raw rank is real information, but it's not the dominant one once visitor behavior
is available. `content_age_days` barely moves the prediction, which is a mild negative
result worth stating plainly: staleness by itself isn't a strong signal in this feature set,
so "the page is old" is a weaker justification for prioritizing a refresh than "the page is
under-visited relative to its rank."

**Error analysis:** the residual plot (actual CTR − predicted CTR) shows the model's largest
misses cluster in positions 1–3, with errors mostly small and centered near zero from
position ~10 onward. This tracks a known pattern in search: a branded, navigational query can
pull an 80%+ CTR at position 1, while a broad informational query might only pull 15% at the
same position — and the feature set has no query-intent signal to tell those apart. Net
effect: the model tends to underpredict CTR for highly-branded top-of-SERP pages and
overpredict it for informational ones in that same top slot. This is a genuine limitation of
the current feature set, not a modeling bug, and it's why the ranked recommendations
(§7) don't treat position-1 branded pages as safe to auto-flag.

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
