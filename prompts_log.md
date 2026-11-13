# prompts\_log.md - Session 3

`[TODO]` = only you have this information (exact text sent, code produced, exact model version, checks you ran). Fill it in before handing in, then delete the marker. Log what you really sent, even if it was weak, and do not edit entries afterwards.

## Entry 1 • Model: ChatGPT6.1\_Sol

System prompt:

```text
you are a data analysis assitant who cleans raw data csv file before modeling, you follow the rules listed: 
1.examine all data before cleaning, record and show problems, then decide and clean data.
2.check how many lines the dataset contains
3.do not delete data without reason, everytime you delete data, show the reason.
4.never drop rows silently: print row count for each removal
5.do not use dropna() without reporting
6.do not change raw dataframe, create new copies instead
if there is no clean task, do not start working
give results in this structure: (1) Problems found: numbered list with column, issue and number of rows affected; (2) Code: one commented Python block, one step per comment; (3) Results: raw rows, rows removed per rule, final row count. 
```

Context note:

```text
Audit trips_lyon.csv, list the problems you find, then write pandas code
that cleans the file. Handle outliers and report the final row count.
```

Prompt:

```text
Audit trips_lyon.csv, list the problems you find, then write pandas code
that cleans the file. Handle outliers and report the final row count.
```

Output:

Code:

```python
import numpy as np
import pandas as pd

# 读取原始数据，并在副本中标准化空白。
raw = pd.read_csv("trips_lyon.csv", dtype="string", keep_default_na=False)
trips = raw.copy()
for col in trips.columns:
    trips[col] = trips[col].str.strip().replace("", pd.NA)

# 记录问题及受影响行数。各项可能涉及同一行。
audit = []

def report(column, issue, mask):
    audit.append((column, issue, int(mask.fillna(False).sum())))

for col in trips.columns:
    report(col, "缺失", trips[col].isna())

duplicate = trips.duplicated()
report("整行", "重复记录（首条之后）", duplicate)

identified = trips["trip_id"].notna() & ~duplicate
conflicts = trips.loc[identified, "trip_id"].duplicated(keep=False)
report("trip_id", "去除整行重复后，同一 ID 仍对应多条记录", conflicts)

# 校验时间和站点格式，保留无法确认的站点代码供复核。
time_text = trips["start_time"]
time_format = time_text.str.fullmatch(
    r"[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}",
    na=False,
)
trips["start_time"] = pd.to_datetime(
    time_text.where(time_format), format="%Y-%m-%d %H:%M:%S", errors="coerce"
)
report("start_time", "格式异常或无效日期", time_text.notna() & trips["start_time"].isna())

for col in ["start_station_id", "end_station_id"]:
    bad_code = trips[col].notna() & ~trips[col].str.fullmatch(r"ST-[0-9]+", na=False)
    report(col, "站点代码格式异常", bad_code)
    trips[f"{col}_format_flag"] = bad_code

# 将无效数值置空，并单独统计各项检查结果。
numeric_cols = [
    "duration_minutes", "distance_km", "distance_meters",
    "battery_start_pct", "price_eur", "applied_rate_per_min",
]
originally_missing = trips[numeric_cols].isna()
for col in numeric_cols:
    parsed = pd.to_numeric(trips[col], errors="coerce")
    finite = parsed.notna() & ~parsed.isin([np.inf, -np.inf])
    report(col, "非数值或无穷值", trips[col].notna() & ~finite)

    invalid = parsed.lt(0)
    if col == "duration_minutes":
        invalid = parsed.le(0)
    elif col == "battery_start_pct":
        invalid = parsed.lt(0) | parsed.gt(100)

    out_of_range = finite & invalid.fillna(False)
    report(col, "超出有效范围", out_of_range)
    trips[col] = parsed.where(finite & ~out_of_range)

# 检查双单位距离。示例允许 1 米的舍入差。
km = trips["distance_km"]
meters = trips["distance_meters"]
mismatch = ((km * 1000 - meters).abs() > 1).fillna(False)
report("distance_km / distance_meters", "换算差超过 1 米", mismatch)
trips["distance_mismatch_flag"] = mismatch

# 按顺序删除重复行及缺少行程 ID 的行，并报告每次实际删除数。
clean = trips.loc[~duplicate].copy()
missing_id = clean["trip_id"].isna()
removed_missing_id = int(missing_id.sum())
clean = clean.loc[~missing_id].copy()
clean["trip_id_conflict_flag"] = clean["trip_id"].duplicated(keep=False)

# 从有效的另一列补齐原本缺失的距离。
fill_km = originally_missing.loc[clean.index, "distance_km"]
fill_meters = originally_missing.loc[clean.index, "distance_meters"]
clean.loc[fill_km, "distance_km"] = clean.loc[fill_km, "distance_meters"] / 1000
clean.loc[fill_meters, "distance_meters"] = clean.loc[fill_meters, "distance_km"] * 1000

# 用 1.5 倍 IQR 标记异常。距离阈值排除两列距离不一致的记录。
for col in ["duration_minutes", "distance_km"]:
    eligible = clean[col].notna()
    if col == "distance_km":
        eligible &= ~clean["distance_mismatch_flag"]

    sample = clean.loc[eligible, col]
    flag = pd.Series(False, index=clean.index)
    if len(sample) >= 4:
        q1, q3 = sample.quantile([0.25, 0.75])
        iqr = q3 - q1
        if iqr > 0:
            flag.loc[sample.index] = (
                sample.lt(q1 - 1.5 * iqr) | sample.gt(q3 + 1.5 * iqr)
            )
    clean[f"{col}_outlier_flag"] = flag
    report(col, "IQR 异常（基于保留行）", flag)

# 输出审计表、行数核对和清洗文件。
print(pd.DataFrame(audit, columns=["字段", "问题", "受影响行数"]).to_string(index=False))
print(f"原始行数 = {len(raw)}")
print(f"删除重复行 = {int(duplicate.sum())}")
print(f"删除缺少 trip_id 的行（去重后） = {removed_missing_id}")
print(f"最终行数 = {len(clean)}")
assert len(clean) == len(raw) - int(duplicate.sum()) - removed_missing_id
clean.to_csv("trips_lyon_clean.csv", index=False)
```

Printed result (from ChatGPT6.1\_Sol\_clean\_result.txt):

```text
字段                    问题  受影响行数
                      trip_id                    缺失      0
                      user_id                    缺失      0
                   start_time                    缺失      0
             start_station_id                    缺失      0
               end_station_id                    缺失    407
                 vehicle_type                    缺失      0
             duration_minutes                    缺失      0
                  distance_km                    缺失      0
              distance_meters                    缺失      0
            battery_start_pct                    缺失      0
                    price_eur                    缺失      0
         applied_rate_per_min                    缺失      0
                           整行            重复记录（首条之后）      0
                      trip_id 去除整行重复后，同一 ID 仍对应多条记录      0
                   start_time             格式异常或无效日期      0
             start_station_id              站点代码格式异常      0
               end_station_id              站点代码格式异常      0
             duration_minutes               非数值或无穷值      0
             duration_minutes                超出有效范围    109
                  distance_km               非数值或无穷值      0
                  distance_km                超出有效范围      0
              distance_meters               非数值或无穷值      0
              distance_meters                超出有效范围      0
            battery_start_pct               非数值或无穷值      0
            battery_start_pct                超出有效范围      0
                    price_eur               非数值或无穷值      0
                    price_eur                超出有效范围      0
         applied_rate_per_min               非数值或无穷值      0
         applied_rate_per_min                超出有效范围      0
distance_km / distance_meters             换算差超过 1 米      0
             duration_minutes         IQR 异常（基于保留行）    233
                  distance_km         IQR 异常（基于保留行）    223
原始行数 = 7000
删除重复行 = 0
删除缺少 trip_id 的行（去重后） = 0
最终行数 = 7000
```

