```python
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
import random
```


```python
# Load the CSV file
df = pd.read_csv("D:/01_data/raw/ledger.csv")
print (df)
```

      transaction_id transaction_date merchant_id  amount_usd   status  \
    0           R001       2026-03-01        M001      1200.0  success   
    1           R002       2026-03-01        M002       850.0  success   
    2           R003       2026-03-02        M001       500.0  success   
    3           R004       2026-03-02        M003      2100.0  success   
    4           R005       2026-03-03        M004      7200.0  success   
    5           R006       2026-03-03        M002       950.0  success   
    6           R007       2026-03-04        M005      3300.0   failed   
    7           R008       2026-03-04        M001       640.0  success   
    8           R009       2026-03-05        M002      4100.0  success   
    9           R010       2026-03-05        M004      2500.0  success   
    
      payment_method  
    0            UPI  
    1           Card  
    2         Wallet  
    3           Card  
    4           Card  
    5            UPI  
    6     NetBanking  
    7           Card  
    8           Card  
    9         Wallet  
    


```python
# Load the Gateway CSV file
df = pd.read_csv("D:/01_data/raw/gateway.csv")
print (df)
```

      transaction_id transaction_date merchant_id  amount_usd   status  \
    0           R001       2026-03-01        M001      1200.0  success   
    1           R002       2026-03-01        M002       900.0  success   
    2           R003       2026-03-02        M001       500.0  success   
    3           R005       2026-03-03        M004      7200.0   failed   
    4           R006       2026-03-03        M002       950.0  success   
    5           R007       2026-03-04        M005      3300.0   failed   
    6           R008       2026-03-04        M001       600.0  success   
    7           R009       2026-03-05        M002      4100.0  success   
    8           R011       2026-03-05        M003      1800.0  success   
    
      payment_method  
    0            UPI  
    1           Card  
    2         Wallet  
    3           Card  
    4            UPI  
    5     NetBanking  
    6           Card  
    7           Card  
    8           Card  
    


```python
# For ledger
ledger.duplicated().sum()          # total duplicate rows
ledger[ledger.duplicated()]        # view duplicate rows

# If you want to check duplicates only by transaction_id
ledger.duplicated(subset=["transaction_id"]).sum()
ledger[ledger.duplicated(subset=["transaction_id"], keep=False)]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>transaction_id</th>
      <th>transaction_date</th>
      <th>merchant_id</th>
      <th>amount_usd</th>
      <th>status</th>
      <th>payment_method</th>
    </tr>
  </thead>
  <tbody>
  </tbody>
</table>
</div>




```python
import pandas as pd

# Load both files into correctly named variables
ledger = pd.read_csv("D:/01_data/raw/ledger.csv")
gateway = pd.read_csv("D:/01_data/raw/gateway.csv")

print("Ledger shape:", ledger.shape)
print("Gateway shape:", gateway.shape)

# Strip spaces from transaction IDs
ledger["transaction_id"] = ledger["transaction_id"].astype(str).str.strip()
gateway["transaction_id"] = gateway["transaction_id"].astype(str).str.strip()

# Records present in ledger but missing in gateway
missing_in_gateway = ledger[~ledger["transaction_id"].isin(gateway["transaction_id"])]

# Show result
print(missing_in_gateway)

# Count missing records
print("Missing count:", len(missing_in_gateway))
```

    Ledger shape: (10, 6)
    Gateway shape: (9, 6)
      transaction_id transaction_date merchant_id  amount_usd   status  \
    3           R004       2026-03-02        M003      2100.0  success   
    9           R010       2026-03-05        M004      2500.0  success   
    
      payment_method  
    3           Card  
    9         Wallet  
    Missing count: 2
    


```python
# Clean up key column
ledger["transaction_id"] = ledger["transaction_id"].astype(str).str.strip()
gateway["transaction_id"] = gateway["transaction__id"].astype(str).str.strip()

# Records present in gateway but NOT present in ledger
missing_in_ledger = gateway[~gateway["transaction_id"].isin(ledger["transaction_id"])]

