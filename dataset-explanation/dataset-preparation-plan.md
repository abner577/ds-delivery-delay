# Dataset Preparation Plan

## Goal
We want to train a model to predict delivery delay.

The model will not predict the estimated delivery date itself. Instead, it will predict how early or late an order was delivered compared to the estimated delivery date.

We need to prepare the Olist dataset for a purchase-time delivery delay model.

The final modeling table will have **one row per order**.

Target:

- `delay_days = order_delivered_customer_date - order_estimated_delivery_date`

Interpretation:

- positive = late
- zero = on time
- negative = early

## Final V1 Decisions

Prediction-time assumption:

- Only use information available at purchase time.

Use these raw datasets:

- `olist_orders_dataset`
- `olist_customers_dataset`
- `olist_sellers_dataset`
- `olist_geolocation_dataset`
- `olist_order_items_dataset`
- `olist_products_dataset`

Do not use in V1:

- `olist_order_payments_dataset`
- `olist_order_reviews_dataset`

Also exclude these as model inputs:

- `order_delivered_customer_date`
- `order_delivered_carrier_date`
- `order_approved_at`
- `order_status`
- raw `product_id`
- raw `seller_id`
- raw coordinates
- raw ZIP prefixes in the final model table

## Intermediate Data Frames

### `geo_zip_df`

One row per ZIP prefix with:

- ZIP prefix
- average latitude
- average longitude

### `customer_features_df`

One row per `customer_id` with:

- `customer_state`
- customer coordinates used only to derive distance

### `seller_features_df`

One row per `seller_id` with:

- `seller_state`
- seller coordinates used only to derive distance

### `item_product_df`

One row per item, created by joining:

- `order_items`
- `products`
- seller information if needed for later order summaries

This is the detailed item-level table used for feature engineering.

### `order_item_agg_df`

One row per `order_id` with these finalized V1 features:

- `item_count`
- `unique_products`
- `unique_sellers`
- `total_price`
- `total_freight`
- `total_weight`
- `total_volume`
- `dominant_category`
- `num_categories`

### `final_model_df`

One row per `order_id` with:

- time features
- geography features
- order-item aggregate features
- target column

## Final V1 Feature Set

### Time features

From `order_purchase_timestamp`:

- `purchase_month`
- `purchase_dayofweek`
- `purchase_hour`
- `is_weekend`

From estimated delivery:

- `estimated_delivery_time_days`

This means the number of days between purchase time and estimated delivery date.

### Geography features

Final geography features:

- `customer_state`
- `seller_state`
- `distance_km`
- `same_state`

Coordinates are used only to calculate `distance_km`, then dropped.

### Order-item features

- `item_count`
- `unique_products`
- `unique_sellers`
- `total_price`
- `total_freight`
- `total_weight`
- `total_volume`
- `dominant_category`
- `num_categories`

## Missing-Value Handling

For V1:

- numeric product fields such as weight and dimensions: median imputation
- missing product category: fill with `unknown`
- missing geolocation match: keep the row and handle `distance_km` during preprocessing
- add a missing-distance flag later only if needed

Important:

- Fit imputation values on the training split only, then apply them to test data.

## Notes For Implementation

- `shipping_limit_date` features are excluded in V1 because we are not fully sure they are valid at purchase time.
- Payment features are saved for V2.
- Customer history features are saved for V2.
- If V1 performance is weak, V2 can add payment features and richer item statistics such as averages or max values.