Audit:

* Right: raw rows 7,000 match my baseline.
* Right: the 407 missing end\_station\_id are found and not dropped (final count 7,000), consistent with my keep-and-flag decision. \[TODO: check the code creates an outside\_station flag]
* Right: 109 out-of-range durations = my 109 negative durations.
* Right: 0 duplicates and 0 battery\_start\_pct problems, same as my Session 2 findings.
* Wrong: no vehicle\_type spelling problem reported, although the file has 6 spellings for 3 vehicles. Labels are probably still 6. \[TODO: confirm with value\_counts() on its output]
* Wrong: the 96 trips of 24 hours or more are not counted as invalid (range issues = 109, my rule gives 205).
* Wrong: final row count is 7,000 vs my 6,795. The negative durations are detected but not removed. \[TODO: confirm in the code]
* Wrong: IQR flags 233 duration rows "on retained rows" (all 7,000 here) vs my 160 on the cleaned durations. Different row set, not comparable. \[TODO: find the line that sets the row set and the k value]
* Line of code responsible for each mismatch: \[TODO]

Correction:

```python
d = df["duration_minutes"]
valid = (d > 0) & (d < 1440)
df_clean = df[valid].copy()          # expected: 6,795 rows
df_clean["outside_station"] = df_clean["end_station_id"].isna()
v = df_clean["vehicle_type"].str.lower().replace(
    {"electric_bike": "e-bike", "trottinette": "scooter"})
df_clean["vehicle_type"] = v          # expected: 3 labels
```

## Entry 2 • Model: Gemini3.5flash

System prompt: \[TODO: paste the exact text you sent, same as Entry 1]

Context note:

```text
Urbanmove is a shared bike company in Lyon, I want to study the pricing model of this company and how the bikes and scooters are used. Here is the definition of each column in trips_lyon.csv
File trips_lyon.csv: one row = one trip made by one user; 12 columns.
trip_id: the unique identifier of the trip. user_id: identifier of the customer.
start_time: timestamp of the trip start, recorded as text (format YYYY-MM-DD HH:MM:SS).
start_station_id, end_station_id: codes (format ST-100) of the station where the trip started and ended; they link to stations_metadata.csv.
vehicle_type: type of vehicle used, stored as text.
duration_minutes: time duration of the trip, in minutes.
distance_km: the distance user travelled, in kilometres. distance_meters: the distance user travelled, in metres.
battery_start_pct: battery charge of the vehicle at the start of the trip, in percent.
price_eur: price paid for the trip, in euros. applied_rate_per_min: price per minute applied to the trip, in euros per minute.
```

Prompt:

```text
Audit trips_lyon.csv, list the problems you find, then write pandas code
that cleans the file. Handle outliers and report the final row count.
```

Output:

Code:

