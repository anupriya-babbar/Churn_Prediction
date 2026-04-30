# KKBOX Churn Prediction — Project Write-Up

---

## What Problem Did We Solve?

KKBOX is a Taiwanese music streaming platform with 30,755 subscribers
and a churn rate sitting at **50.35%** — one in every two users was not
renewing their subscription.

The business pain was twofold:

**1. No early warning system.**
The team only knew a user had churned *after* the subscription expired.
By then it was too late to intervene. There was no mechanism to identify
at-risk users *before* they cancelled.

**2. Retention spend was untargeted.**
Without knowing who was likely to churn, any retention offer — a discount,
a push notification, a personalised email — had to be sent either to
everyone (expensive) or randomly (wasteful). The hit rate was essentially
the background churn rate: 50%.

**What we built:**
A machine learning model that scores churn risk at every song-play
interaction — not once a month, but in real time, every time a user
plays a song. This enables:

- Flagging a user as at-risk the moment their behaviour shifts
- Targeting retention offers only at high-risk users
- Identifying which product surfaces drive churn vs retention
- Giving the product team a specific, measurable north star metric

**The scale of the problem:**
7,377,418 listening events across 30,755 users, joined from 4 data sources:
listening logs, user demographics, song metadata, and song origin data.

---

## Why Did We Choose That Solution?

### 1. Row-level prediction over user-level

The first instinct was to aggregate all listening events per user and
predict one churn label per person. This is the standard approach.

We discovered it was wrong.

**86.8% of users had BOTH churned (1) and stayed (0) labels** across
their rows. Aggregating destroyed the signal. Every aggregation method
— first, majority vote, max — produced a different churn rate, none
matching the ground truth 50.35%.

The dataset was designed for interaction-level prediction. Each row
answered: *"Given this user played this song via this source — did
they churn?"*

This is actually more powerful for a product team. Instead of a monthly
batch report, you get a real-time risk score at every touchpoint.

### 2. Context features per row

Row-level data has no memory. Knowing a user played a song via "search"
tells you nothing without knowing whether that user *normally* searches
or *normally* uses their library.

We solved this by computing each user's behavioral profile across all
their rows — percentage of plays from each source, total plays, language
preferences — then joining those aggregates back to every individual row.

Each row now carried two layers:
- **What happened right now** (this specific play event)
- **Who this user normally is** (their overall behavioral fingerprint)

This let the model detect deviation — a habitual library user suddenly
searching is a stronger churn signal than a habitual searcher doing the
same thing.

### 3. LightGBM over XGBoost

We trained XGBoost first — the industry standard for tabular churn
problems. It reached AUC 0.7706 after hyperparameter tuning.

We switched to LightGBM because of one architectural difference:
**leaf-wise tree growth vs level-wise**.

XGBoost splits all leaves at each level equally, even weak ones.
LightGBM always splits the leaf with the highest error reduction first.
With complex behavioral patterns across 7.3M rows, this means LightGBM
reaches deep conditional patterns — "library user AND new user AND
city 1" — far more efficiently.

Result: LightGBM hit AUC 0.8181 vs XGBoost's best of 0.7706.
A gain of +4.75 AUC points from algorithm choice alone.

### 4. Behavioral features over demographics

Correlation analysis confirmed what we suspected: gender, city, and age
ranked dead last in predictive power (r < 0.02 each). Churn is not a
demographic problem.

The strongest features were all behavioral:
- How does the user find music? (source_type — importance 0.238)
- Which screen do they use? (source_screen_name — importance 0.156)
- What is their discovery ratio? (active vs passive listening)

We built the feature set accordingly — 11 behavioral percentage features,
5 interaction features crossing behavioral signals, and 6 target-encoded
categorical features — and dropped columns that added noise without signal.

---

## What Trade-Offs Did We Make?

### Trade-off 1 — Prediction level vs interpretability

**What we chose:** Row-level prediction (one score per song play)
**What we gave up:** Simple user-level reports (one score per user)

Row-level prediction is more powerful for real-time systems but harder
to explain to stakeholders. "This user has a churn score of 0.73 right
now" is less intuitive than "this user will churn this month."

**Why it was worth it:** The product team gains the ability to trigger
interventions *during* a session, not after a billing cycle. The business
value of real-time outweighs the interpretability cost.

### Trade-off 2 — Recall vs Precision

At the default 0.5 decision threshold our model produces:

| Metric | Value |
|---|---|
| Churn catch rate (Recall) | 73.5% |
| Precision | 74.0% |
| False alarm rate | 26.1% |

We chose a balanced threshold because we did not have the business's
exact cost ratio (cost of a wasted offer vs cost of a lost subscriber).

In production this threshold should be tuned. If a retention offer costs
₹100 and a lost subscriber costs ₹800, you should lower the threshold
to 0.35 — catching more churners at the cost of more false alarms is
mathematically worth it at that cost ratio.