# Show result
print(missing_in_ledger)
```

      transaction_id transaction_date merchant_id  amount_usd   status  \
    8           R011       2026-03-05        M003      1800.0  success   
    
      payment_method  
    8           Card  
    


```python
# Inner join on transaction_id to compare amounts
merged = pd.merge(
    ledger,
    gateway,
    on="transaction_id",
    how="inner",
    suffixes=("_ledger", "_gateway")
)

amount_mismatch = merged[merged["amount_usd_ledger"] != merged["amount_usd_gateway"]]

print(amount_mismatch)
print("Total amount mismatches:", len(amount_mismatch))
```

      transaction_id transaction_date_ledger merchant_id_ledger  \
    1           R002              2026-03-01               M002   
    6           R008              2026-03-04               M001   
    
       amount_usd_ledger status_ledger payment_method_ledger  \
    1              850.0       success                  Card   
    6              640.0       success                  Card   
    
      transaction_date_gateway merchant_id_gateway  amount_usd_gateway  \
    1               2026-03-01                M002               900.0   
    6               2026-03-04                M001               600.0   
    
      status_gateway payment_method_gateway  
    1        success                   Card  
    6        success                   Card  
    Total amount mismatches: 2
    


```python
# Find status mismatches
status_mismatch = merged[merged["status_ledger"] != merged["status_gateway"]]

# Show mismatches
print(status_mismatch[["transaction_id", "status_ledger", "status_gateway"]])

# Count
print("Total status mismatches:", len(status_mismatch))
```

      transaction_id status_ledger status_gateway
    3           R005       success         failed
    Total status mismatches: 1
    


```python

import pandas as pd

# End-to-end reconciliation template; adjust gateway column names and file paths

# 1) Build combined base with all transaction_ids from both sides
all_ids = pd.Series(
    pd.concat([ledger["transaction_id"], gateway["transaction_id"]], ignore_index=True).unique(),
    name="transaction_id"
)

# 2) Merge ledger and gateway onto this list
recon = all_ids.to_frame().merge(
    ledger.add_suffix("_ledger"),
    left_on="transaction_id",
    right_on="transaction_id_ledger",
    how="left"
).merge(
    gateway.add_suffix("_gateway"),
    left_on="transaction_id",
    right_on="transaction_id_gateway",
    how="left"
)

# 3) Flags: presence / missing
recon["present_in_ledger"] = ~recon["transaction_id_ledger"].isna()
recon["present_in_gateway"] = ~recon["transaction_id_gateway"].isna()

recon["missing_in_ledger"] = ~recon["present_in_ledger"] & recon["present_in_gateway"]
recon["missing_in_gateway"] = recon["present_in_ledger"] & ~recon["present_in_gateway"]

# 4) Amount mismatch (only where present in both)
recon["amount_usd_ledger"] = pd.to_numeric(recon["amount_usd_ledger"], errors="coerce")
recon["amount_usd_gateway"] = pd.to_numeric(recon["amount_usd_gateway"], errors="coerce")

tolerance = 0.00  # set to 0.01 if you want to allow rounding differences
recon["amount_mismatch"] = (
    recon["present_in_ledger"]
    & recon["present_in_gateway"]
    & (recon["amount_usd_ledger"] - recon["amount_usd_gateway"]).abs() > tolerance
)

# 5) Status mismatch (only where present in both)
recon["status_ledger_norm"] = recon["status_ledger"].astype(str).str.strip().str.lower()
recon["status_gateway_norm"] = recon["status_gateway"].astype(str).str.strip().str.lower()

recon["status_mismatch"] = (
    recon["present_in_ledger"]
    & recon["present_in_gateway"]
    & (recon["status_ledger_norm"] != recon["status_gateway_norm"])
)

# 6) Overall reconciliation status per row
def classify_row(row):
    if row["missing_in_ledger"]:
        return "Missing in ledger"
    if row["missing_in_gateway"]:
        return "Missing in gateway"
    if row["amount_mismatch"] and row["status_mismatch"]:
        return "Amount & status mismatch"
    if row["amount_mismatch"]:
        return "Amount mismatch"
    if row["status_mismatch"]:
        return "Status mismatch"
    return "Matched"