```python
import pandas as pd
import numpy as np

# 1. Load raw data and inspect total row and column counts without modifying the original DataFrame
df_raw = pd.read_csv("trips_lyon.csv")
print(f"Raw dataset shape: {df_raw.shape[0]} rows, {df_raw.shape[1]} columns")

# 2. Audit all columns before cleaning and record data quality issues
audit_issues = []
audit_issues.append(("trip_id", "Missing or duplicate trip_id", df_raw["trip_id"].isna().sum() + df_raw["trip_id"].duplicated().sum()))
audit_issues.append(("user_id", "Missing user_id", df_raw["user_id"].isna().sum()))
parsed_time = pd.to_datetime(df_raw["start_time"], format="%Y-%m-%d %H:%M:%S", errors="coerce")
audit_issues.append(("start_time", "Missing or invalid timestamp format", parsed_time.isna().sum()))
valid_start_st = df_raw["start_station_id"].astype(str).str.match(r"^ST-\d+$", na=False)
valid_end_st = df_raw["end_station_id"].astype(str).str.match(r"^ST-\d+$", na=False)
audit_issues.append(("start_station_id", "Missing or invalid station format (expected ST-XXX)", (~valid_start_st).sum()))
audit_issues.append(("end_station_id", "Missing or invalid station format (expected ST-XXX)", (~valid_end_st).sum()))
audit_issues.append(("vehicle_type", "Missing or unstandardized casing/whitespace", df_raw["vehicle_type"].isna().sum() + (df_raw["vehicle_type"].dropna() != df_raw["vehicle_type"].dropna().str.strip().str.lower()).sum()))
audit_issues.append(("duration_minutes", "Missing, non-positive (<= 0), or extreme outliers (> 1440 min)", (df_raw["duration_minutes"].isna() | (df_raw["duration_minutes"] <= 0) | (df_raw["duration_minutes"] > 1440)).sum()))
audit_issues.append(("distance_km / distance_meters", "Missing, negative (< 0), or unit mismatch (> 0.05 km diff)", (df_raw["distance_km"].isna() | df_raw["distance_meters"].isna() | (df_raw["distance_km"] < 0) | (df_raw["distance_meters"] < 0) | ((df_raw["distance_km"] - df_raw["distance_meters"] / 1000.0).abs() > 0.05)).sum()))
audit_issues.append(("battery_start_pct", "Missing or out of valid range [0, 100]", (df_raw["battery_start_pct"].isna() | (df_raw["battery_start_pct"] < 0) | (df_raw["battery_start_pct"] > 100)).sum()))
audit_issues.append(("price_eur / applied_rate_per_min", "Missing or negative pricing values (< 0)", (df_raw["price_eur"].isna() | df_raw["applied_rate_per_min"].isna() | (df_raw["price_eur"] < 0) | (df_raw["applied_rate_per_min"] < 0)).sum()))

print("\n--- (1) Problems Found (Pre-Cleaning Audit) ---")
for idx, (col, desc, count) in enumerate(audit_issues, 1):
    print(f"{idx}. Column: {col} | Issue: {desc} | Rows affected: {count}")

# 3. Create a working copy so the raw DataFrame remains untouched
df_clean = df_raw.copy()
removal_log = []

# 4. Report and drop rows with missing critical identifiers or duplicate trip_id
before = len(df_clean)
missing_ids = df_clean["trip_id"].isna() | df_clean["user_id"].isna()
print(f"\n[Cleaning Step] Missing trip_id or user_id count: {missing_ids.sum()}")
df_clean = df_clean[~missing_ids].drop_duplicates(subset=["trip_id"], keep="first")
removed = before - len(df_clean)
removal_log.append(("Missing trip_id/user_id or duplicate trip_id", removed))
print(f"Removed {removed} rows -> Reason: Primary/foreign keys cannot be null or duplicated. Remaining: {len(df_clean)}")

# 5. Parse start_time to datetime and remove rows with unparseable timestamps
before = len(df_clean)
df_clean["start_time"] = pd.to_datetime(df_clean["start_time"], format="%Y-%m-%d %H:%M:%S", errors="coerce")
invalid_time = df_clean["start_time"].isna()
print(f"[Cleaning Step] Invalid/missing start_time count: {invalid_time.sum()}")
df_clean = df_clean[~invalid_time]
removed = before - len(df_clean)
removal_log.append(("Missing or invalid start_time format", removed))
print(f"Removed {removed} rows -> Reason: Invalid timestamp format (expected YYYY-MM-DD HH:MM:SS). Remaining: {len(df_clean)}")

# 6. Validate station ID format (ST-XXX) and drop rows with missing or malformed station codes
before = len(df_clean)
df_clean["start_station_id"] = df_clean["start_station_id"].astype(str).str.strip().str.upper()
df_clean["end_station_id"] = df_clean["end_station_id"].astype(str).str.strip().str.upper()
valid_stations = df_clean["start_station_id"].str.match(r"^ST-\d+$") & df_clean["end_station_id"].str.match(r"^ST-\d+$")
df_clean = df_clean[valid_stations]
removed = before - len(df_clean)
removal_log.append(("Invalid or missing start_station_id / end_station_id", removed))
print(f"Removed {removed} rows -> Reason: Station IDs must match format ST-XXX to link with stations_metadata.csv. Remaining: {len(df_clean)}")

# 7. Standardize vehicle_type text and drop rows where vehicle_type is missing
before = len(df_clean)
missing_vehicle = df_clean["vehicle_type"].isna()
print(f"[Cleaning Step] Missing vehicle_type count: {missing_vehicle.sum()}")
df_clean = df_clean[~missing_vehicle]
df_clean["vehicle_type"] = df_clean["vehicle_type"].astype(str).str.strip().str.lower()
removed = before - len(df_clean)
removal_log.append(("Missing vehicle_type", removed))
print(f"Removed {removed} rows -> Reason: Vehicle type is required to analyze bike vs. scooter usage and pricing. Remaining: {len(df_clean)}")

# 8. Reconcile distance_km and distance_meters, drop negative values, and reconcile unit mismatches
before = len(df_clean)
both_dist_missing = df_clean["distance_km"].isna() & df_clean["distance_meters"].isna()
print(f"[Cleaning Step] Both distance_km and distance_meters missing: {both_dist_missing.sum()}")
df_clean = df_clean[~both_dist_missing]
df_clean["distance_km"] = df_clean["distance_km"].fillna(df_clean["distance_meters"] / 1000.0)
df_clean["distance_meters"] = df_clean["distance_meters"].fillna(df_clean["distance_km"] * 1000.0)
invalid_dist = (df_clean["distance_km"] < 0) | (df_clean["distance_meters"] < 0) | ((df_clean["distance_km"] - df_clean["distance_meters"] / 1000.0).abs() > 0.05)
df_clean = df_clean[~invalid_dist]
removed = before - len(df_clean)
removal_log.append(("Missing, negative, or conflicting distance_km & distance_meters", removed))
print(f"Removed {removed} rows -> Reason: Negative distances or irreconcilable km/meter discrepancies (> 50m). Remaining: {len(df_clean)}")

# 9. Remove missing or out-of-range battery_start_pct values outside [0, 100]
before = len(df_clean)
invalid_battery = df_clean["battery_start_pct"].isna() | (df_clean["battery_start_pct"] < 0) | (df_clean["battery_start_pct"] > 100)
print(f"[Cleaning Step] Missing or out-of-range battery_start_pct count: {invalid_battery.sum()}")
df_clean = df_clean[~invalid_battery]
removed = before - len(df_clean)
removal_log.append(("Missing or invalid battery_start_pct (<0 or >100)", removed))
print(f"Removed {removed} rows -> Reason: Battery percentage must be within [0, 100]. Remaining: {len(df_clean)}")

# 10. Remove missing, non-positive, or extreme duration_minutes and negative pricing records
before = len(df_clean)
invalid_duration_price = (
    df_clean["duration_minutes"].isna() | (df_clean["duration_minutes"] <= 0) |
    df_clean["price_eur"].isna() | (df_clean["price_eur"] < 0) |
    df_clean["applied_rate_per_min"].isna() | (df_clean["applied_rate_per_min"] < 0)
)
print(f"[Cleaning Step] Invalid duration_minutes (<=0/NaN) or negative/NaN pricing count: {invalid_duration_price.sum()}")
df_clean = df_clean[~invalid_duration_price]
removed = before - len(df_clean)
removal_log.append(("Non-positive duration_minutes or negative/missing price/rate", removed))
print(f"Removed {removed} rows -> Reason: Duration must be > 0 and pricing fields must be non-negative and non-null. Remaining: {len(df_clean)}")

# 11. Detect and remove statistical outliers in duration_minutes, distance_km, and price_eur using IQR per vehicle_type
before = len(df_clean)
outlier_mask = pd.Series(False, index=df_clean.index)
for col in ["duration_minutes", "distance_km", "price_eur"]:
    q1 = df_clean.groupby("vehicle_type")[col].transform(lambda s: s.quantile(0.25))
    q3 = df_clean.groupby("vehicle_type")[col].transform(lambda s: s.quantile(0.75))
    iqr = q3 - q1
    lower_bound = q1 - 3.0 * iqr
    upper_bound = q3 + 3.0 * iqr
    outlier_mask = outlier_mask | (df_clean[col] < lower_bound) | (df_clean[col] > upper_bound)

df_clean = df_clean[~outlier_mask]
removed = before - len(df_clean)
removal_log.append(("Extreme outliers (3*IQR by vehicle_type) in duration, distance, or price", removed))
print(f"Removed {removed} rows -> Reason: Extreme statistical outliers distort pricing model calibration. Remaining: {len(df_clean)}")

# 12. Export cleaned dataset and print final results summary
df_clean.to_csv("trips_lyon_clean.csv", index=False)

print("\n--- (3) Results Summary ---")
print(f"Raw rows: {len(df_raw)}")
for rule, count in removal_log:
    print(f" - Removed ({rule}): {count}")
print(f"Total rows removed: {len(df_raw) - len(df_clean)}")
print(f"Final row count: {len(df_clean)}")
```

Printed result (from Gemini3.5flash\_clean\_results.txt):

