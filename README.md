Overview

This assignment covers working with JSON data in Python — specifically how to
fetch data from remote sources (a GitHub-hosted file and a live REST API),
convert it into structured DataFrames, and save the results as CSV files.

Three output files were produced across two workflows:

| File | Source | Rows | Columns |
|------|--------|------|---------|
| `hospital_data.csv` | Mockaroo → GitHub → Python | 1,000 | 5 |
| `products_data.csv` | DummyJSON `/products` API | 30 | 22 |
| `carts_data.csv` | DummyJSON `/carts` API | 114 | 12 |


---

# PART 2 — hospital_data.csv

## Step 1 — Creating the Dataset on Mockaroo

The dataset was created using [Mockaroo](https://mockaroo.com), a free online
tool for generating synthetic (fake but realistic) data. You define the field
names and their data types, set how many rows you want, and export in your
chosen format.

**Fields defined on Mockaroo:**

| Field Name | 
|------------|-
| `patient_id` | 
| `patient_name` | 
| `patient_dob` | 
| `diagnosis_code` | 
| `admission_date` | 
**Export format chosen:** JSON
**Number of records:** 1,000

Mockaroo exports the data as a JSON array — a list of 1,000 objects,
one per patient, each with the same 5 fields.

---

## Step 2 — Uploading to GitHub

After downloading `MOCK_DATA.json` from Mockaroo, it was uploaded to a
public GitHub repository named `Hospital_data` under the account `Mukunza`.

**Why GitHub?**
GitHub gives the file a stable, publicly accessible URL. Any Python script
running on any machine can fetch the file using that URL — which is the
point of the next step.

**How to get the Raw URL:**
1. Open the file on GitHub
2. Click the **Raw** button at the top right of the file view
3. Copy the URL from the browser address bar

Regular GitHub URL (has the website navigation around it — not usable by Python):
```
https://github.com/Mukunza/Hospital_data/blob/main/MOCK_DATA.json
```

Raw URL (pure file content only — this is what Python needs):
```
https://raw.githubusercontent.com/Mukunza/Hospital_data/refs/heads/main/MOCK_DATA.json
```

---

## Step 3 — Fetching the File with Python

**Library used:** `requests`

```python
import requests

url = "https://raw.githubusercontent.com/Mukunza/Hospital_data/refs/heads/main/MOCK_DATA.json"
r = requests.get(url)
```

`requests.get()` sends an HTTP GET request to the URL — the code equivalent
of typing the URL into a browser and pressing Enter. The response is stored in `r`.

**Checking the status code:**

```python
r.status_code
# Output: 200
```

200 means the request succeeded and data was returned. This was confirmed
in the notebook before proceeding.

---

## Step 4 — Parsing the JSON Response

```python
mock_data = r.json()
print(type(mock_data))
# Output: <class 'list'>
```

`r.json()` converts the raw text response into a Python object.
Because `MOCK_DATA.json` is a JSON array, Python receives it as a
**list of dictionaries** — one dictionary per patient.

Previewing the first two records to confirm the structure:

```python
mock_data[:2]
```

Output:
```python
[
  {'patient_id': 1, 'patient_name': 'Dagny Ferneyhough',
   'patient_dob': '9/13/1949', 'diagnosis_code': 'T285',
   'admission_date': '2021-09-22T16:46:17Z'},
  {'patient_id': 2, 'patient_name': 'Mahalia Beartup',
   'patient_dob': '11/14/1980', 'diagnosis_code': 'S43394',
   'admission_date': '2021-01-02T18:51:45Z'}
]
```

---

## Step 5 — Building the DataFrame

```python
import pandas as pd

df = pd.DataFrame(mock_data)
print(df.shape)   # (1000, 5)
df.head()
```

`pd.DataFrame()` takes the list of 1,000 dictionaries and turns it into a table:
each dictionary becomes one row, each key becomes a column. Result: 1,000 rows
and 5 columns.

---

## Step 6 — Checking and Fixing Data Types

```python
print(df.dtypes)
```

Initial output:
```
patient_id        int64     correct
patient_name      object    fine — text is always object
patient_dob       object    should be datetime
diagnosis_code    object    fine — it's a code string
admission_date    object   should be datetime
```

`patient_dob` and `admission_date` showed as `object` (text) because pandas
does not automatically recognise date-shaped strings as dates when reading JSON.
They must be converted explicitly.

**Fix:**

```python
df["patient_dob"]    = pd.to_datetime(df["patient_dob"])
df["admission_date"] = pd.to_datetime(df["admission_date"])

print(df.dtypes)
```

After fix:
```
patient_id                      int64
patient_name                   object
patient_dob            datetime64[ns]        fixed
diagnosis_code                 object
admission_date    datetime64[ns, UTC]        fixed
```

`admission_date` became `datetime64[ns, UTC]` because its original values
included a timezone indicator (`Z` = UTC). pandas preserves this timezone
information automatically.

---

## Step 7 — Saving as CSV

```python
df.to_csv("hospital_data.csv", index=False)
print("Saved as hospital_data.csv")
# Output: Saved as hospital_data.csv
```

`index=False` prevents pandas from writing its internal row numbers
(0, 1, 2...) as an extra first column in the CSV.

Because Jupyter Notebook runs locally on the machine, the file was saved
directly to the same folder as the notebook:
```
C:\Users\user\hospital_data.csv
```

This was confirmed with:
```python
import os
print(os.path.abspath("hospital_data.csv"))
# Output: C:\Users\user\hospital_data.csv
```

### Final Output — `hospital_data.csv`

| Rows | Columns | Column Names |
|------|---------|--------------|
| 1,000 | 5 | `patient_id`, `patient_name`, `patient_dob`, `diagnosis_code`, `admission_date` |

---

---

# PART 3 — products_data.csv and carts_data.csv

Both files came from the DummyJSON public API. Unlike Part 2 where the data
was a plain JSON array, DummyJSON wraps its records inside a dictionary —
which required a different approach for each endpoint.

---

## ENDPOINT 1 — products_data.csv

**URL:** `https://dummyjson.com/products`

### Step 1 — Sending the Request

```python
import requests

url_products = "https://dummyjson.com/products"
r = requests.get(url_products)
r.status_code
# Output: 200
```

### Step 2 — Parsing and Inspecting the JSON

```python
products_data = r.json()
print(type(products_data))
# Output: <class 'dict'>
```

The response is a **dictionary**, not a list like MOCK_DATA. It looks like:

```json
{
  "products": [ {...}, {...}, ... ],
  "total": 194,
  "skip": 0,
  "limit": 30
}
```

The actual product records live inside the `"products"` key.
`total` = 194 products exist in the database; `limit` = 30 were returned
in this call (the default page size).

### Step 3 — First Attempt and What Went Wrong

The first attempt ran `pd.DataFrame(products_data)` directly on the whole
response dictionary:

```python
# First attempt — wrong
df = pd.DataFrame(products_data)
print(df.shape)
# Output: (30, 4)   ← only 4 columns: products, total, skip, limit
```

This produced the wrong result. pandas saw 4 keys in the dictionary and made
4 columns — with the entire list of 30 products crammed into the `products`
column as a single object per cell. That is not usable data.

### Step 4 — Correct Approach: Drill Into the Key First

```python
# Correct — extract the list first, then build the DataFrame
products_list = products_data["products"]
df = pd.DataFrame(products_list)

print(df.shape)
# Output: (30, 22)
df.head()
```

This produced 30 rows and 22 columns — one row per product, one column per field.
`id`, `title`, `category`, `brand`, `price` and all other product fields each got
their own proper column.

### Step 5 — Saving as CSV

```python
df.to_csv("products_data.csv", index=False)
print("Saved as products_data.csv")
# Output: Saved as products_data.csv
```

File location confirmed:
```python
import os
print(os.path.abspath("products_data.csv"))
# Output: C:\Users\user\products_data.csv
```

### Final Output — `products_data.csv`

| Rows | Columns | Source |
|------|---------|--------|
| 30 | 22 | `https://dummyjson.com/products` |

Notable columns: `id`, `title`, `description`, `category`, `price`,
`discountPercentage`, `rating`, `stock`, `brand`, `availabilityStatus`,
`returnPolicy`, `warrantyInformation`, `shippingInformation`, plus
nested columns like `tags`, `images`, `reviews`, `meta`, `dimensions`

---

## ENDPOINT 2 — carts_data.csv

**URL:** `https://dummyjson.com/carts`

### Step 1 — Sending the Request

```python
url_carts = "https://dummyjson.com/carts"
r = requests.get(url_carts)
r.status_code   # checking if we got the URL successfully
# Output: 200
```

### Step 2 — Parsing the JSON

```python
carts_data = r.json()
print(type(carts_data))
# Output: <class 'dict'>
```

### Step 3 — First Attempt and What Went Wrong

```python
#  First attempt — wrong
carts_list = carts_data["carts"]
df = pd.DataFrame(carts_data)   # used the wrong variable

print(df.shape)
# Output: (30, 4)   ← carts, total, skip, limit
```

`carts_list` was extracted correctly but then the wrong variable (`carts_data`
instead of `carts_list`) was passed to `pd.DataFrame()`. The result was the
same broken 4-column table seen with products — with each entire cart object
crammed into a single cell in the `carts` column.

This was identified, understood, and then fixed in the next section of the
notebook (labelled **"CHANGING THE CARTS DATA TO THE RIGHT FORMAT"**).

### Step 4 — Why Carts Need Flattening

Even after fixing the variable name, carts have an additional complexity.
The JSON has **three levels of nesting**:

```
Level 1 → response wrapper    { "carts": [...], "total": 208 }
Level 2 → each cart           { "id": 1, "userId": 1, "products": [...] }
Level 3 → each product        { "id": 162, "title": "Blue Frock", "quantity": 4 }
```

A cart contains **multiple products inside it**. Simply doing
`pd.DataFrame(carts_list)` would give one row per cart with all its products
buried inside a list in a single cell — still not usable for analysis.

The solution is **flattening**: loop through every cart, then every product
inside that cart, and build one flat row per cart-product combination.

### Step 5 — Flattening the Nested Structure

```python
#  Correct — flatten by looping through carts → products inside each cart
rows = []

for cart in carts_list:
    for product in cart["products"]:
        rows.append({
            "cart_id"              : cart["id"],
            "user_id"              : cart["userId"],
            "cart_total_usd"       : cart["total"],
            "cart_total_items"     : cart["totalQuantity"],
            "cart_unique_products" : cart["totalProducts"],
            "product_id"           : product.get("id"),
            "product_title"        : product.get("title"),
            "unit_price_usd"       : product.get("price"),
            "quantity_ordered"     : product.get("quantity"),
            "line_total_usd"       : product.get("total"),
            "discount_pct"         : product.get("discountPercentage"),
            "discounted_total_usd" : product.get("discountedTotal"),
        })

df = pd.DataFrame(rows)

print(df.shape)
# Output: (114, 12)
df.head()
```

**Result:** 114 rows and 12 columns. Cart 1 had 4 products → 4 rows.
Cart 2 had 2 products → 2 rows. The `cart_id`, `user_id`, and `cart_total_usd`
values repeat across all rows that belong to the same cart — this is correct
and expected. It means no context is lost.

**First 5 rows of the output:**

| cart_id | user_id | cart_total_usd | product_title | unit_price_usd | quantity_ordered |
|---------|---------|---------------|---------------|---------------|-----------------|
| 1 | 1 | 13037.88 | Blue Frock | 29.99 | 4 |
| 1 | 1 | 13037.88 | Generic Motorcycle | 3999.99 | 3 |
| 1 | 1 | 13037.88 | iPhone 6 | 299.99 | 3 |
| 1 | 1 | 13037.88 | Baseball Ball | 8.99 | 2 |
| 2 | 2 | 139.93 | Man Short Sleeve Shirt | 19.99 | 5 |

### Step 6 — Saving as CSV

```python
df.to_csv("carts_data.csv", index=False)
print("Saved as carts_data.csv")
# Output: Saved as carts_data.csv
```

File location confirmed:
```python
import os
print(os.path.abspath("carts_data.csv"))
# Output: C:\Users\user\carts_data.csv
```

### Final Output — `carts_data.csv`

| Rows | Columns | Source |
|------|---------|--------|
| 114 | 12 | `https://dummyjson.com/carts` |

Columns: `cart_id`, `user_id`, `cart_total_usd`, `cart_total_items`,
`cart_unique_products`, `product_id`, `product_title`, `unit_price_usd`,
`quantity_ordered`, `line_total_usd`, `discount_pct`, `discounted_total_usd`
