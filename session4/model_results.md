# Session 4: Model results

**Case:** UrbanMove Lyon, 6,795 cleaned trips. Target: `price_eur`.
**Models:** A = ChatGPT (GPT 6.1 Sol), B = Gemini (Gemini 4 Hydrogen).
**Features:** duration_minutes, distance_km, vehicle_type (+ battery_start_pct for PCA). Split 80/20, random_state=42.

---

## 1. Comparison grid: framing v2 (Run 1 and Run 2)

| Result | Reference | ChatGPT | Gemini | Agree? |
|---|---|---|---|---|
| Columns used (PCA) | duration, km, battery | same | same | Yes, because my framing listed them |
| PCA: components kept | 2 (99.1%) | 2 (99.1%) | 2 (99.1%) | Yes |
| PCA: PC1 / PC2 | duration + distance / battery | same | same | Yes |
| Missing vehicle_type (1,118) | not handled at first; then fillna("scooter") | "Unknown" category | "Unknown" category | No: different methods |
| Regression: test R² | 0.9984 (1.0 after fix) | 1.0 | 1.0 | Yes after fix |
| Regression: test MAE (€) | 0.048 (≈ 0 after fix) | ≈ 0 | 0.0000 | Yes after fix |
| Regression: intercept | 1.0 (after fix) | 1.1 | 1.2 | **No: different base category** |
| Gradient boosting: test R² | 0.9982 | 0.99983 | 0.9998 | Yes |
| Gradient boosting: test MAE (€) | 0.055 | 0.0164 | 0.0169 | No: my missing values |
| Top feature | duration 0.998 | duration 0.996 | duration 0.997 | Yes |

## 2. Comparison grid: framing v3 (Run 3)

| Result | ChatGPT | Gemini | Agree? |
|---|---|---|---|
| Regression: train / test R² | 1.0 / 1.0 | 1.0 / 1.0 | Yes |
| Regression: 5-fold CV R² | 1.0 (all 5) | 1.0 (all 5) | Yes |
| Ridge: test MAE (€) | 0.00015 | 0.00015 | Yes |
| GB: train / test R² | 0.99988 / 0.99983 | 0.99988 / 0.99983 | Yes |
| GB: 5-fold CV R² mean | 0.99946 (lowest fold 0.99814) | 0.99969 | Close |
| Intercept (e-bike base) | 1.0 | 1.0 | Yes |
| Start fee: scooter / unknown | 1.20 / 1.20 | 1.20 / 1.20 | Yes |
| Ridge vs linear coefficients | differ by < 0.0004 | differ by < 0.0004 | Yes |
| Mechanical bikes in the model | 1,146 | **0 (became NaN)** | **No** |

---

## 3. Suspicions and tests

**1. Missing vehicle_type**
- Number: 1,118 trips (16%) have no vehicle_type; my first reference MAE was 0.048 € instead of ≈ 0.
- Suspicion: `get_dummies` silently treats NaN as the base category (e-bike).
- Test: `pd.crosstab(df_clean["vehicle_type"].fillna("MISSING"), (df_clean["price_eur"] - 0.2*df_clean["duration_minutes"]).round(2))`
- Seen: e-bike and mechanical_bike fee 1.0; scooter 1.2; MISSING 1.2 for all 1,118.
- Outcome: **confirmed.** The missing trips were charged as scooters.

**2. Three different intercepts (1.0 / 1.1 / 1.2)**
- Suspicion: same predictions, different base category.
- Test: intercept + coefficient for each vehicle type.
- Seen: bike = 1.0 and scooter = 1.2 in all three models. Gemini used "Unknown" as base, because "U" sorts before "e".
- Outcome: **confirmed.** The coefficients depend on the encoding; the pricing rule is the same.

**3. ChatGPT weights of ±0.1 (v2)**
- Suspicion: all 4 categories kept together with an intercept (dummy trap), so the weights are not unique.
- Test: count the one-hot columns in ChatGPT's model.
- Seen: 4 columns (Unknown, e-bike, mechanical_bike, scooter).
- Outcome: **confirmed.** Predictions are fine; the weights cannot be read as euros.

**4. Gemini: "vehicle type changes the rate per minute"**
- Suspicion: its own output shows one duration coefficient for all vehicles.
- Test: Gemini's printed coefficients.
- Seen: duration 0.2; vehicle type only shifts the intercept.
- Outcome: **confirmed wrong.** Vehicle type changes the start fee, not the per-minute rate.

**5. Gemini "Results" without running code (v2)**
- Suspicion: the Results section describes expected values, not printed numbers.
- Test: read its note: "no live Python runtime ... the expected outputs are".
- Outcome: **confirmed.** It wrote guesses in place of results, against my framing.

**6. R² = 1.0**
- Suspicion: the price is a fixed formula, so the model only copies it.
- Test: the crosstab in suspicion 1.
- Seen: every trip = start fee + 0.20 € × minutes, no exceptions.
- Outcome: **confirmed.** A perfect score shows the rule, not predictive power.

**7. Category name mismatch (v3)**
- Number: Gemini GB importance for "mechanical bike" = 0.000000; ChatGPT's first v3 code crashed.
- Suspicion: my context note wrote "mechanical bike"; the data says "mechanical_bike".
- Test: `print(df_model["vehicle_type"].value_counts(dropna=False))`
- Seen: e-bike 3378, scooter 1153, **NaN 1146**, unknown 1118, mechanical bike 0.
- Outcome: **confirmed.** ChatGPT stopped with an AssertionError (a safe failure); Gemini ran and silently merged 1,146 mechanical bikes into e-bike. The scores stayed at R² 1.0 because both have the same 1.00 € fee. Importance 0 alone does not prove it: in ChatGPT's corrected run mechanical_bike is also 0.000000, because it adds no information. The row count is the real test.

---

## 4. Framing v2 → v3

| Change | Why (audit) | Before (v2) | After (v3) |
|---|---|---|---|
| Role and "no clean task" rule cover modelling | v2 only allowed cleaning tasks | ignored by both models | consistent |
| Keep "unknown", report count, compare its fee | Suspicion 1 | not reported | both report unknown = 1.20 € |
| e-bike as explicit base category | Suspicions 2, 3 | intercept 1.1 / 1.2 | 1.0 / 1.0 |
| No per-minute claim without a test | Suspicion 4 | wrong claim (Gemini) | not repeated |
| No invented results | Suspicion 5 | Gemini guessed | real output only |
| 5-fold CV and Ridge | slides 23–24 | single split | CV stable, Ridge ≈ linear |

**Scores did not improve** (already R² 1.0). **Interpretability did:** both models now give the same coefficients as the reference. **But a new error appeared** (suspicion 7): a single word in my framing created a silent error that the perfect score hid.

---

## 5. Final recommendation (three sentences)

1. The price of a trip follows a fixed rule: a start fee (1.00 € for bikes, 1.20 € for scooters) plus 0.20 € per minute; distance has no effect.
2. The near-perfect scores (R² 1.0, MAE ≈ 0 €, stable over 5 folds) only show that the model reproduces this rule, so it should not be used to set or forecast prices.
3. vehicle_type is missing for 1,118 trips that were all charged the scooter fee; the operator should check why these trips were not labelled.