```text
Raw dataset shape: 7000 rows, 12 columns

--- (1) Problems Found (Pre-Cleaning Audit) ---
1. Column: trip_id | Issue: Missing or duplicate trip_id | Rows affected: 0
2. Column: user_id | Issue: Missing user_id | Rows affected: 0
3. Column: start_time | Issue: Missing or invalid timestamp format | Rows affected: 0
4. Column: start_station_id | Issue: Missing or invalid station format (expected ST-XXX) | Rows affected: 0
5. Column: end_station_id | Issue: Missing or invalid station format (expected ST-XXX) | Rows affected: 407
6. Column: vehicle_type | Issue: Missing or unstandardized casing/whitespace | Rows affected: 2257
7. Column: duration_minutes | Issue: Missing, non-positive (<= 0), or extreme outliers (> 1440 min) | Rows affected: 205
8. Column: distance_km / distance_meters | Issue: Missing, negative (< 0), or unit mismatch (> 0.05 km diff) | Rows affected: 0
9. Column: battery_start_pct | Issue: Missing or out of valid range [0, 100] | Rows affected: 0
10. Column: price_eur / applied_rate_per_min | Issue: Missing or negative pricing values (< 0) | Rows affected: 0

[Cleaning Step] Missing trip_id or user_id count: 0
Removed 0 rows -> Reason: Primary/foreign keys cannot be null or duplicated. Remaining: 7000
[Cleaning Step] Invalid/missing start_time count: 0
Removed 0 rows -> Reason: Invalid timestamp format (expected YYYY-MM-DD HH:MM:SS). Remaining: 7000
Removed 407 rows -> Reason: Station IDs must match format ST-XXX to link with stations_metadata.csv. Remaining: 6593
[Cleaning Step] Missing vehicle_type count: 0
Removed 0 rows -> Reason: Vehicle type is required to analyze bike vs. scooter usage and pricing. Remaining: 6593
[Cleaning Step] Both distance_km and distance_meters missing: 0
Removed 0 rows -> Reason: Negative distances or irreconcilable km/meter discrepancies (> 50m). Remaining: 6593
[Cleaning Step] Missing or out-of-range battery_start_pct count: 0
Removed 0 rows -> Reason: Battery percentage must be within [0, 100]. Remaining: 6593
[Cleaning Step] Invalid duration_minutes (<=0/NaN) or negative/NaN pricing count: 106
Removed 106 rows -> Reason: Duration must be > 0 and pricing fields must be non-negative and non-null. Remaining: 6487
Removed 93 rows -> Reason: Extreme statistical outliers distort pricing model calibration. Remaining: 6394

--- (3) Results Summary ---
Raw rows: 7000
 - Removed (Missing trip_id/user_id or duplicate trip_id): 0
 - Removed (Missing or invalid start_time format): 0
 - Removed (Invalid or missing start_station_id / end_station_id): 407
 - Removed (Missing vehicle_type): 0
 - Removed (Missing, negative, or conflicting distance_km & distance_meters): 0
 - Removed (Missing or invalid battery_start_pct (<0 or >100)): 0
 - Removed (Non-positive duration_minutes or negative/missing price/rate): 106
 - Removed (Extreme outliers (3*IQR by vehicle_type) in duration, distance, or price): 93
Total rows removed: 606
Final row count: 6394
```

Audit:

* Right: raw rows 7,000 match my baseline.
* Right: the audit counts 205 duration problems (<= 0 or > 1,440 min) = my 109 negative + 96 long trips.
* Right: non-positive durations are removed (106 rows).
* Right: 0 duplicates and 0 battery\_start\_pct problems, same as my Session 2 findings.
* Wrong: it deleted the 407 trips with an empty end\_station\_id, labelled "invalid station format". These are free-floating rides, a business signal; my decision was keep and flag. This is the dropna()-type error from the slides.
* Wrong: vehicle\_type flags 2,257 rows = 1,101 (E-BIKE) + 1,156 (TROTTINETTE) from my Session 2 table, so only the casing was detected. electric\_bike (1,208 rows) and trottinette are probably still separate labels, i.e. 5 labels instead of 3. \[TODO: confirm with value\_counts() on its output]
* Wrong: 106 removed instead of 109 negatives, 93 instead of 96 long trips. Likely cause: 3 + 3 rows were already deleted in the 407-row step. \[TODO: verify, hypothesis only]
* Wrong: the 93 "extreme outliers" use a 3 x IQR rule by vehicle\_type on duration, distance and price. Deleting real but extreme rides goes against my rule "flagged does not mean deleted".
* Wrong: final row count 6,394 vs my 6,795 (401 fewer). The k = 1.5 IQR count (my 160) is never reported.
* Line of code responsible for each mismatch: \[TODO]

Correction: \[TODO: paste your corrected code and why. Same baseline code as Entry 1.]

## Entry 3 • Model: ChatGPT 6.1 SOL • Round 2, framing version 2

System prompt:

```text
You are a senior data analyst helping a junior analyst at UrbanMove Analytics clean a shared bike and scooter dataset with Python and pandas.
Write pandas code that runs top to bottom in a Jupyter notebook, on the file I name. Use only the columns I list.
Never drop, fill or change rows silently: for every cleaning step print what was found, how many rows are affected, what you do, why, and the row count after the step.
Rule 1: a trip is valid only if 0 < duration_minutes < 1440 (less than 24 hours). Remove all other rows.
Rule 2: vehicle_type has exactly 3 valid values: e-bike, scooter, mechanical_bike. Unify every spelling variant into them and print value_counts() afterwards.
Rule 3: an empty end_station_id means the vehicle was left outside a station. Keep these rows and add a boolean column outside_station. Never drop or fill them.
Rule 4: for outliers, compute IQR fences (k = 1.5) on the cleaned duration_minutes. Flag them in a column and report the count. Do not delete them.
If something is ambiguous, state your assumption instead of guessing.
Answer with: (a) the list of problems with affected rows, (b) the code, (c) a before/after row-count summary.
```

\[TODO: confirm this is exactly what you sent (= system\_prompt\_v2.md). The code comments refer to "Rule 1" to "Rule 4", which matches this text]

Context note:

```text
Company: UrbanMove Analytics, a shared bike and scooter operator in French cities.
Business question: where and when is the fleet used, and what drives the price of a trip?
File: trips_lyon.csv, 7,000 rows and 12 columns. One row = one trip (one rental of one vehicle). I describe the schema only; no data is pasted.
trip_id: unique trip code. user_id: customer code.
start_time: trip start, text in the format YYYY-MM-DD HH:MM:SS.
start_station_id, end_station_id: station codes like ST-100, matching station_id in stations_metadata.csv.
vehicle_type: type of vehicle, text label.
duration_minutes: trip length in minutes. distance_km: distance in kilometres. distance_meters: the same distance in metres.
battery_start_pct: battery level at the start of the trip, in percent (0 to 100).
price_eur: trip price in euros. applied_rate_per_min: price rate in euros per minute.
```

\[TODO: confirm this is exactly what you sent (= context\_note\_v2.md)]

Prompt:

```text
Audit trips_lyon.csv, list the problems you find, then write pandas code
that cleans the file. Handle outliers and report the final row count.
```

Output:

Code (gpt code.py, unedited):

