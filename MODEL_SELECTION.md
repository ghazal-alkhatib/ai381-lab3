# Model selection

## The question
The model gives each customer a probability that they will leave (churn), so the retention team knows who to call first.
If it misses a customer who is leaving, we lose that customer. If it flags a happy customer, the team wastes a call on them.

## Candidates
| Candidate | Why I tried it | AUC | F1 | F1 per slice (lowest) | p95 latency (ms) | Size (KB) | MLflow run ID |
|---|---|---|---|---|---|---|---|
| logreg_balanced | Simple and fast baseline. Only 25% of customers churn, so I used class_weight="balanced" to make it care more about churners. | 0.798 | 0.589 | 0.333 (two-year) | 10.8 | 3.5 | 8c3de5bb |
| rf_100_depth6_balanced | A different family (many trees voting) that can learn non-linear patterns. Smaller and shallower than the starter (100 trees, depth 6). | 0.771 | 0.569 | 0.400 (two-year) | 15.9 | 794 | 14f532e5 |
| gboost_150_depth2 | Boosting: small trees added one after another, each one fixing the last one's mistakes. It has no class_weight option. | 0.770 | 0.438 | 0.250 (one-year) | 7.5 | 122 | d9002168 |

## The champion
I picked logreg_balanced. It has the best F1 (0.589) and the best AUC (0.798), and it is about 200 times smaller than the random forest.
I accepted two losses: gradient boosting is a bit faster (7.5 ms vs 10.8 ms), but both are far inside my budget, and its F1 is much lower (0.438).
The random forest does better on two-year customers (0.400 vs 0.333), but it is worse on average and much bigger.
The simplest model won. Using class_weight="balanced" mattered more than using a complex model: the only model without it had the lowest F1.

## My promotion rules
- Latency budget (p95): 50 ms, because a churn score is used by the retention team, not in a real-time screen. My laptop measured 7 to 16 ms, and CI runners can be slower, so 50 ms leaves a safety margin.
- Minimum gain to replace the champion (F1 points): 0.01, because the test set has about 240 customers and about 60 churners. Catching one more churner changes F1 by about 0.01, so a smaller gain could just be noise.

## What I would check next
The two-year slice. All three models are weakest on two-year or one-year customers, and very few of them churn, so their F1 is based on only a handful of people.
If more two-year customers start leaving in production, my champion could miss them. I would watch F1 per contract type on new data, and retrain if the slices drift.