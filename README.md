# ds-delivery-delay

This project explores whether we can predict, at purchase time, if an Olist order will be delivered late relative to the promised delivery date.

The work started as a regression problem using `delay_days`, but the target distribution showed that most deliveries were early and only a small minority were late. Because of that imbalance, the project was reframed as a binary classification task:

- `is_late = 1` if the order arrived after the estimated delivery calendar date
- `is_late = 0` if the order arrived on time or early

## What We Did

The notebook:

- built a purchase-time modeling table from order, customer, seller, geography, item, product, and payment features
- created a day-level lateness target instead of using the exact delivery timestamp
- trained three classifiers:
  - a dummy baseline that always predicts not-late
  - a class-weighted logistic regression
  - a class-weighted random forest
- evaluated the models with precision-recall focused metrics instead of relying on accuracy
- used permutation importance to understand which features were helping the random forest most
- tested a smaller top-8-feature version of the random forest and compared it with the full-feature model

## Main Result

The full-feature random forest was the best model in this experiment, so that is the version we keep.

At the default `0.50` threshold, the full random forest produced:

- PR AUC: `0.2087`
- Precision: `0.3184`
- Recall: `0.1469`

Interpretation:

- when the model predicts an order will be late, it is correct about `31.8%` of the time
- at the default threshold, it catches about `14.7%` of truly late orders
- the model is better than the dummy baseline at ranking risky orders, but the overall signal is still modest

This means the result is best described as **promising but weak**, not strong. The model has real signal, but not enough yet to be considered highly accurate for operational use without further feature work.

## What “Percent Correct” Means Here

Because only about `6.8%` of orders are late, plain accuracy is misleading.

For example:

- the full random forest is about `92.1%` accurate at the default threshold
- but a dummy model that predicts every order is not-late is about `93.2%` accurate

So even though the random forest looks “accurate” in an overall sense, that number is not the right one to trust. In this project, the more useful measures are:

- `PR AUC`, which tells us how well the model ranks rare late orders above non-late ones
- `Precision`, which tells us how often a late prediction is actually correct
- `Recall`, which tells us how many of the truly late orders we successfully catch


## Full Model vs Top-8 Feature Experiment

Permutation importance showed the strongest features in the full random forest were:

- `purchase_month`
- `estimated_delivery_time_days`
- `distance_km`
- `same_state`
- `total_freight`
- `total_weight`
- `total_price`
- `total_volume`

We retrained the random forest using only those top 8 features to test whether removing weaker features would improve the model.

Results at the default `0.50` threshold:

- Full-feature random forest:
  - PR AUC: `0.2087`
  - Precision: `0.3184`
  - Recall: `0.1469`
- Top-8-feature random forest:
  - PR AUC: `0.2055`
  - Precision: `0.3027`
  - Recall: `0.1737`

Conclusion:

- the top-8 model improved recall slightly
- the full-feature model kept better overall ranking quality and precision
- the weaker-ranked features were not pure noise; together they still added some value

For that reason, the full-feature model remains the preferred version.

## Why Classification Was Better Than Regression

Classification was a better fit than the earlier regression framing for three reasons:

- the main business question is whether an order will be late, not the exact number of delay days
- the original regression target was dominated by early deliveries, which made the late cases harder to model directly
- classification gives us better evaluation tools for this rare-event problem, especially precision, recall, and PR AUC

In other words, classification did not solve the low-signal problem by itself, but it framed the problem in a way that better matched both the target distribution and the business decision we care about.