recon["recon_result"] = recon.apply(classify_row, axis=1)

# 7) Select key columns for final report
final_report = recon[[
    "transaction_id",
    "transaction_date_ledger",
    "merchant_id_ledger",
    "amount_usd_ledger",
    "status_ledger",
    "transaction_date_gateway",
    "merchant_id_gateway",
    "amount_usd_gateway",
    "status_gateway",
    "missing_in_ledger",
    "missing_in_gateway",
    "amount_mismatch",
    "status_mismatch",
    "recon_result"
]]

# 8) Save to CSV
final_report.to_csv("reconciliation_report.csv", index=False)
print("Reconciliation report created: reconciliation_report.csv")

final_report.head()
```

    Reconciliation report created: reconciliation_report.csv
    




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>transaction_id</th>
      <th>transaction_date_ledger</th>
      <th>merchant_id_ledger</th>
      <th>amount_usd_ledger</th>
      <th>status_ledger</th>
      <th>transaction_date_gateway</th>
      <th>merchant_id_gateway</th>
      <th>amount_usd_gateway</th>
      <th>status_gateway</th>
      <th>missing_in_ledger</th>
      <th>missing_in_gateway</th>
      <th>amount_mismatch</th>
      <th>status_mismatch</th>
      <th>recon_result</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>R001</td>
      <td>2026-03-01</td>
      <td>M001</td>
      <td>1200.0</td>
      <td>success</td>
      <td>2026-03-01</td>
      <td>M001</td>
      <td>1200.0</td>
      <td>success</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>Matched</td>
    </tr>
    <tr>
      <th>1</th>
      <td>R002</td>
      <td>2026-03-01</td>
      <td>M002</td>
      <td>850.0</td>
      <td>success</td>
      <td>2026-03-01</td>
      <td>M002</td>
      <td>900.0</td>
      <td>success</td>
      <td>False</td>
      <td>False</td>
      <td>True</td>
      <td>False</td>
      <td>Amount mismatch</td>
    </tr>
    <tr>
      <th>2</th>
      <td>R003</td>
      <td>2026-03-02</td>
      <td>M001</td>
      <td>500.0</td>
      <td>success</td>
      <td>2026-03-02</td>
      <td>M001</td>
      <td>500.0</td>
      <td>success</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>Matched</td>
    </tr>
    <tr>
      <th>3</th>
      <td>R004</td>
      <td>2026-03-02</td>
      <td>M003</td>
      <td>2100.0</td>
      <td>success</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>False</td>
      <td>True</td>
      <td>False</td>
      <td>False</td>
      <td>Missing in gateway</td>
    </tr>
    <tr>
      <th>4</th>
      <td>R005</td>
      <td>2026-03-03</td>
      <td>M004</td>
      <td>7200.0</td>
      <td>success</td>
      <td>2026-03-03</td>
      <td>M004</td>
      <td>7200.0</td>
      <td>failed</td>
      <td>False</td>
      <td>False</td>
      <td>False</td>
      <td>True</td>
      <td>Status mismatch</td>
    </tr>
  </tbody>
</table>
</div>




```python
import os
import json

# Create required directories
os.makedirs("01_data/processed", exist_ok=True)
os.makedirs("04_python", exist_ok=True)

# Save all output files to correct paths

# 1. missing_in_gateway.csv
missing_in_gateway.to_csv("01_data/processed/missing_in_gateway.csv", index=False)
print("Saved: 01_data/processed/missing_in_gateway.csv")

# 2. missing_in_ledger.csv
missing_in_ledger.to_csv("01_data/processed/missing_in_ledger.csv", index=False)
print("Saved: 01_data/processed/missing_in_ledger.csv")

# 3. amount_mismatches.csv
amount_mismatches.to_csv("01_data/processed/amount_mismatches.csv", index=False)
print("Saved: 01_data/processed/amount_mismatches.csv")

# 4. status_mismatches.csv
status_mismatches.to_csv("01_data/processed/status_mismatches.csv", index=False)
print("Saved: 01_data/processed/status_mismatches.csv")