```python
import pandas as pd
import numpy as np

# Explicitly set the CSV path. This script does not search your computer.
file_path = 'trips_lyon.csv'
df = pd.read_csv(file_path)
original_rows = len(df)
audit = []


def log_step(step, affected, action, reason):
    audit.append({'Step': step, 'Affected rows': int(affected), 'Action': action,
                  'Reason': reason, 'Rows remaining': len(df)})
    print(f'\n{step}: {affected} affected | {action} | {reason} | rows remaining: {len(df)}')

print(f'Original rows: {original_rows}; columns: {len(df.columns)}')
print('Missing values:\n', df.isna().sum())

# Rule 1: duration strictly between 0 and 1440 minutes.
df['duration_minutes'] = pd.to_numeric(df['duration_minutes'], errors='coerce')
invalid = df['duration_minutes'].isna() | (df['duration_minutes'] <= 0) | (df['duration_minutes'] >= 1440)
n_invalid = int(invalid.sum())
df = df.loc[~invalid].copy()
log_step('Invalid duration', n_invalid, 'Removed', 'Must satisfy 0 < duration < 1440')

# Rule 2: normalize known variants; report unmapped values without guessing.
print('\nVehicle types before:\n', df['vehicle_type'].value_counts(dropna=False))
original = df['vehicle_type'].astype('string')
normalized = original.str.strip().str.lower().str.replace(r'[\s_-]+', '', regex=True)
vehicle_mapping = {
    'ebike': 'e-bike', 'electricbike': 'e-bike', 'electricbicycle': 'e-bike',
    'scooter': 'scooter', 'electricscooter': 'scooter', 'escooter': 'scooter',
    'mechanicalbike': 'mechanical_bike', 'mechanicalbicycle': 'mechanical_bike',
    'regularbike': 'mechanical_bike', 'normalbike': 'mechanical_bike'
}
mapped = normalized.map(vehicle_mapping).astype('string')
unmapped = mapped.isna() & original.notna()
print('Unmapped vehicle labels:\n', original[unmapped].value_counts(dropna=False))
# Preserve unknown original labels rather than silently replacing them with NaN.
df['vehicle_type'] = mapped.where(~unmapped, original)
changed = int((original.ne(df['vehicle_type']).fillna(False)).sum())
log_step('Vehicle type standardization', changed, 'Normalized known variants; retained unknown labels',
         'Unknown labels require review')
print('Vehicle types after:\n', df['vehicle_type'].value_counts(dropna=False))

# Rule 3: missing end station indicates an outside-station trip; retain all rows.
end = df['end_station_id'].astype('string')
df['outside_station'] = (end.isna() | end.str.strip().eq('').fillna(False)).astype(bool)
log_step('Outside-station trips', df['outside_station'].sum(), 'Flagged, retained',
         'Blank end station indicates parking outside station')

# Rule 4: IQR outliers on valid durations; flag, never delete.
q1 = df['duration_minutes'].quantile(.25)
q3 = df['duration_minutes'].quantile(.75)
iqr = q3 - q1
lower, upper = q1 - 1.5 * iqr, q3 + 1.5 * iqr
df['duration_outlier'] = (df['duration_minutes'] < lower) | (df['duration_minutes'] > upper)
print(f'IQR fences: {lower:.2f} to {upper:.2f}')
log_step('Duration outliers', df['duration_outlier'].sum(), 'Flagged, retained', '1.5 x IQR fences')

# Additional diagnostics only: no undocumented row deletions or corrections.
log_step('Duplicate trip IDs', df['trip_id'].duplicated(keep=False).sum(), 'Reported only', 'IDs should be unique')
times = pd.to_datetime(df['start_time'], format='%Y-%m-%d %H:%M:%S', errors='coerce')
log_step('Invalid start times', times.isna().sum(), 'Reported only', 'Expected YYYY-MM-DD HH:MM:SS')
km = pd.to_numeric(df['distance_km'], errors='coerce')
meters = pd.to_numeric(df['distance_meters'], errors='coerce')
mismatch = km.notna() & meters.notna() & ~np.isclose(km * 1000, meters, atol=1, rtol=0)
log_step('Distance mismatch', mismatch.sum(), 'Reported only', 'Tolerance: 1 metre')
battery = pd.to_numeric(df['battery_start_pct'], errors='coerce')
log_step('Invalid battery', (battery.isna() | (battery < 0) | (battery > 100)).sum(),
         'Reported only', 'Expected 0-100 percent')
price = pd.to_numeric(df['price_eur'], errors='coerce')
rate = pd.to_numeric(df['applied_rate_per_min'], errors='coerce')
bad_price = price.isna() | rate.isna() | (price < 0) | (rate < 0)
log_step('Invalid prices/rates', bad_price.sum(), 'Reported only', 'Nonnegative and nonmissing')
price_mismatch = (~bad_price) & ~np.isclose(price, df['duration_minutes'] * rate, atol=.02, rtol=0)
log_step('Price mismatch', price_mismatch.sum(), 'Reported only',
         'Assumes no fixed fees; verify pricing model')

final_rows = len(df)
print('\n=== BEFORE / AFTER ===')
print(f'Original: {original_rows} | Removed: {original_rows-final_rows} | Final: {final_rows}')
print(pd.DataFrame(audit).to_string(index=False))
df.to_csv('trips_lyon_cleaned.csv', index=False, encoding='utf-8-sig')
pd.DataFrame(audit).to_csv('trips_lyon_audit_report.csv', index=False, encoding='utf-8-sig')
print('Saved trips_lyon_cleaned.csv and trips_lyon_audit_report.csv')
```

Printed result (chatgpt output.txt, unedited):

```text
=== INITIAL DATA ===
Original rows: 7000
Original columns: 12

Missing values:
trip_id                   0
user_id                   0
start_time                0
start_station_id          0
end_station_id          407
vehicle_type              0
duration_minutes          0
distance_km               0
distance_meters           0
battery_start_pct         0
price_eur                 0
applied_rate_per_min      0
dtype: int64

--- Invalid trip duration ---
Affected rows: 205
Action: Removed invalid trips
Reason: Valid duration must be > 0 and < 1440 minutes
Rows remaining: 6795

Original vehicle types:
vehicle_type
electric_bike      1181
scooter            1153
mechanical_bike    1146
e-bike             1129
TROTTINETTE        1118
E-BIKE             1068
Name: count, dtype: int64

--- Vehicle type standardization ---
Affected rows: 2249
Action: Standardized recognized labels; unknown labels flagged
Reason: Only three canonical vehicle types are allowed
Rows remaining: 6795

Standardized vehicle types:
vehicle_type
e-bike             3378
scooter            1153
mechanical_bike    1146
NaN                1118
Name: count, dtype: int64
Unknown or missing types: 1118

--- Outside-station trips ---
Affected rows: 397
Action: Kept all rows; added outside_station flag
Reason: Missing end station indicates parking outside a station
Rows remaining: 6795

IQR lower fence: -7.90
IQR upper fence: 36.10

--- Duration outliers ---
Affected rows: 160
Action: Flagged outliers; no rows removed
Reason: IQR method with k = 1.5
Rows remaining: 6795

--- Duplicate trip IDs ---
Affected rows: 0
Action: Reported only
Reason: Trip IDs should be unique; no automatic deletion
Rows remaining: 6795

--- Invalid start times ---
Affected rows: 0
Action: Reported only
Reason: Start times must follow YYYY-MM-DD HH:MM:SS
Rows remaining: 6795

--- Distance consistency ---
Affected rows: 0
Action: Reported only
Reason: Kilometres and metres should agree within 1 metre
Rows remaining: 6795

--- Invalid battery percentage ---
Affected rows: 0
Action: Reported only
Reason: Battery percentage must be between 0 and 100
Rows remaining: 6795

--- Invalid prices or rates ---
Affected rows: 0
Action: Reported only
Reason: Prices and rates must be non-negative
Rows remaining: 6795

--- Price calculation differences ---
Affected rows: 6795
Action: Reported only
Reason: Assumes price = duration x rate; other fees may apply
Rows remaining: 6795

========== FINAL SUMMARY ==========
Original rows: 7000
Final rows: 6795
Rows removed: 205
Rows retained: 97.07%

Audit report:
                         Step  Affected rows                                                 Action                                                  Reason  Rows remaining
        Invalid trip duration            205                                  Removed invalid trips           Valid duration must be > 0 and < 1440 minutes            6795
 Vehicle type standardization           2249 Standardized recognized labels; unknown labels flagged          Only three canonical vehicle types are allowed            6795
        Outside-station trips            397              Kept all rows; added outside_station flag Missing end station indicates parking outside a station            6795
            Duration outliers            160                      Flagged outliers; no rows removed                                 IQR method with k = 1.5            6795
           Duplicate trip IDs              0                                          Reported only        Trip IDs should be unique; no automatic deletion            6795
          Invalid start times              0                                          Reported only             Start times must follow YYYY-MM-DD HH:MM:SS            6795
         Distance consistency              0                                          Reported only       Kilometres and metres should agree within 1 metre            6795
   Invalid battery percentage              0                                          Reported only                   Battery percentage must be between 0 and 100            6795
      Invalid prices or rates              0                                          Reported only                   Prices and rates must be non-negative            6795
Price calculation differences           6795                                          Reported only   Assumes price = duration x rate; other fees may apply            6795

Final vehicle distribution:
vehicle_type
e-bike             3378
scooter            1153
mechanical_bike    1146
NaN                1118
Name: count, dtype: int64

Final dataset shape:
(6795, 14)

Files saved:
trips_lyon_cleaned.csv
trips_lyon_audit_report.csv
```

