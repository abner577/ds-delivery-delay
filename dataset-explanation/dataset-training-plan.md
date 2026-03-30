# Dataset Training Plan

## Goal
We want to train a model to predict delivery delay.

The model will not predict the estimated delivery date itself. Instead, it will predict how early or late an order was delivered compared to the estimated delivery date.

## Target Variable
We create a new column for the target:

`delay_days = actual_delivery_date - estimated_delivery_date`

How to interpret it:

- Positive value = delivered late
- Zero = delivered on time
- Negative value = delivered early


## Features
The model should only use features that are available at the time we want to make the prediction.

Examples of useful features:

- Customer location
- Seller location
- Distance between seller and customer
- Purchase timestamp
- Estimated delivery date
- Item count
- Product weight and size
- Freight value
- Payment information

## Distance Feature
Distance is not already given directly, so we need to create it.

Steps:

1. Get the customer ZIP code prefix from the customers dataset.
2. Use the geolocation dataset to map that ZIP prefix to latitude and longitude.
3. Get the seller ZIP code prefix from the sellers dataset.
4. Use the geolocation dataset again to map the seller ZIP prefix to latitude and longitude.
5. Compute the distance between seller and customer.
6. Store that result in a new feature such as `distance_km`.


## Training and Testing
For both training and testing, we only input the features that would have been known before the delivery happened.

That means we do not use future information such as:

- `actual_delivery_date`
- review data
- other columns only known after delivery

The model learns from the input features and predicts the delay amount.

## Main Idea
In supervised learning:

- `X` = the input features known before delivery
- `y` = the actual delivery delay

So the process is:

1. Join the relevant datasets
2. Engineer useful features like distance
3. Create the target column for delay
4. Split the data into training and test sets
5. Train the model to predict delay
6. Evaluate how accurate the predictions are