# 5. reconciliation_report.csv
final_report.to_csv("01_data/processed/reconciliation_report.csv", index=False)
print("Saved: 01_data/processed/reconciliation_report.csv")

# 6. summary_metrics.json
with open("04_python/summary_metrics.json", "w") as f:
    json.dump(summary_metrics, f, indent=4)
print("Saved: 04_python/summary_metrics.json")

print("\nAll output files saved successfully!")
```

    Saved: 01_data/processed/missing_in_gateway.csv
    Saved: 01_data/processed/missing_in_ledger.csv
    


    ---------------------------------------------------------------------------

    NameError                                 Traceback (most recent call last)

    Cell In[42], line 19
         16 print("Saved: 01_data/processed/missing_in_ledger.csv")
         18 # 3. amount_mismatches.csv
    ---> 19 amount_mismatches.to_csv("01_data/processed/amount_mismatches.csv", index=False)
         20 print("Saved: 01_data/processed/amount_mismatches.csv")
         22 # 4. status_mismatches.csv
    

    NameError: name 'amount_mismatches' is not defined



```python

```


```python
import os
import json

# Create required directories
os.makedirs("01_data/processed", exist_ok=True)
os.makedirs("04_python", exist_ok=True)

# 3. amount_mismatch -> amount_mismatches.csv
amount_mismatch.to_csv("01_data/processed/amount_mismatches.csv", index=False)
print("Saved: 01_data/processed/amount_mismatches.csv")

# 4. status_mismatch -> status_mismatches.csv
status_mismatch.to_csv("01_data/processed/status_mismatches.csv", index=False)
print("Saved: 01_data/processed/status_mismatches.csv")

# 5. reconciliation_report.csv (already saved, re-saving to correct path)
final_report.to_csv("01_data/processed/reconciliation_report.csv", index=False)
print("Saved: 01_data/processed/reconciliation_report.csv")

# 6. summary_metrics.json
try:
    with open("04_python/summary_metrics.json", "w") as f:
        json.dump(summary_metrics, f, indent=4)
    print("Saved: 04_python/summary_metrics.json")
except NameError:
    print("summary_metrics not found - checking %whos dict...")

print("\nDone! All output files saved.")
```

    Saved: 01_data/processed/amount_mismatches.csv
    Saved: 01_data/processed/status_mismatches.csv
    Saved: 01_data/processed/reconciliation_report.csv
    summary_metrics not found - checking %whos dict...
    
    Done! All output files saved.
    


```python

```


```python
%whos DataFrame
```

    Variable             Type         Data/Info
    -------------------------------------------
    amount_mismatch      DataFrame      transaction_id transact<...>                   Card  
    df                   DataFrame               generated_at  <...>chant': {'merchant_i...  
    final_report         DataFrame       transaction_id transac<...>lse   Missing in ledger  
    gateway              DataFrame      transaction_id transact<...>ard  \n8           Card  
    ledger               DataFrame      transaction_id transact<...>ard  \n9         Wallet  
    merged               DataFrame      transaction_id transact<...>                   Card  
    missing_in_gateway   DataFrame      transaction_id transact<...>ard  \n9         Wallet  
    missing_in_ledger    DataFrame      transaction_id transact<...>hod  \n8           Card  
       transaction_id transac<...>n\n[11 rows x 22 columns]
    status_mismatch      DataFrame      transaction_id transact<...>                   Card  
    


```python
import json
import os

os.makedirs("04_python", exist_ok=True)

# Build summary_metrics from existing variables
summary_metrics = {
    "total_ledger_records": int(len(ledger)),
    "total_gateway_records": int(len(gateway)),
    "missing_in_gateway": int(len(missing_in_gateway)),
    "missing_in_ledger": int(len(missing_in_ledger)),
    "amount_mismatches": int(len(amount_mismatch)),
    "status_mismatches": int(len(status_mismatch)),
    "total_reconciliation_records": int(len(final_report))
}

with open("04_python/summary_metrics.json", "w") as f:
    json.dump(summary_metrics, f, indent=4)

