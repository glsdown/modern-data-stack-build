# Modern Data Stack Build

This repo is designed to hold code for building a production-grade Modern Data Stack to support both batch and streaming data ingestion. It will be created with pragmatism in mind with both self-hosted and managed services included as reference points where possible.

## The datasets

The project is relatively contrived in order to simplify the requirements of the data stack. It will very simply mimic a sales functionality, where sales can be recorded in an app for products. Supporting data about the customers, products and exchanges rates are also retrieved.

### Source data -> Bronze

There will be 4 sources of ingestion:

- A simple Python app which will run in [PythonAnywhere](https://www.pythonanywhere.com/) which will send event data simulating product purchases to a Kafka topic
- A free PostgreSQL database holding customer details hosted on [Clever Cloud](https://www.clever.cloud/) and mocked via [Mockaroo](https://mockaroo.com/)
- API endpoint for [exchange rate data](https://docs.openexchangerates.org/reference/historical-json)
- Customer data held in [HubSpot](https://www.hubspot.com/products/crm) as example SaaS data extraction

Data will be ingested into Snowflake via Kafka Connect (Streaming), dlt running in Prefect (Database / API) and Airbyte.

### Silver

dbt will be responsible for loading the data from bronze and into silver.

Streamed data - `fact_purchases`

| customer_email | product_id | quantity | purchase_ts |
| --- | --- | --- | --- |
| string | string | int | timestamp |

Database data - `dim_products`

| product_id | product_name | price_usd |
| --- | --- | --- |
| bigint | string | float |

API data - `dim_currency`

| id | base_currency | currency | rate | exchange_ts | 
| --- |--- | --- | --- | --- |
| string | string | string | float | timestamp |

SaaS data - `dim_customers`

| customer_id | first_name | last_name | email | created_ts |
| --- | --- | --- | --- | --- |
| bigint | string | string | string | timestamp |

### Gold

Finally they will be joined in Gold to create the following sales report - `sales`

| customer_id | first_name | last_name | total_transactions | total_items | total_usd | total_gbp |
| --- | --- | --- | --- | --- | --- | --- |
| string | string | string | int | int | float | float |

