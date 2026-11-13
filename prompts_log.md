# prompts_log.md - Session 3, Run 1

Student: [your name]
Date: 2026-11-13
File: trips_lyon.csv (7,000 rows, 12 columns)
Setup: same framing, same task prompt, same file, two models.

`[TODO]` marks information only you have (exact text sent, code produced, checks you ran). Fill these in before handing in. Log what you really sent, even if it was weak.

---

## Framing used (identical for both models)

### Layer 1 - System prompt
[TODO: paste the exact text you sent]

### Layer 2 - Context note
[TODO: paste the exact text you sent]

Framing version: [TODO: 1 = columns and units only / 2 = with the rules discovered in Session 2]

### Layer 3 - Task prompt (Session 3 slide text; confirm you sent it unchanged)

```
Audit trips_lyon.csv, list the problems you find, then write pandas code
that cleans the file. Handle outliers and report the final row count.
```

---

## Entry 1 - Model A: ChatGPT [TODO: exact name and version; file name says "ChatGPT6.1_Sol"]

**Prompt:** system prompt + context note + task prompt above.

**Output - code:** [TODO: paste the code ChatGPT produced, unedited]

**Output - printed result** (from ChatGPT6.1_Sol_clean_result.txt, unedited):

```
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

**Audit - what was right**
- Raw rows: 7,000, same as my baseline.
- Found the 407 missing end_station_id and did not drop them (final count stays 7,000), consistent with my keep-and-flag decision. [TODO: check the code creates an outside_station flag column]
- 109 out-of-range durations = my 109 negative durations.
- 0 duplicates and 0 battery_start_pct problems, same as my Session 2 findings.

**Audit - what was wrong or missing**
- vehicle_type: no spelling problem reported, although the file has 6 spellings for 3 vehicles. Labels are probably still 6 after "cleaning". [TODO: confirm with value_counts() on its output]
- The 96 trips of 24 hours or more are not counted as invalid (range issues = 109, my rule gives 205).
- Final row count is 7,000 vs my 6,795: the result reads as an audit, the negative durations are detected but not removed. [TODO: confirm in the code]
- IQR: 233 duration outliers "on retained rows" (here all 7,000 rows) vs my 160 on the cleaned durations. Different row set, so not comparable. [TODO: find the line that sets the row set and the k value]

**Responsible lines:** [TODO]

**Correction:** [TODO: paste your corrected code. Baseline from Session 2 for reference:]

```python
d = df["duration_minutes"]
valid = (d > 0) & (d < 1440)
df_clean = df[valid].copy()          # expected: 6,795 rows
df_clean["outside_station"] = df_clean["end_station_id"].isna()
v = df_clean["vehicle_type"].str.lower().replace(
    {"electric_bike": "e-bike", "trottinette": "scooter"})
df_clean["vehicle_type"] = v          # expected: 3 labels
```

---

## Entry 2 - Model B: Gemini [TODO: exact name and version; file name says "Gemini3.5flash"]

**Prompt:** identical to Entry 1.

**Output - code:** [TODO: paste the code Gemini produced, unedited]

**Output - printed result** (from Gemini3.5flash_clean_results.txt, unedited):

```
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

**Audit - what was right**
- Raw rows: 7,000, same as my baseline.
- The audit counts 205 duration problems (<= 0 or > 1,440 min), which equals my 109 negative + 96 long trips.
- Removed non-positive durations (the 106 rows).
- 0 duplicates and 0 battery_start_pct problems, same as my Session 2 findings.

**Audit - what was wrong or missing**
- It deleted the 407 trips with an empty end_station_id (labelled "invalid station format"). These are free-floating rides, a business signal; my decision was keep and flag. This is the `dropna()`-type error from the slides.
- vehicle_type: 2,257 rows flagged = 1,101 (E-BIKE) + 1,156 (TROTTINETTE) from my Session 2 table, so only the casing was detected. electric_bike (1,208 rows) and trottinette are probably still separate labels, i.e. 5 labels instead of 3. [TODO: confirm with value_counts() on its output]
- Row counts do not match my baseline: 106 instead of 109 negatives, 93 instead of 96 long trips. Likely cause: 3 + 3 rows were already deleted in the 407-row step. [TODO: verify; hypothesis only]
- The 93 "extreme outliers" use a 3 x IQR rule by vehicle_type on duration, distance and price. Removing real but extreme rides goes against my rule "flagged does not mean deleted", and it was applied to columns I never asked about.
- Final row count 6,394 vs my 6,795 (401 fewer). It never reports the k = 1.5 IQR count (my 160), since it used k = 3 by vehicle_type.

**Responsible lines:** [TODO]

**Correction:** [TODO: paste your corrected code. Same baseline code as Entry 1.]

---

## Comparison and decision

| | ChatGPT | Gemini |
|---|---|---|
| Closer to my baseline | Yes on the 407 rows and on detecting the 109 negatives, but it removes nothing (7,000 rows) | Applies removal rules, but ends at 6,394 rows |
| Safer | Yes: no data destroyed, but the file is not actually cleaned | No: deletes the 407 free-floating rides and extra extreme rides without being asked |
| Missed | vehicle spellings, 96 trips >= 24 h | electric_bike / trottinette spellings, keep-and-flag of the 407 |

Suggested verdict (edit to your own view): neither model can be used as is. ChatGPT is safer but incomplete; Gemini is more complete but damages the data. Both miss the vehicle_type unification.

## Rules to add to framing version 2

- A trip lasts more than 0 and less than 1,440 minutes (remove the rest).
- vehicle_type has exactly 3 valid labels: e-bike, scooter, mechanical_bike; map electric_bike -> e-bike and trottinette -> scooter.
- An empty end_station_id means the vehicle was left outside a station: keep the row and add outside_station. Never drop it.
- Never delete outliers; compute IQR (k = 1.5) on the cleaned durations and only report/flag them.
- Print vehicle_type.value_counts() and the row count after every cleaning step.

## Next entries

[TODO: Entry 3 and 4 = Run 2 with framing version 2 on both models; Entry 5 and 6 = chart audit; Entry 7 and 8 = Paris mini-challenge]