print("Saved: 04_python/summary_metrics.json")
print(summary_metrics)
```

    Saved: 04_python/summary_metrics.json
    {'total_ledger_records': 10, 'total_gateway_records': 9, 'missing_in_gateway': 2, 'missing_in_ledger': 1, 'amount_mismatches': 2, 'status_mismatches': 1, 'total_reconciliation_records': 11}
    


```python
import os
print("Notebook working directory:")
print(os.getcwd())
```

    Notebook working directory:
    C:\Users\Hp
    


```python
import os
cwd = os.getcwd()
print(repr(cwd))
```

    'C:\\Users\\Hp'
    


```python
import json
with open(r"D:\01_data\raw\api_response_sample.json", "r", encoding="utf-8") as f:
    data = json.load(f)
print(type(data))
if isinstance(data, list):
    print(f"List with {len(data)} items")
    print("First item keys:", data[0].keys() if data else "empty")
    import pprint
    pprint.pprint(data[0])
elif isinstance(data, dict):
    print("Dict keys:", data.keys())
    import pprint
    pprint.pprint(data)
```

    <class 'dict'>
    Dict keys: dict_keys(['generated_at', 'source', 'batches'])
    {'batches': [{'batch_id': 'B001',
                  'merchant': {'merchant_id': 'M001',
                               'merchant_name': 'Alpha Mart',
                               'region': 'APAC'},
                  'settlements': [{'amount_usd': 1520.5,
                                   'bank': {'country': 'IN', 'name': 'Bank A'},
                                   'processed_at': '2026-03-07T08:10:00Z',
                                   'settlement_id': 'S001',
                                   'status': 'settled'},
                                  {'amount_usd': 980.0,
                                   'bank': {'country': 'IN', 'name': 'Bank A'},
                                   'processed_at': '2026-03-07T08:45:00Z',
                                   'settlement_id': 'S002',
                                   'status': 'pending'},
                                  {'amount_usd': 640.0,
                                   'bank': {'country': 'SG', 'name': 'Bank B'},
                                   'processed_at': '2026-03-07T09:15:00Z',
                                   'settlement_id': 'S003',
                                   'status': 'settled'}]},
                 {'batch_id': 'B002',
                  'merchant': {'merchant_id': 'M004',
                               'merchant_name': 'Delta Travels',
                               'region': 'US'},
                  'settlements': [{'amount_usd': 2100.0,
                                   'bank': {'country': 'US', 'name': 'Bank C'},
                                   'processed_at': '2026-03-07T08:20:00Z',
                                   'settlement_id': 'S004',
                                   'status': 'settled'},
                                  {'amount_usd': 500.0,
                                   'bank': {'country': 'US', 'name': 'Bank C'},
                                   'processed_at': '2026-03-07T08:50:00Z',
                                   'settlement_id': 'S005',
                                   'status': 'failed'},
                                  {'amount_usd': 7200.0,
                                   'bank': {'country': 'US', 'name': 'Bank C'},
                                   'processed_at': '2026-03-07T09:30:00Z',
                                   'settlement_id': 'S006',
                                   'status': 'settled'}]}],
     'generated_at': '2026-03-07T10:00:00Z',
     'source': 'QuickPay Settlement API'}
    


```python
import json
import pandas as pd
import os

# ── PART 4: Normalize api_response_sample.json ──────────────────────────────

# Step 1: Read the nested JSON file
with open(r"D:\01_data\raw\api_response_sample.json", "r", encoding="utf-8") as f:
    api_data = json.load(f)

print("Step 1 ✔ JSON loaded")
print(f"  source     : {api_data['source']}")
print(f"  generated_at: {api_data['generated_at']}")
print(f"  batches    : {len(api_data['batches'])}")

# Step 2: Flatten nested structure into tabular form
rows = []
for batch in api_data["batches"]:
    batch_id          = batch["batch_id"]
    merchant_id       = batch["merchant"]["merchant_id"]
    merchant_name     = batch["merchant"]["merchant_name"]
    merchant_region   = batch["merchant"]["region"]
    for s in batch["settlements"]:
        rows.append({
            "batch_id"        : batch_id,
            "merchant_id"     : merchant_id,
            "merchant_name"   : merchant_name,
            "merchant_region" : merchant_region,
            "settlement_id"   : s["settlement_id"],
            "amount_usd"      : s["amount_usd"],
            "status"          : s["status"],
            "processed_at"    : s["processed_at"],
            "bank_name"       : s["bank"]["name"],
            "bank_country"    : s["bank"]["country"],
        })

