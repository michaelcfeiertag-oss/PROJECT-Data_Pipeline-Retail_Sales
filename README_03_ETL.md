# README 03 - ETL Pipeline Guide

# ETL Pipeline Guide

## Building the ETL Version of the Retail Sales Data Pipeline

---

## 1. Why this README exists

In this part of the project, you will build the **ETL version** of the pipeline.

ETL means:

* **Extract**
* **Transform**
* **Load**

In this design, the data is:

1. extracted from the source files
2. transformed and cleaned in Python
3. loaded into PostgreSQL only after it is ready

This is a real data engineering pattern used in industry.

In real jobs, teams choose ETL when they want to control data quality before inserting data into the final reporting table.

So while this is still a beginner-friendly project, the workflow itself reflects a real engineering idea.

Take this seriously and imagine this is a task given to you by a data engineering lead.

---

## 2. What you will build

In this ETL version, you will:

* read raw source files
* clean customer, order, and product data in Python
* join the data together
* calculate useful business fields
* load the final result into PostgreSQL

The final result will be a clean PostgreSQL reporting table called:

* `sales_report`

This table should be ready for business reporting.

---

## 3. ETL flow for this project

Here is the flow you are building:

```text
Raw files → Extract with Python → Transform in Python → Load clean data into PostgreSQL
```

More specifically:

```text
customers.csv
orders.json
products.xml
      ↓
  Python extraction
      ↓
  Python cleaning and joining
      ↓
  Final clean records
      ↓
 PostgreSQL sales_report
```

Keep this order in your mind the whole time.

That order is what makes this pipeline ETL.

---

## 4. Before you begin

Make sure your repository already has this structure:

```text
retail-sales-data-pipeline/
├── data/
│   └── raw/
│       ├── customers.csv
│       ├── orders.json
│       └── products.xml
├── sql/
│   └── create_tables.sql
├── src/
│   ├── extract.py
│   ├── transform_etl.py
│   ├── load_postgres.py
│   └── run_etl.py
```

If your project does not look close to this yet, go back to **README 02** first.

---

## 5. Step 1 - Create the PostgreSQL target table

### Your task

Before loading anything, prepare the final PostgreSQL table.

Open:

```text
sql/create_tables.sql
```

and add this SQL:

```sql
CREATE TABLE IF NOT EXISTS sales_report (
    order_id INTEGER,
    order_date DATE,
    customer_id INTEGER,
    customer_name TEXT,
    city TEXT,
    product_id INTEGER,
    product_name TEXT,
    category TEXT,
    quantity INTEGER,
    price NUMERIC(10,2),
    total_amount NUMERIC(10,2)
);
```

### Why this matters

In the ETL version, the final cleaned data will go directly into this table.

So this table represents the reporting-ready business output.

This is realistic because data engineers often create a target schema before building the rest of the pipeline.

---

## 6. Step 2 - Start the extraction layer

### Your task

Open:

```text
src/extract.py
```

and begin with the imports and path setup:

```python
import csv
import json
import xml.etree.ElementTree as ET
from pathlib import Path

RAW_DATA_FOLDER = Path("data") / "raw"
```

### Why this code matters

This code prepares the tools needed for the three file formats in the project:

* `csv` for customer data
* `json` for order data
* `xml.etree.ElementTree` for product data
* `Path` for clean and readable file paths

This is already realistic because real data engineering projects often work with more than one data format.

---

## 7. Step 3 - Read the customer CSV file

### Your task

Still inside `src/extract.py`, add this function:

```python
def read_customers_csv():
    customers_file = RAW_DATA_FOLDER / "customers.csv"
    customers = []

    with open(customers_file, mode="r", encoding="utf-8") as file:
        reader = csv.DictReader(file)
        for row in reader:
            customers.append(row)

    return customers
```

### What this code does

This function:

* builds the file path
* opens the customer CSV file
* reads each row as a dictionary
* stores all rows in a list
* returns that list

### Why this matters

Reading CSV files correctly is one of the most common beginner data engineering tasks.

---

## 8. Step 4 - Read the orders JSON file

### Your task

Now add this function below the CSV reader:

```python
def read_orders_json():
    orders_file = RAW_DATA_FOLDER / "orders.json"

    with open(orders_file, mode="r", encoding="utf-8") as file:
        orders = json.load(file)

    return orders
```

### What this code does

This function opens the JSON file and loads its content into Python.

Depending on the structure of the file, the result will usually be:

* a list of dictionaries
* or a dictionary containing nested data

For this project, we assume it returns order records that we can loop through.

---

## 9. Step 5 - Read the products XML file

### Your task

Now add the XML reader:

```python
def read_products_xml():
    products_file = RAW_DATA_FOLDER / "products.xml"
    tree = ET.parse(products_file)
    root = tree.getroot()

    products = []

    for product in root.findall("product"):
        products.append({
            "product_id": product.find("product_id").text,
            "product_name": product.find("product_name").text,
            "category": product.find("category").text,
            "price": product.find("price").text
        })

    return products
```

### What this code does

This function:

* opens the XML file
* parses it
* loops through each `<product>` element
* extracts the needed fields
* converts them into dictionaries

### Why this matters

XML is very common in older enterprise systems, so learning to turn XML into a usable Python structure is a valuable skill.

---

## 10. Step 6 - Test the extraction layer

### Your task

At the bottom of `src/extract.py`, temporarily add this test block:

```python
if __name__ == "__main__":
    customers = read_customers_csv()
    orders = read_orders_json()
    products = read_products_xml()

    print("Customers:", customers[:2])
    print("Orders:", orders[:2])
    print("Products:", products[:2])
```

### Why this matters

A professional habit is to test each step before moving on.

This helps you confirm:

* the file paths are correct
* the files open successfully
* the data structure looks reasonable

Do not wait until the whole project is finished before testing basic extraction.

---

## 11. Step 7 - Start the transformation layer

### Your task

Open:

```text
src/transform_etl.py
```

and begin with this helper function:

```python
def clean_text(value):
    if value is None:
        return ""
    return value.strip().title()
```

### What this code does

This function:

* handles `None`
* removes extra spaces
* converts text into title case

### Why this matters

In real source data, text often contains:

* extra spaces
* inconsistent capitalization
* missing values

A helper like this makes your transformation logic cleaner and reusable.

---

## 12. Step 8 - Clean customer records

### Your task

Add this function into `src/transform_etl.py`:

```python
def clean_customers(customers):
    cleaned_customers = []
    seen_customer_ids = set()

    for customer in customers:
        customer_id = customer.get("customer_id", "").strip()

        if not customer_id:
            continue

        if customer_id in seen_customer_ids:
            continue

        seen_customer_ids.add(customer_id)

        cleaned_customers.append({
            "customer_id": int(customer_id),
            "customer_name": clean_text(customer.get("customer_name")),
            "city": clean_text(customer.get("city")),
            "signup_date": customer.get("signup_date", "").strip()
        })

    return cleaned_customers
```

### What this code does

This function:

* skips rows with missing customer IDs
* removes duplicate customers
* cleans text fields
* converts `customer_id` into an integer

### Why this matters

This is realistic ETL cleaning work.

In real projects, customer data often contains duplicates and formatting problems.

---

## 13. Step 9 - Clean product records

### Your task

Now add product cleaning:

```python
def clean_products(products):
    cleaned_products = []

    for product in products:
        product_id = str(product.get("product_id", "")).strip()
        price_value = str(product.get("price", "")).strip()

        if not product_id:
            continue

        try:
            price = float(price_value)
        except ValueError:
            continue

        cleaned_products.append({
            "product_id": int(product_id),
            "product_name": clean_text(product.get("product_name")),
            "category": clean_text(product.get("category")),
            "price": price
        })

    return cleaned_products
```

### What this code does

This function:

* skips products with missing IDs
* converts price into a float
* ignores invalid prices
* standardizes product names and categories

### Why this matters

Products with invalid prices are dangerous for reporting.

This is a good example of why ETL is useful: you can stop bad data before it reaches the final table.

---

## 14. Step 10 - Clean order records

### Your task

Now add order cleaning:

```python
def clean_orders(orders):
    cleaned_orders = []

    for order in orders:
        order_id = str(order.get("order_id", "")).strip()
        customer_id = str(order.get("customer_id", "")).strip()
        product_id = str(order.get("product_id", "")).strip()
        quantity_value = str(order.get("quantity", "")).strip()
        order_date = str(order.get("order_date", "")).strip()

        if not order_id or not customer_id or not product_id:
            continue

        try:
            order_id = int(order_id)
            customer_id = int(customer_id)
            product_id = int(product_id)
            quantity = int(quantity_value)
        except ValueError:
            continue

        cleaned_orders.append({
            "order_id": order_id,
            "customer_id": customer_id,
            "product_id": product_id,
            "quantity": quantity,
            "order_date": order_date
        })

    return cleaned_orders
```

> **Hint:** This function safely skips bad rows.
>
> For example, if a raw record contains an invalid quantity such as `"two"`, the pipeline ignores that record instead of crashing.
>
> This is realistic because raw business data is often messy, and ETL pipelines must be robust enough to handle invalid values.

### What this code does

This function:

* skips rows missing important IDs
* converts IDs into integers
* converts quantity into integer
* keeps the date value ready for loading

