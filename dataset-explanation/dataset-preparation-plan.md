# Dataset Preparation Plan

## Goal

Prepare the Olist data for a delivery delay prediction model.

The final training dataset should have **one row per order**. We can and should use multiple intermediate data frames during preparation, but the final modeling table should be order-level.

## Recommended Modeling Strategy

Keep the raw datasets separate at first, then create intermediate feature tables before building one final merged table.

Recommended flow:

1. Start from `olist_orders_dataset` as the base order table.
2. Join customer-level location information from `olist_customers_dataset`.
3. Build seller and item-level aggregates from `olist_order_items_dataset`.
4. Join product attributes from `olist_products_dataset` before aggregating to order level.
5. Use `olist_geolocation_dataset` to map ZIP prefixes to coordinates.
6. Derive distance and other engineered features.
7. Build one final order-level data frame for training.

This avoids duplication problems from datasets like `order_items` and `payments`, where one order can have multiple rows.

## Prediction-Time Assumption

This plan assumes we want to predict delay **at purchase time**.

That means we should only include features that are known when the customer places the order, or very shortly after if they are already part of the order record. We should not use fields that are only known after fulfillment or delivery.

## Dataset-by-Dataset Column Decisions

### 1. `olist_customers_dataset`

Include:

- `customer_id`
- `customer_zip_code_prefix`
- `customer_city`
- `customer_state`

Reasoning:

- `customer_id` is needed to join with `orders`.
- `customer_zip_code_prefix` lets us connect the customer to geolocation data.
- `customer_city` and `customer_state` may capture regional delivery patterns that are not fully explained by distance alone.

Example:

- Two customers may be a similar distance from the seller, but one state may still experience more delays because of logistics coverage or regional routing patterns.

### 2. `olist_geolocation_dataset`

Use for feature engineering, not as a raw final table.

Keep:

- `geolocation_zip_code_prefix`
- `geolocation_lat`
- `geolocation_lng`

Optional:

- `geolocation_city`
- `geolocation_state`

Reasoning:

- The main purpose of this table is to convert ZIP prefixes into coordinates.
- Then we can calculate seller-to-customer distance.
- City and state in this table are usually redundant if we already keep them from customers and sellers.

Important note:

- This dataset contains multiple rows per ZIP prefix, so we should first collapse it to one row per ZIP prefix, usually by averaging latitude and longitude for each prefix.

### 3. `olist_order_items_dataset`

Keep during preparation:

- `order_id`
- `order_item_id`
- `product_id`
- `seller_id`
- `shipping_limit_date`
- `price`
- `freight_value`

Reasoning:

- This dataset is extremely useful, but most of its value comes from aggregation and feature engineering.
- `order_item_id` helps us count how many items are in the order.
- `product_id` lets us join to product characteristics.
- `seller_id` lets us join seller geography and count how many sellers are involved in the order.
- `price` and `freight_value` give us order value and shipping-cost information.
- `shipping_limit_date` may help create a feature related to fulfillment pressure if we compare it with purchase time.

Recommended engineered features at the order level:

- `item_count`
- `unique_products`
- `unique_sellers`
- `total_price`
- `avg_price`
- `total_freight`
- `avg_freight`
- `hours_until_shipping_limit` or `days_until_shipping_limit`

Important note:

- Raw `product_id` and raw `seller_id` usually should not remain in the first final model because they are high-cardinality identifiers.

### 4. `olist_order_payments_dataset`

Include as optional but useful.

Keep during preparation:

- `order_id`
- `payment_type`
- `payment_installments`
- `payment_value`
- `payment_sequential`

Reasoning:

- Payment data may not directly cause delivery delay, but it can still be predictive.
- Some payment methods may be associated with slower approval or different order behavior.
- Installments may correlate with order profile or order value.

Recommended engineered features:

- `num_payment_records`
- `payment_type_mode` or most common payment type
- `max_installments`
- `total_payment_value`

Important note:

- This dataset is not as essential as orders, items, products, customers, sellers, and geolocation, so it can be added in version 2 if we want a simpler first pass.

### 5. `olist_order_reviews_dataset`

Exclude from modeling features.

Reasoning:

- Reviews happen after the order experience.
- They are useful for post-delivery analysis, but not for a purchase-time delivery delay prediction model.

### 6. `olist_orders_dataset`

This should be the main base table.

Keep:

- `order_id`
- `customer_id`
- `order_purchase_timestamp`
- `order_estimated_delivery_date`

Use only for target creation, not as features:

- `order_delivered_customer_date`

Exclude from purchase-time features:

- `order_approved_at`
- `order_delivered_carrier_date`
- `order_status`

Reasoning:

- `order_purchase_timestamp` is one of the most important inputs because it lets us derive time-based features like weekday, month, and hour.
- `order_estimated_delivery_date` can be used to create the promised lead time.
- `order_delivered_customer_date` is needed to compute the target, but should not be used as an input feature.
- `order_delivered_carrier_date` and delivery-status information are only known later, so they should not be included in a purchase-time model.

Target definition:

- `delay_days = order_delivered_customer_date - order_estimated_delivery_date`

Interpretation:

- Positive = late
- Zero = on time
- Negative = early

### 7. `olist_products_dataset`

Include:

- `product_id`
- `product_category_name`
- `product_weight_g`
- `product_length_cm`
- `product_height_cm`
- `product_width_cm`

Optional:

- `product_name_lenght`
- `product_description_lenght`
- `product_photos_qty`

Reasoning:

- Product type, size, and weight can strongly affect delivery difficulty.
- Large or heavy products may require more handling or specialized transport.
- Product category may capture common shipping patterns.

Recommended engineered features:

- `product_volume_cm3 = product_length_cm * product_height_cm * product_width_cm`
- order-level totals or averages for weight and volume
- dominant product category or category counts

Important note:

- This dataset does not contain actual product names, descriptions, or image URLs. It contains lengths and counts related to those fields.

### 8. `olist_sellers_dataset`

Include:

- `seller_id`
- `seller_zip_code_prefix`
- `seller_city`
- `seller_state`

Reasoning:

- Seller ZIP prefix is needed to connect sellers to geolocation data.
- Seller city and state may capture origin-region effects that distance alone does not explain.

## Recommended Intermediate Data Frames

Using multiple data frames is a standard and useful strategy here.

Suggested intermediate tables:

### `geo_zip_df`

One row per ZIP prefix with:

- ZIP prefix
- average latitude
- average longitude

### `customer_features_df`

One row per `customer_id` with:

- customer ZIP prefix
- customer city
- customer state
- customer coordinates

### `seller_features_df`

One row per `seller_id` with:

- seller ZIP prefix
- seller city
- seller state
- seller coordinates

### `item_product_df`

Item-level table created by joining:

- `order_items`
- `products`
- `sellers`

This table can then be aggregated to order level.

### `order_item_agg_df`

One row per `order_id` with:

- item counts
- seller counts
- price aggregates
- freight aggregates
- weight aggregates
- volume aggregates
- category-based summary features

### `payment_agg_df`

One row per `order_id` with:

- payment aggregates
- installment features
- payment-type summary

### `final_model_df`

One row per `order_id` with:

- order features
- customer features
- order-item aggregates
- payment aggregates if included
- engineered distance and time features
- target column

## Strong V1 Feature Set

Recommended first-version features:

- purchase weekday
- purchase month
- purchase hour
- promised lead time in days
- customer ZIP/state/city
- seller ZIP/state/city
- customer latitude and longitude
- seller latitude and longitude
- seller-customer distance
- item count
- unique sellers
- total price
- total freight
- total weight
- total volume
- product category summary

Optional V2 additions:

- payment features
- text-length product metadata
- photo count features

## Main Exclusions

Do not use as model features:

- `order_delivered_customer_date`
- `order_delivered_carrier_date`
- review data
- raw review scores or review timestamps
- raw high-cardinality identifiers like `product_id` and `seller_id` in the first version

## Final Recommendation

The best approach is:

- use multiple intermediate data frames while preparing data
- aggregate item-level and payment-level tables to the order level
- build one final modeling dataset with one row per `order_id`

This structure is easier to reason about, easier to debug, and much safer for machine learning than merging every raw column from every dataset into one table.