df_api = pd.DataFrame(rows)
print(f"\nStep 2 ✔ Flattened into {df_api.shape[0]} rows x {df_api.shape[1]} columns")

# Step 3: Clean column names (lowercase, strip spaces, replace spaces with _)
df_api.columns = (
    df_api.columns
    .str.strip()
    .str.lower()
    .str.replace(" ", "_", regex=False)
)
print("Step 3 ✔ Column names cleaned:", list(df_api.columns))

# Step 4: Convert date/time fields
df_api["processed_at"] = pd.to_datetime(df_api["processed_at"], utc=True)
print("Step 4 ✔ processed_at converted to datetime")
print(df_api[["settlement_id", "processed_at"]].head())

# Step 5: Save the normalized output
os.makedirs("01_data/processed", exist_ok=True)
df_api.to_csv("01_data/processed/api_normalized.csv", index=False)
print("\nStep 5 ✔ Saved: 01_data/processed/api_normalized.csv")

df_api.head()
```

    Step 1 ✔ JSON loaded
      source     : QuickPay Settlement API
      generated_at: 2026-03-07T10:00:00Z
      batches    : 2
    
    Step 2 ✔ Flattened into 6 rows x 10 columns
    Step 3 ✔ Column names cleaned: ['batch_id', 'merchant_id', 'merchant_name', 'merchant_region', 'settlement_id', 'amount_usd', 'status', 'processed_at', 'bank_name', 'bank_country']
    Step 4 ✔ processed_at converted to datetime
      settlement_id              processed_at
    0          S001 2026-03-07 08:10:00+00:00
    1          S002 2026-03-07 08:45:00+00:00
    2          S003 2026-03-07 09:15:00+00:00
    3          S004 2026-03-07 08:20:00+00:00
    4          S005 2026-03-07 08:50:00+00:00
    
    Step 5 ✔ Saved: 01_data/processed/api_normalized.csv
    




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>batch_id</th>
      <th>merchant_id</th>
      <th>merchant_name</th>
      <th>merchant_region</th>
      <th>settlement_id</th>
      <th>amount_usd</th>
      <th>status</th>
      <th>processed_at</th>
      <th>bank_name</th>
      <th>bank_country</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>B001</td>
      <td>M001</td>
      <td>Alpha Mart</td>
      <td>APAC</td>
      <td>S001</td>
      <td>1520.5</td>
      <td>settled</td>
      <td>2026-03-07 08:10:00+00:00</td>
      <td>Bank A</td>
      <td>IN</td>
    </tr>
    <tr>
      <th>1</th>
      <td>B001</td>
      <td>M001</td>
      <td>Alpha Mart</td>
      <td>APAC</td>
      <td>S002</td>
      <td>980.0</td>
      <td>pending</td>
      <td>2026-03-07 08:45:00+00:00</td>
      <td>Bank A</td>
      <td>IN</td>
    </tr>
    <tr>
      <th>2</th>
      <td>B001</td>
      <td>M001</td>
      <td>Alpha Mart</td>
      <td>APAC</td>
      <td>S003</td>
      <td>640.0</td>
      <td>settled</td>
      <td>2026-03-07 09:15:00+00:00</td>
      <td>Bank B</td>
      <td>SG</td>
    </tr>
    <tr>
      <th>3</th>
      <td>B002</td>
      <td>M004</td>
      <td>Delta Travels</td>
      <td>US</td>
      <td>S004</td>
      <td>2100.0</td>
      <td>settled</td>
      <td>2026-03-07 08:20:00+00:00</td>
      <td>Bank C</td>
      <td>US</td>
    </tr>
    <tr>
      <th>4</th>
      <td>B002</td>
      <td>M004</td>
      <td>Delta Travels</td>
      <td>US</td>
      <td>S005</td>
      <td>500.0</td>
      <td>failed</td>
      <td>2026-03-07 08:50:00+00:00</td>
      <td>Bank C</td>
      <td>US</td>
    </tr>
  </tbody>
</table>
</div>




```python
ledger = pd.read_csv("D:/01_data/raw/ledger.csv")
ledger["transaction_id"] = ledger["transaction_id"].astype(str).str.strip()