### Why this matters

Orders are usually the center of the reporting logic, so they must be cleaned carefully.

---

## 15. Step 11 - Build lookup dictionaries

### Your task

Still in `src/transform_etl.py`, add this function:

```python
def build_lookup(records, key_field):
    lookup = {}

    for record in records:
        lookup[record[key_field]] = record

    return lookup
```

### Why this is useful

When joining data in Python, it is helpful to create fast lookup dictionaries.

For example:

* customers by `customer_id`
* products by `product_id`

This makes the join logic easier and clearer.

---

## 16. Step 12 - Join the cleaned datasets

### Your task

Now add the main join function:

```python
def build_sales_report(customers, products, orders):
    customer_lookup = build_lookup(customers, "customer_id")
    product_lookup = build_lookup(products, "product_id")

    sales_report = []

    for order in orders:
        customer = customer_lookup.get(order["customer_id"])
        product = product_lookup.get(order["product_id"])

        if not customer or not product:
            continue

        total_amount = order["quantity"] * product["price"]

        sales_report.append({
            "order_id": order["order_id"],
            "order_date": order["order_date"],
            "customer_id": customer["customer_id"],
            "customer_name": customer["customer_name"],
            "city": customer["city"],
            "product_id": product["product_id"],
            "product_name": product["product_name"],
            "category": product["category"],
            "quantity": order["quantity"],
            "price": product["price"],
            "total_amount": round(total_amount, 2)
        })

    return sales_report
```
> **Important:** At this point, `build_sales_report()` must receive the **cleaned customer records**, **cleaned product records**, and **cleaned order records**.
>
> The **raw order data** should **not** be passed directly into the report-building step.


### What this code does

This function:

* creates lookups for customers and products
* loops through orders
* finds the matching customer and product
* skips incomplete matches
* calculates `total_amount`
* builds the final clean reporting rows

### Why this matters

This is one of the most important ETL steps.

Business value appears when separate systems are connected into one useful reporting dataset.

---

## 17. Step 13 - Add a simple ETL pipeline function

### Your task

At the bottom of `src/transform_etl.py`, add this:

```python
def run_etl_transform(customers_raw, products_raw, orders_raw):
    customers_clean = clean_customers(customers_raw)
    products_clean = clean_products(products_raw)
    orders_clean = clean_orders(orders_raw)

    sales_report = build_sales_report(
        customers_clean,
        products_clean,
        orders_clean
    )

    return sales_report
```

### Why this matters

This function acts like the main transformation entry point.

It keeps your ETL logic organized and readable.

That is much better than scattering transformation code everywhere.

---

## 18. Step 14 - Prepare PostgreSQL loading

### Your task

Open:

```text
src/load_postgres.py
```

and add this code:

```python
import psycopg2


def get_connection():
    return psycopg2.connect(
        dbname="retail_project",
        user="postgres",
        password="your_password",
        host="localhost",
        port="5432"
    )
```

### Important note

Replace `"your_password"` with your real PostgreSQL password.

### Why this matters

This function creates the connection to PostgreSQL.

In real projects, this information is usually moved later into a config or environment variable, but for this beginner project, this is a good start.

---

## 19. Step 15 - Add the ETL loader function

### Your task

Continue inside `src/load_postgres.py` and add:

```python
def load_sales_report(records):
    connection = get_connection()
    cursor = connection.cursor()

    cursor.execute("DELETE FROM sales_report;")

    insert_query = """
        INSERT INTO sales_report (
            order_id,
            order_date,
            customer_id,
            customer_name,
            city,
            product_id,
            product_name,
            category,
            quantity,
            price,
            total_amount
        )
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
    """

    for record in records:
        cursor.execute(
            insert_query,
            (
                record["order_id"],
                record["order_date"],
                record["customer_id"],
                record["customer_name"],
                record["city"],
                record["product_id"],
                record["product_name"],
                record["category"],
                record["quantity"],
                record["price"],
                record["total_amount"]
            )
        )

    connection.commit()
    cursor.close()
    connection.close()
```

### What this code does

This function:

* connects to PostgreSQL
* clears the old `sales_report` rows
* inserts the new ETL result
* commits the transaction
* closes the connection

### Why this matters

In this ETL design, PostgreSQL receives the final cleaned output, not the raw messy source data.

That is the defining pattern of ETL.

---

## 20. Step 16 - Create the ETL runner script

### Your task

Open:

```text
src/run_etl.py
```

and add this code:

```python
from extract import read_customers_csv, read_orders_json, read_products_xml
from transform_etl import run_etl_transform
from load_postgres import load_sales_report


def main():
    customers_raw = read_customers_csv()
    orders_raw = read_orders_json()
    products_raw = read_products_xml()

    sales_report = run_etl_transform(
        customers_raw,
        products_raw,
        orders_raw
    )

    load_sales_report(sales_report)

    print(f"ETL completed successfully. Loaded {len(sales_report)} rows into sales_report.")


if __name__ == "__main__":
    main()
```

### What this code does

This is the ETL entry point.

It runs the full flow in order:

1. extract raw data
2. transform in Python
3. load the clean result into PostgreSQL

### Why this matters

A clear entry-point script makes the pipeline easier to understand and run.

That is good engineering practice.

---

## 21. Step 17 - Run the ETL pipeline

### Your task

From the project root, run:

```bash
python src/run_etl.py
```

### What you should expect

If everything is correct, you should see a success message similar to:

```text
ETL completed successfully. Loaded X rows into sales_report.
```

If you get an error, check these common issues:

* wrong PostgreSQL password
* table `sales_report` not created yet
* source files missing from `data/raw/`
* module import problems
* wrong working directory

This is realistic.
Debugging small issues is part of real data engineering work.

---

## 22. Step 18 - Validate the loaded data in PostgreSQL

### Your task

Now open PostgreSQL and run these checks.

First, verify the row count:

```sql
SELECT COUNT(*) FROM sales_report;
```

Then inspect a few rows:

```sql
SELECT * FROM sales_report LIMIT 10;
```

Then check total revenue by category:

```sql
SELECT category, SUM(total_amount) AS total_revenue
FROM sales_report
GROUP BY category
ORDER BY total_revenue DESC;
```

Then check orders by city:

```sql
SELECT city, COUNT(*) AS total_orders
FROM sales_report
GROUP BY city
ORDER BY total_orders DESC;
```

### Why this matters

A pipeline is not complete just because the script ran.

You also need to verify that the output makes sense.

This is a real engineering habit.

---

## 23. Step 19 - Optional improvement: create a category summary table

### Your task

If you want a small extension, add another table.

In `sql/create_tables.sql`, add:

```sql
CREATE TABLE IF NOT EXISTS category_summary (
    category TEXT,
    total_revenue NUMERIC(10,2)
);
```

Then later, after loading `sales_report`, you could populate that summary table with SQL.

For example:

```sql
INSERT INTO category_summary (category, total_revenue)
SELECT category, SUM(total_amount)
FROM sales_report
GROUP BY category;
```

### Why this helps

This makes the project feel even more like a reporting pipeline.

It shows that ETL can prepare both detailed tables and summary outputs.

---

## 24. Step 20 - What makes this ETL design realistic

This ETL pipeline reflects real engineering work because it includes:

* multiple source systems
* multiple file formats
* cleaning before loading
* joining business data
* calculating reporting fields
* inserting clean results into PostgreSQL
* validating output afterward

This is exactly why you should treat this project seriously.

It is still beginner-friendly, but the logic is realistic.

---

## 25. Step 21 - Full code overview by file

At this stage, your ETL version should look roughly like this:

### `src/extract.py`

Contains:

* `read_customers_csv()`
* `read_orders_json()`
* `read_products_xml()`

### `src/transform_etl.py`

Contains:

* `clean_text()`
* `clean_customers()`
* `clean_products()`
* `clean_orders()`
* `build_lookup()`
* `build_sales_report()`
* `run_etl_transform()`

### `src/load_postgres.py`

Contains:

* `get_connection()`
* `load_sales_report()`

### `src/run_etl.py`

Contains:

* `main()`

This helps you see the structure clearly.

---

## 26. Step 22 - What you should understand after finishing

By the end of this ETL part, you should understand that:

* ETL means transform before load
* Python performs the main cleaning in this version
* PostgreSQL stores the final clean output
* the target table is designed for business use
* the pipeline order defines the architecture

That is the core learning goal.

---

## 27. Step 23 - Suggested ETL deliverables

By the end of the ETL version, you should have:

* working extraction code
* working transformation code
* PostgreSQL loading code
* a `sales_report` table populated with clean data
* validation SQL queries
* a clear understanding of why this is ETL

These are strong beginner deliverables and reflect real engineering logic.

---

## 28. Final message

Treat this ETL pipeline like a real junior engineering task.

The goal is not to memorize the letters E, T, and L.

The goal is to understand what happens in each stage, why the order matters, and how raw business data becomes clean reporting data.

That is real data engineering thinking.

---

## 29. What comes next

Now continue with the second architecture style:

* [README 04 - ELT Pipeline Guide](./README_04_ELT.md)

In that version, you will load raw data into PostgreSQL first and transform it later using SQL.

That comparison will help you truly understand the difference between ETL and ELT.