Audit:

* Right: 7,000 rows loaded; the 407 missing end\_station\_id appear in the missing-values table.
* Right: 205 invalid durations removed (= my 109 negative + 96 of 24 hours or more; only the combined count is printed), leaving 6,795 rows, same as my baseline. \[TODO: verify the 109 / 96 split]
* Right: outside-station trips are kept and flagged with outside\_station. 397 are flagged instead of my 407, probably because 10 of those trips were among the 205 removed rows (407 - 397 = 10). \[TODO: verify]
* Right: IQR (k = 1.5) flags 160 duration outliers (fences -7.90 and 36.10) and removes none, same as my 160.
* Right: duplicates, start times, distances, battery and negative prices all report 0, consistent with my Session 2 findings.
* Wrong: TROTTINETTE (1,118 rows) is not recognised as scooter. The result has 4 values (e-bike 3,378, scooter 1,153, mechanical\_bike 1,146, NaN 1,118) instead of 3 labels with scooter = 1,153 + 1,118 = 2,271. The model does not know that "trottinette" is the French word for scooter, and my framing named the three valid labels but gave no mapping. Responsible line: the vehicle\_mapping dict in the Rule 2 block has no 'trottinette' key.
* Wrong (code vs printed result): in the submitted code, unknown labels are kept unchanged (`mapped.where(~unmapped, original)`), but the printed result shows them as NaN. The step names and print format also differ (e.g. "Price calculation differences" in the output, "Price mismatch" in the code). So the printed result was probably not produced by this exact code. \[TODO: run gpt code.py unedited on trips\_lyon.csv and compare with the printed result]
* Note: "Price calculation differences" flags all 6,795 rows: the assumption price = duration x rate fails on every row (tolerance 0.02 EUR in the code). It is report-only and not part of my baseline, but it matters for the business question on what drives the price. \[TODO: investigate]
* Progress vs Entry 1: final rows 7,000 -> 6,795; IQR flagged 233 -> 160; the 96 trips of 24 hours or more are now removed; vehicle labels are still not unified.

Correction: add the missing key to the mapping, then re-run \[TODO: confirm you ran it and got the expected counts]:

```python
vehicle_mapping = {
    # ... same entries as above ...
    'trottinette': 'scooter',   # French word for scooter, added after the audit
}
# expected after the fix: e-bike 3,378 | scooter 2,271 | mechanical_bike 1,146 | total 6,795
```

Why: Session 2 baseline maps electric\_bike -> e-bike and trottinette -> scooter. The rule to add to the framing: give the explicit mapping, because a model cannot guess a French label.

## Entry 4 • Model: Gemini \[TODO: exact name and version, same model as Entry 2] • Round 2, framing version 2

System prompt: \[TODO: paste the exact text you sent. Same as Entry 3 if you used system\_prompt\_v2.md; the code comments also refer to "Rule 1" to "Rule 4"]

Context note: \[TODO: paste the exact text you sent. Same as Entry 3 if you used context\_note\_v2.md]

Prompt:

```text
Audit trips_lyon.csv, list the problems you find, then write pandas code
that cleans the file. Handle outliers and report the final row count.
```

\[TODO: confirm this is exactly what you sent]

Output:

Code (gemini-code-1794562051624.py, unedited):