try:
    import json
    with open("04_python/api_raw.json") as f:
        api_raw = json.load(f)
    api_normalized = pd.json_normalize(api_raw["batches"], record_path="settlements",
                                       meta=["batch_id", "merchant_id", "merchant_name",
                                             ["merchant", "region"]])
    api_normalized.columns = [c.replace(".", "_") for c in api_normalized.columns]
    if "merchant_region" not in api_normalized.columns:
        api_normalized["merchant_region"] = api_normalized.get("merchant_region",
            pd.Series(dtype=str))
except Exception:
    pass

import pandas as pd
import os

os.makedirs("01_data/processed", exist_ok=True)

# ── 1. daily_summary.csv ──────────────────────────────────────────────────────
# Use ledger as the base (has transaction_date, amount_usd, status)
daily_summary = (
    ledger
    .groupby("transaction_date")
    .agg(
        total_transactions=("transaction_id", "count"),
        total_amount_usd=("amount_usd", "sum"),
        avg_amount_usd=("amount_usd", "mean"),
        successful_transactions=("status", lambda x: (x == "success").sum()),
        failed_transactions=("status", lambda x: (x == "failed").sum()),
    )
    .reset_index()
)
daily_summary["success_rate_%"] = (
    daily_summary["successful_transactions"] / daily_summary["total_transactions"] * 100
).round(2)
daily_summary.to_csv("01_data/processed/daily_summary.csv", index=False)
print("Saved: 01_data/processed/daily_summary.csv")
print(daily_summary)

# ── 2. payment_method_breakdown.csv ──────────────────────────────────────────
payment_method_breakdown = (
    ledger
    .groupby("payment_method")
    .agg(
        total_transactions=("transaction_id", "count"),
        total_amount_usd=("amount_usd", "sum"),
        avg_amount_usd=("amount_usd", "mean"),
        successful_transactions=("status", lambda x: (x == "success").sum()),
        failed_transactions=("status", lambda x: (x == "failed").sum()),
    )
    .reset_index()
)
payment_method_breakdown["success_rate_%"] = (
    payment_method_breakdown["successful_transactions"] / payment_method_breakdown["total_transactions"] * 100
).round(2)
payment_method_breakdown.to_csv("01_data/processed/payment_method_breakdown.csv", index=False)
print("\nSaved: 01_data/processed/payment_method_breakdown.csv")
print(payment_method_breakdown)

# ── 3. region_breakdown.csv ───────────────────────────────────────────────────
# Use api_normalized (has merchant_region) if available, else derive from ledger + gateway
try:
    region_breakdown = (
        api_normalized
        .groupby("merchant_region")
        .agg(
            total_transactions=("settlement_id", "count"),
            total_amount_usd=("amount_usd", "sum"),
            avg_amount_usd=("amount_usd", "mean"),
            successful_transactions=("status", lambda x: (x == "settled").sum()),
            failed_transactions=("status", lambda x: (x == "failed").sum()),
        )
        .reset_index()
        .rename(columns={"merchant_region": "region"})
    )
except NameError:
    # Fallback: assign region from merchant_id mapping using ledger
    region_map = {"M001": "APAC", "M002": "US", "M003": "EMEA", "M004": "US", "M005": "APAC"}
    ledger_r = ledger.copy()
    ledger_r["region"] = ledger_r["merchant_id"].map(region_map).fillna("Unknown")
    region_breakdown = (
        ledger_r
        .groupby("region")
        .agg(
            total_transactions=("transaction_id", "count"),
            total_amount_usd=("amount_usd", "sum"),
            avg_amount_usd=("amount_usd", "mean"),
            successful_transactions=("status", lambda x: (x == "success").sum()),
            failed_transactions=("status", lambda x: (x == "failed").sum()),
        )
        .reset_index()
    )
