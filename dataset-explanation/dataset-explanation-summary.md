# Dataset Explanation Summary

## Overview

The Olist dataset is a group of related CSV files that describe the full lifecycle of an e-commerce order, from purchase to delivery and review. For delivery delay prediction, the most important idea is to connect these datasets so we can compare the promised delivery date with the actual delivery date and identify what factors may explain delays.

## `olist_customers_dataset.csv`

**What it represents:** Customer identity at the order level, plus customer location.

### Columns

- `customer_id`: Unique customer identifier tied to a specific order. Used to join with `orders`.
- `customer_unique_id`: Identifier for the actual person across multiple purchases.
  Example: The same person may have different `customer_id` values on two orders but share the same `customer_unique_id`.
- `customer_zip_code_prefix`: First five digits of the customer's ZIP code.
- `customer_city`: Customer city.
- `customer_state`: Customer state.

**How we could use it:** This table helps us understand where the order is going. The most useful columns are likely `customer_zip_code_prefix`, `customer_city`, and `customer_state`, especially when combined with geolocation data.

## `olist_geolocation_dataset.csv`

**What it represents:** A mapping from ZIP code prefixes to latitude and longitude.

### Columns

- `geolocation_zip_code_prefix`: ZIP prefix used to join location data.
- `geolocation_lat`: Latitude coordinate.
- `geolocation_lng`: Longitude coordinate.
- `geolocation_city`: City associated with the ZIP prefix.
- `geolocation_state`: State associated with the ZIP prefix.

**How we could use it:** This table helps us estimate geographic distance between seller and customer, which is likely one of the strongest delay-related features. The most important columns are `geolocation_zip_code_prefix`, `geolocation_lat`, and `geolocation_lng`.

## `olist_order_items_dataset.csv`

**What it represents:** The products inside each order, one row per item.

### Columns

- `order_id`: Order identifier.
- `order_item_id`: Item number within the order.
  Example: If the same order has `order_item_id` values `1`, `2`, and `3`, that order contains 3 items.
- `product_id`: Product identifier.
- `seller_id`: Seller responsible for that item.
- `shipping_limit_date`: Deadline for the seller to hand the item to the carrier.
- `price`: Price of the item.
- `freight_value`: Shipping cost assigned to the item.
  Example: If freight is split across 3 items, summing `freight_value` across the order gives total freight.

**How we could use it:** This table helps us measure order complexity, seller involvement, and shipping cost. Useful columns likely include `order_item_id`, `seller_id`, `shipping_limit_date`, `price`, and `freight_value`.

## `olist_order_payments_dataset.csv`

**What it represents:** Payment details for each order.

### Columns

- `order_id`: Order identifier.
- `payment_sequential`: Sequence number when an order uses multiple payments.
  Example: An order split across two payment records may have sequences `1` and `2`.
- `payment_type`: Payment method, such as credit card or boleto.
- `payment_installments`: Number of installments.
- `payment_value`: Amount paid in that payment record.

**How we could use it:** This table may add supporting signal around processing behavior and order value. Columns worth considering include `payment_type`, `payment_installments`, `payment_sequential`, and `payment_value`.

## `olist_order_reviews_dataset.csv`

**What it represents:** Customer review outcomes after the order experience.

### Columns

- `review_id`: Review identifier.
- `order_id`: Order identifier.
- `review_score`: Rating from 1 to 5.
  Example: A very low score may reflect a poor experience, potentially including late delivery.
- `review_comment_title`: Short review title.
- `review_comment_message`: Full review text.
- `review_creation_date`: Date the review survey was sent.
- `review_answer_timestamp`: Date the customer answered.

**How we could use it:** This table is more useful for analyzing the impact of delays than for predicting them ahead of time, since reviews happen after delivery. The main column of interest is usually `review_score`.

## `olist_orders_dataset.csv`

**What it represents:** The central order-level table and the main timeline of the order process.

### Columns

- `order_id`: Unique order identifier.
- `customer_id`: Connects the order to the customer table.
- `order_status`: Order status, such as delivered or shipped.
- `order_purchase_timestamp`: When the order was placed.
- `order_approved_at`: When payment was approved.
- `order_delivered_carrier_date`: When the seller handed the order to the logistics partner.
- `order_delivered_customer_date`: When the customer actually received the order.
- `order_estimated_delivery_date`: The promised delivery date.
  Example: If estimated delivery is May 10 and actual delivery is May 12, the delay is 2 days.

**How we could use it:** This is the most important dataset because it defines the target and the delivery timeline. The key columns are `order_delivered_customer_date`, `order_estimated_delivery_date`, `order_purchase_timestamp`, `order_approved_at`, and `order_delivered_carrier_date`.

## `olist_products_dataset.csv`

**What it represents:** Product details, especially category, size, and weight.

### Columns

- `product_id`: Product identifier.
- `product_category_name`: Product category.
- `product_name_lenght`: Length of the product name.
- `product_description_lenght`: Length of the product description.
- `product_photos_qty`: Number of product photos.
- `product_weight_g`: Product weight in grams.
- `product_length_cm`: Product length.
- `product_height_cm`: Product height.
- `product_width_cm`: Product width.
  Example: Weight and dimensions can be combined to estimate whether a product is bulky or harder to ship.

**How we could use it:** This table helps describe what is being shipped and how difficult it may be to transport. The most useful columns are likely `product_category_name`, `product_weight_g`, `product_length_cm`, `product_height_cm`, and `product_width_cm`.

## `olist_sellers_dataset.csv`

**What it represents:** Seller identity and seller location.

### Columns

- `seller_id`: Seller identifier.
- `seller_zip_code_prefix`: First five digits of the seller ZIP code.
- `seller_city`: Seller city.
- `seller_state`: Seller state.

**How we could use it:** This table tells us where the shipment starts, which matters for estimating route difficulty and distance. The most useful columns are `seller_zip_code_prefix`, `seller_city`, and `seller_state`.

## Final Note

For delivery delay prediction, the most important tables are usually `olist_orders_dataset.csv`, `olist_customers_dataset.csv`, `olist_sellers_dataset.csv`, `olist_geolocation_dataset.csv`, `olist_order_items_dataset.csv`, and `olist_products_dataset.csv`. In practice, we will likely start from `orders`, join customer and seller geography, then add order item and product features to build the final modeling table.