```python
import numpy as np
import pandas as pd

# ---------------------------------------------------------------------------
# 0. Load dataset & verify schema (only the 12 specified columns)
# ---------------------------------------------------------------------------
EXPECTED_COLS = [
    "trip_id",
    "user_id",
    "start_time",
    "start_station_id",
    "end_station_id",
    "vehicle_type",
    "duration_minutes",
    "distance_km",
    "distance_meters",
    "battery_start_pct",
    "price_eur",
    "applied_rate_per_min",
]

df_raw = pd.read_csv("trips_lyon.csv", usecols=EXPECTED_COLS)
initial_rows = len(df_raw)
df = df_raw.copy()

print(f"Loaded trips_lyon.csv | Initial row count: {initial_rows:,}\n")


def log_step(step_name, found_desc, affected_count, action_desc, reason_desc, current_df):
    """Standardized audit logger so no change occurs silently."""
    print(f"=== {step_name} ===")
    print(f"  Found         : {found_desc}")
    print(f"  Affected rows : {affected_count:,}")
    print(f"  Action        : {action_desc}")
    print(f"  Why           : {reason_desc}")
    print(f"  Row count now : {len(current_df):,}\n")


# ---------------------------------------------------------------------------
# Step 1: Audit & remove duplicate trip_id rows
# Assumption: trip_id is a unique identifier; keep first occurrence.
# ---------------------------------------------------------------------------
dup_mask = df.duplicated(subset=["trip_id"], keep="first")
dup_count = int(dup_mask.sum())
df = df.loc[~dup_mask].copy()
log_step(
    "Step 1: Duplicate trip_id check",
    f"{dup_count} duplicate trip_id records detected",
    dup_count,
    "Dropped duplicate rows keeping first occurrence" if dup_count else "No rows dropped",
    "Each row must represent one unique trip rental (trip_id is unique).",
    df,
)

# ---------------------------------------------------------------------------
# Step 2 (Rule 1): Filter valid trip durations (0 < duration_minutes < 1440)
# ---------------------------------------------------------------------------
df["duration_minutes"] = pd.to_numeric(df["duration_minutes"], errors="coerce")
invalid_dur_mask = (
    df["duration_minutes"].isna()
    | (df["duration_minutes"] <= 0)
    | (df["duration_minutes"] >= 1440)
)
invalid_dur_count = int(invalid_dur_mask.sum())
df = df.loc[~invalid_dur_mask].copy()
log_step(
    "Step 2 (Rule 1): Valid duration filter",
    f"{invalid_dur_count} rows with null, <= 0, or >= 1440 duration_minutes",
    invalid_dur_count,
    "Removed rows outside (0, 1440) minutes",
    "Rule 1 defines valid trips strictly as 0 < duration_minutes < 1440 (< 24h).",
    df,
)

# ---------------------------------------------------------------------------
# Step 3 (Rule 2): Standardize vehicle_type into 3 canonical labels
# ---------------------------------------------------------------------------
raw_vt = df["vehicle_type"].astype("string").str.strip().str.lower()
# Normalize separators (spaces/underscores/hyphens) for reliable mapping
norm_vt = raw_vt.str.replace(r"[\s_]+", "-", regex=True)

vehicle_map = {
    "e-bike": "e-bike",
    "ebike": "e-bike",
    "electric-bike": "e-bike",
    "electric-bicycle": "e-bike",
    "velo-electrique": "e-bike",
    "scooter": "scooter",
    "e-scooter": "scooter",
    "escooter": "scooter",
    "kick-scooter": "scooter",
    "trottinette": "scooter",
    "mechanical-bike": "mechanical_bike",
    "mechanicalbike": "mechanical_bike",
    "mech-bike": "mechanical_bike",
    "bike": "mechanical_bike",
    "classic-bike": "mechanical_bike",
    "regular-bike": "mechanical_bike",
    "velo-mecanique": "mechanical_bike",
}

mapped_vt = norm_vt.map(vehicle_map)
changed_vt_count = int((df["vehicle_type"].astype("string") != mapped_vt).fillna(True).sum())
unmapped_mask = mapped_vt.isna()
unmapped_count = int(unmapped_mask.sum())

if unmapped_count > 0:
    print("Unmapped vehicle_type values found:", df.loc[unmapped_mask, "vehicle_type"].unique())

df["vehicle_type"] = mapped_vt
# Assumption: drop rows with unidentifiable vehicle_type if any remain after mapping
df = df.loc[~unmapped_mask].copy()

log_step(
    "Step 3 (Rule 2): Standardize vehicle_type",
    f"{changed_vt_count} rows with non-canonical spelling/casing; {unmapped_count} unmappable",
    changed_vt_count,
    "Unified variants into {e-bike, scooter, mechanical_bike} and dropped unmappable rows",
    "Rule 2 requires exact canonical labels for fleet segmentation.",
    df,
)
print("vehicle_type value_counts() after standardization:")
print(df["vehicle_type"].value_counts(dropna=False), "\n")

# ---------------------------------------------------------------------------
# Step 4 (Rule 3): Flag outside_station without dropping or filling end_station_id
# ---------------------------------------------------------------------------
# Treat empty/whitespace-only strings as missing (NaN) while preserving NaN
df["end_station_id"] = df["end_station_id"].replace(r"^\s*$", np.nan, regex=True)
df["outside_station"] = df["end_station_id"].isna()
outside_count = int(df["outside_station"].sum())

log_step(
    "Step 4 (Rule 3): Flag dockless trips (outside_station)",
    f"{outside_count} trips with empty/null end_station_id",
    outside_count,
    "Added boolean column 'outside_station' (True when end_station_id is empty); kept all rows and left end_station_id unfilled",
    "Rule 3 specifies empty end_station_id means left outside a station and must never be dropped or filled.",
    df,
)

# ---------------------------------------------------------------------------
# Step 5: Parse start_time & check start_station_id
# Assumption: Drop rows with invalid/missing start_time since temporal analysis requires it.
# ---------------------------------------------------------------------------
parsed_start = pd.to_datetime(df["start_time"], format="%Y-%m-%d %H:%M:%S", errors="coerce")
bad_time_mask = parsed_start.isna()
bad_time_count = int(bad_time_mask.sum())
df["start_time"] = parsed_start
df = df.loc[~bad_time_mask].copy()

log_step(
    "Step 5: Parse start_time timestamps",
    f"{bad_time_count} rows with null or malformed start_time",
    bad_time_count,
    "Converted start_time to datetime64 and removed unparseable rows",
    "Valid timestamps are required to answer when the fleet is used.",
    df,
)

# ---------------------------------------------------------------------------
# Step 6: Reconcile distance_km and distance_meters
# Assumption: Keep both schema columns; backfill missing from counterpart; flag mismatches/negatives.
# ---------------------------------------------------------------------------
for col in ["distance_km", "distance_meters"]:
    df[col] = pd.to_numeric(df[col], errors="coerce")

neg_dist_mask = (df["distance_km"] < 0) | (df["distance_meters"] < 0)
neg_dist_count = int(neg_dist_mask.sum())
df = df.loc[~neg_dist_mask].copy()

km_from_m = df["distance_km"].isna() & df["distance_meters"].notna()
m_from_km = df["distance_meters"].isna() & df["distance_km"].notna()
df.loc[km_from_m, "distance_km"] = df.loc[km_from_m, "distance_meters"] / 1000.0
df.loc[m_from_km, "distance_meters"] = df.loc[m_from_km, "distance_km"] * 1000.0

unit_mismatch_mask = (
    df["distance_km"].notna()
    & df["distance_meters"].notna()
    & ~np.isclose(df["distance_meters"], df["distance_km"] * 1000.0, atol=1.0)
)
mismatch_count = int(unit_mismatch_mask.sum())

log_step(
    "Step 6: Audit distance_km vs distance_meters",
    f"{neg_dist_count} negative distances; {int(km_from_m.sum() + m_from_km.sum())} backfilled; {mismatch_count} unit mismatches (>1m diff)",
    neg_dist_count + int(km_from_m.sum() + m_from_km.sum()) + mismatch_count,
    "Dropped negative distances; cross-filled missing unit pairs; retained mismatches for review",
    "Distances cannot be negative, and unit consistency ensures reliable distance features.",
    df,
)

# ---------------------------------------------------------------------------
# Step 7: Validate battery_start_pct (0 to 100) and pricing columns
# Assumption: Out-of-range battery values -> NaN; negative prices/rates -> drop.
# ---------------------------------------------------------------------------
df["battery_start_pct"] = pd.to_numeric(df["battery_start_pct"], errors="coerce")
bad_batt_mask = df["battery_start_pct"].notna() & (
    (df["battery_start_pct"] < 0) | (df["battery_start_pct"] > 100)
)
bad_batt_count = int(bad_batt_mask.sum())
df.loc[bad_batt_mask, "battery_start_pct"] = np.nan

log_step(
    "Step 7a: Validate battery_start_pct domain [0, 100]",
    f"{bad_batt_count} rows outside [0, 100]%",
    bad_batt_count,
    "Replaced out-of-bounds battery_start_pct values with NaN (kept rows)",
    "Battery percentage must lie within 0–100; nulling avoids losing valid trip/price records.",
    df,
)

for col in ["price_eur", "applied_rate_per_min"]:
    df[col] = pd.to_numeric(df[col], errors="coerce")

bad_price_mask = (
    df["price_eur"].isna()
    | (df["price_eur"] < 0)
    | df["applied_rate_per_min"].isna()
    | (df["applied_rate_per_min"] < 0)
)
bad_price_count = int(bad_price_mask.sum())
df = df.loc[~bad_price_mask].copy()

log_step(
    "Step 7b: Validate price_eur and applied_rate_per_min",
    f"{bad_price_count} rows with null or negative price_eur / applied_rate_per_min",
    bad_price_count,
    "Removed rows with missing or negative pricing values",
    "Clean non-negative pricing fields are required to model trip price drivers.",
    df,
)

# ---------------------------------------------------------------------------
# Step 8 (Rule 4): Flag duration_minutes outliers via IQR (k = 1.5)
# ---------------------------------------------------------------------------
q1 = df["duration_minutes"].quantile(0.25)
q3 = df["duration_minutes"].quantile(0.75)
iqr = q3 - q1
lower_fence = q1 - 1.5 * iqr
upper_fence = q3 + 1.5 * iqr

df["duration_outlier"] = (df["duration_minutes"] < lower_fence) | (
    df["duration_minutes"] > upper_fence
)
outlier_count = int(df["duration_outlier"].sum())

log_step(
    "Step 8 (Rule 4): Flag duration_minutes IQR outliers (k = 1.5)",
    f"Q1={q1:.2f}, Q3={q3:.2f}, IQR={iqr:.2f}, Fences=[{lower_fence:.2f}, {upper_fence:.2f}] -> {outlier_count} outliers",
    outlier_count,
    "Added boolean column 'duration_outlier'; retained all outlier rows",
    "Rule 4 requires flagging IQR outliers on cleaned duration_minutes without deleting them.",
    df,
)

# ---------------------------------------------------------------------------
# Step 9: Final Before / After Summary & Export
# ---------------------------------------------------------------------------
final_rows = len(df)
removed_rows = initial_rows - final_rows
pct_retained = (final_rows / initial_rows * 100) if initial_rows else 0.0

print("==========================================")
print("BEFORE / AFTER ROW-COUNT SUMMARY")
print("==========================================")
print(f"Initial rows (raw)      : {initial_rows:,}")
print(f"Rows removed            : {removed_rows:,}")
print(f"Final rows (cleaned)    : {final_rows:,} ( {pct_retained:.2f}% retained)".replace("( ", "("))
print(f"Outside-station trips   : {int(df['outside_station'].sum()):,} (kept)")
print(f"Duration IQR outliers   : {int(df['duration_outlier'].sum()):,} (kept & flagged)")
print("==========================================")

df.to_csv("trips_lyon_cleaned.csv", index=False)
```