region_breakdown["success_rate_%"] = (
    region_breakdown["successful_transactions"] / region_breakdown["total_transactions"] * 100
).round(2)
region_breakdown.to_csv("01_data/processed/region_breakdown.csv", index=False)
print("\nSaved: 01_data/processed/region_breakdown.csv")
print(region_breakdown)

# ── 4. merchant_performance_summary.csv ──────────────────────────────────────
merchant_perf = (
    ledger
    .groupby("merchant_id")
    .agg(
        total_transactions=("transaction_id", "count"),
        total_amount_usd=("amount_usd", "sum"),
        avg_amount_usd=("amount_usd", "mean"),
        successful_transactions=("status", lambda x: (x == "success").sum()),
        failed_transactions=("status", lambda x: (x == "failed").sum()),
    )
    .reset_index()
)
merchant_perf["success_rate_%"] = (
    merchant_perf["successful_transactions"] / merchant_perf["total_transactions"] * 100
).round(2)

# Enrich with merchant_name and region if api_normalized is available
try:
    merchant_meta = (
        api_normalized[["merchant_id", "merchant_name", "merchant_region"]]
        .drop_duplicates(subset="merchant_id")
    )
    merchant_perf = merchant_perf.merge(merchant_meta, on="merchant_id", how="left")
except NameError:
    pass

merchant_perf.to_csv("01_data/processed/merchant_performance_summary.csv", index=False)
print("\nSaved: 01_data/processed/merchant_performance_summary.csv")
print(merchant_perf)

print("\nAll 4 processed CSV files generated successfully!")
```

    Saved: 01_data/processed/daily_summary.csv
      transaction_date  total_transactions  total_amount_usd  avg_amount_usd  \
    0       2026-03-01                   2            2050.0          1025.0   
    1       2026-03-02                   2            2600.0          1300.0   
    2       2026-03-03                   2            8150.0          4075.0   
    3       2026-03-04                   2            3940.0          1970.0   
    4       2026-03-05                   2            6600.0          3300.0   
    
       successful_transactions  failed_transactions  success_rate_%  
    0                        2                    0           100.0  
    1                        2                    0           100.0  
    2                        2                    0           100.0  
    3                        1                    1            50.0  
    4                        2                    0           100.0  
    
    Saved: 01_data/processed/payment_method_breakdown.csv
      payment_method  total_transactions  total_amount_usd  avg_amount_usd  \
    0           Card                   5           14890.0          2978.0   
    1     NetBanking                   1            3300.0          3300.0   
    2            UPI                   2            2150.0          1075.0   
    3         Wallet                   2            3000.0          1500.0   
    
       successful_transactions  failed_transactions  success_rate_%  
    0                        5                    0           100.0  
    1                        0                    1             0.0  
    2                        2                    0           100.0  
    3                        2                    0           100.0  
    
    Saved: 01_data/processed/region_breakdown.csv
      region  total_transactions  total_amount_usd  avg_amount_usd  \
    0   APAC                   4            5640.0          1410.0   
    1   EMEA                   1            2100.0          2100.0   
    2     US                   5           15600.0          3120.0   
    
       successful_transactions  failed_transactions  success_rate_%  
    0                        3                    1            75.0  
    1                        1                    0           100.0  
    2                        5                    0           100.0  
    
    Saved: 01_data/processed/merchant_performance_summary.csv
      merchant_id  total_transactions  total_amount_usd  avg_amount_usd  \
    0        M001                   3            2340.0      780.000000   
    1        M002                   3            5900.0     1966.666667   
    2        M003                   1            2100.0     2100.000000   
    3        M004                   2            9700.0     4850.000000   
    4        M005                   1            3300.0     3300.000000   
    
       successful_transactions  failed_transactions  success_rate_%  
    0                        3                    0           100.0  
    1                        3                    0           100.0  
    2                        1                    0           100.0  
    3                        2                    0           100.0  
    4                        0                    1             0.0  
    
    All 4 processed CSV files generated successfully!
    