### Trade-off 3 — Feature engineering depth vs data leakage risk

Target encoding (replacing category codes with their churn rates) is a
powerful technique but introduces data leakage if done carelessly — using
test set churn rates to encode training data inflates AUC artificially.

We computed all churn rates exclusively from the training set and applied
them to the test set. This added implementation complexity but ensured
the evaluation was honest.

### Trade-off 4 — Model complexity vs diminishing returns

We stopped at 5,997 trees. The model was still improving (+0.0004 AUC
per 200 trees) but the gain curve had clearly flattened. Training 4,000
more trees would have added ~90 minutes of compute for roughly +0.003
AUC — not worth it.

The correct next step for meaningful AUC improvement (+0.04 to +0.10)
is adding transaction data and daily listening logs — new information
sources, not more trees on existing information.

### Trade-off 5 — Dataset version vs ideal data

The version of the KKBOX dataset available (bvmadduluri/wsdm-kkbox)
did not include `transactions.csv` or `user_logs.csv` — the two files
from the original WSDM Cup competition that contain payment history and
daily listening counts.

These are the strongest churn signals in subscription businesses:
- Payment behaviour (auto-renew off, discount history, plan downgrades)
- Engagement trend (listening declining week over week before expiry)

Without them, our ceiling was approximately AUC 0.82. The full
competition dataset with those files is expected to support AUC 0.88–0.92.

We shipped at 0.818 rather than waiting for perfect data — a working
model with known limitations is more valuable than a perfect model
that doesn't exist yet.

---

## What Happened After Launch?

*The following section is structured as post-launch recommendations
since this is a portfolio project. In a real deployment these would
be measured outcomes.*

### Immediate product changes (Week 1–2)

**Discovery strip inside My Library tab**
The single highest-impact change. 49.9% of all plays happen in My Library
at a 62% churn rate. Adding "New releases from your artists" and a
Radio CTA inside the tab directly targets the largest churn surface.

**Settings tab retention intercept**
Users visiting the Settings screen showed 59.1% churn rate — they are
checking their subscription before cancelling. A retention offer
("Stay 3 months at 50% off — you've listened to 847 songs this month")
on that screen is the last high-leverage intervention point.

### Metric the team started tracking (Week 3)

**Discovery ratio** — the proportion of plays coming from active platform
features (discover, search home, radio) vs passive/local sources
(my library, local files).

```
discovery_ratio = (discover + search + radio plays) / total plays
```

Target: push average user's discovery_ratio above 0.50.
Users above 0.50 churn at approximately 35%.
Users below 0.50 churn at approximately 65%.
A 30 percentage point difference from one metric.

### A/B test launched (Month 1)

**Hypothesis:** Routing new users to the Radio tab during onboarding
(instead of My Library) will reduce 90-day churn.

**Control:** Standard onboarding → My Library tab
**Treatment:** Onboarding → Radio tab with "Your personal station" framing

Expected outcome based on churn rates:
- Radio tab churn: 22.3%
- My Library tab churn: 62.0%
- If even 30% of new users adopt radio habit: ~12 point churn reduction

### Model retraining plan (Quarter 2)

The model was trained on a balanced 50/50 dataset. Real KKBOX churn
is closer to 8–10%. Before using for production targeting, retrain on:

1. Real historical churn labels (not competition-balanced)
2. With `transactions.csv` added — payment signals
3. With `user_logs.csv` added — daily listening trend features

Expected AUC with full data: 0.88–0.92, enabling significantly more
precise targeting and lower false alarm rates.

### Key lesson for the data team

Churn is a **behavioral problem**, not a demographic one.

Gender, city, and age — the three features most commonly used in
demographic segmentation campaigns — ranked 22nd, 23rd, and 24th out
of 24 features by predictive importance.

The #1 predictor was `source_type`: how the user finds their music.
The #2 predictor was `source_screen_name`: which screen they were on.

Every retention campaign segmented by age bracket or city is
leaving money on the table. Segment by listening behavior instead.

---

## Summary

| | |
|---|---|
| **Problem** | 50% churn rate, no early warning system, untargeted retention spend |
| **Approach** | Row-level LightGBM on behavioral + contextual features |
| **Key insight** | Churn is behavioral — how users find music predicts whether they leave |
| **Final AUC** | 0.8181 (above industry average of 0.70–0.80) |
| **Catch rate** | 73.5% of churners correctly identified |
| **North star** | discovery_ratio — push above 0.5, churn drops ~30 points |
| **Next step** | Add transaction + daily log data → target AUC 0.88–0.92 |

---

*Built end-to-end on Google Colab using Python, LightGBM, XGBoost,
pandas, scikit-learn. Dataset: KKBOX WSDM Cup 2018 via KaggleHub.*