Printed result:

```text
Loaded trips_lyon.csv | Initial row count: 7,000

=== Step 1: Duplicate trip_id check ===
  Found         : 0 duplicate trip_id records detected
  Affected rows : 0
  Action        : No rows dropped
  Why           : Each row must represent one unique trip rental (trip_id is unique).
  Row count now : 7,000

=== Step 2 (Rule 1): Valid duration filter ===
  Found         : 205 rows with null, <= 0, or >= 1440 duration_minutes
  Affected rows : 205
  Action        : Removed rows outside (0, 1440) minutes
  Why           : Rule 1 defines valid trips strictly as 0 < duration_minutes < 1440 (< 24h).
  Row count now : 6,795

=== Step 3 (Rule 2): Standardize vehicle_type ===
  Found         : 3367 rows with non-canonical spelling/casing; 0 unmappable
  Affected rows : 3,367
  Action        : Unified variants into {e-bike, scooter, mechanical_bike} and dropped unmappable rows
  Why           : Rule 2 requires exact canonical labels for fleet segmentation.
  Row count now : 6,795

vehicle_type value_counts() after standardization:
vehicle_type
e-bike             3378
scooter            2271
mechanical_bike    1146
Name: count, dtype: int64 

=== Step 4 (Rule 3): Flag dockless trips (outside_station) ===
  Found         : 397 trips with empty/null end_station_id
  Affected rows : 397
  Action        : Added boolean column 'outside_station' (True when end_station_id is empty); kept all rows and left end_station_id unfilled
  Why           : Rule 3 specifies empty end_station_id means left outside a station and must never be dropped or filled.
  Row count now : 6,795

=== Step 5: Parse start_time timestamps ===
  Found         : 0 rows with null or malformed start_time
  Affected rows : 0
  Action        : Converted start_time to datetime64 and removed unparseable rows
  Why           : Valid timestamps are required to answer when the fleet is used.
  Row count now : 6,795

=== Step 6: Audit distance_km vs distance_meters ===
  Found         : 0 negative distances; 0 backfilled; 0 unit mismatches (>1m diff)
  Affected rows : 0
  Action        : Dropped negative distances; cross-filled missing unit pairs; retained mismatches for review
  Why           : Distances cannot be negative, and unit consistency ensures reliable distance features.
  Row count now : 6,795

=== Step 7a: Validate battery_start_pct domain [0, 100] ===
  Found         : 0 rows outside [0, 100]%
  Affected rows : 0
  Action        : Replaced out-of-bounds battery_start_pct values with NaN (kept rows)
  Why           : Battery percentage must lie within 0–100; nulling avoids losing valid trip/price records.
  Row count now : 6,795

=== Step 7b: Validate price_eur and applied_rate_per_min ===
  Found         : 0 rows with null or negative price_eur / applied_rate_per_min
  Affected rows : 0
  Action        : Removed rows with missing or negative pricing values
  Why           : Clean non-negative pricing fields are required to model trip price drivers.
  Row count now : 6,795

=== Step 8 (Rule 4): Flag duration_minutes IQR outliers (k = 1.5) ===
  Found         : Q1=8.60, Q3=19.60, IQR=11.00, Fences=[-7.90, 36.10] -> 160 outliers
  Affected rows : 160
  Action        : Added boolean column 'duration_outlier'; retained all outlier rows
  Why           : Rule 4 requires flagging IQR outliers on cleaned duration_minutes without deleting them.
  Row count now : 6,795

==========================================
BEFORE / AFTER ROW-COUNT SUMMARY
==========================================
Initial rows (raw)      : 7,000
Rows removed            : 205
Final rows (cleaned)    : 6,795 (97.07% retained)
Outside-station trips   : 397 (kept)
Duration IQR outliers   : 160 (kept & flagged)
==========================================
```

Audit: \[TODO: replace this with the audit of the real printed result. Until then, this is a static review of the code only, nothing has been run.]

* Right (code): Rule 1 is applied as 0 < duration\_minutes < 1440, which should remove the 205 invalid rows and leave 6,795.
* Right (code): the vehicle\_type mapping contains 'trottinette' and 'electric-bike', and underscores are turned into hyphens first, so all 6 spellings should map to 3 labels. This is the point where ChatGPT's Round 2 code fails. \[TODO: confirm with value\_counts() on the real output]
* Right (code): Rule 3 adds outside\_station and leaves end\_station\_id unfilled; Rule 4 flags IQR (k = 1.5) outliers on the cleaned durations and keeps them.
* Risk (code): the code removes rows my rules never asked it to remove: duplicate trip\_id, unparseable start\_time, negative distances, null or negative price/rate, and any unmappable vehicle\_type. Each deletion is logged, and ChatGPT's run found 0 such rows, so probably nothing is lost on this file. Battery values outside 0-100 are set to NaN instead of dropped. \[TODO: check the printed counts]
* Not checkable until the code is run: final row count (baseline 6,795), outside-station count, IQR count (baseline 160).

Correction: \[TODO: only if the run shows a mismatch: your code, and why]