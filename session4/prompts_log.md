# Session 4 — Prompts log

## 0. Reference (my own code)
### PCA
Columns: duration_minutes, distance_km, battery_start_pct (standardized)
Shares: [0.658, 0.333, 0.009] — cumulative [0.658, 0.991, 1.0]
Keep: 2 components (99.1%). PC1 = duration + distance (0.71 / 0.71), PC2 = battery (1.00), PC3 = duration − distance
### Regression
First run: R² 0.9984, MAE 0.048, intercept 1.05, mech_bike -0.051 (slide: 1.0 / ≈0 / 1.0 / 0)
After fillna("scooter"): R² 1.0, MAE 8.7e-16, duration 0.2, distance 0.0, mech 0.0, scooter 0.2, intercept 1.0
### Gradient boosting
R² 0.9982, MAE 0.055, importance duration 0.998

## 1. Check of my reference
Suspicion: my MAE differs from the slide although PCA matched
Test: pd.crosstab(vehicle_type.fillna("MISSING"), price - 0.2*minutes)
Seen: e-bike/mech fee 1.0, scooter 1.2, MISSING (1,118 rows) all 1.2
Outcome: confirmed — get_dummies treated NaN as e-bike

## 2. Framing v2
Files: system_prompt_v2.md, context_note_v2.md (Session 3 version + Modelling section)
Models: Model A = ChatGPT (version __), Model B = Gemini (version __)

## 3. Run 1 — PCA
Task prompt: Run a PCA on the numeric columns of the cleaned trips. Tell me how many components to keep, and what each one represents.
### Model A — ChatGPT
Output: (paste unedited)
Ran unedited: yes
### Model B — Gemini
Output: (paste unedited)
Ran unedited: yes

## 4. Run 2 — Regression and gradient boosting
Task prompt: Predict price_eur from the cleaned trips, once with a linear regression and once with gradient boosting. Report R² and MAE on a test set, the coefficients, and the feature importances.
### Model A — ChatGPT
Output: (paste unedited)
### Model B — Gemini
Output: (paste unedited — including the "expected results" it wrote without running anything)

## 5. Recommendation audit
Prompt: In three sentences, what should the operator do with these results?
### ChatGPT
Answer: (paste)
1. FIX — no number; operator already knows the rule; distance has no effect
2. CROSS OUT — a step for the analyst, not a decision for the operator
3. KEEP with fix — remove the "if": R² is 1.0 because the model reproduces the rule
### Gemini
Answer: (paste)
1. FIX — "logging issue" is assumed; missing trips were charged the scooter fee
2. FIX — per-minute rate is 0.20 € for every vehicle; only the start fee differs
3. CROSS OUT — 99% was copied from my framing and misread; no data links battery to revenue

## 6. Framing v3
(one entry per change)
Version: 3
What I changed: __
Why: __
Score before (v2): train __ / test __
Score after (v3): train __ / test __
Did it get better? __

## 7. Run 3 — v3 rerun (ChatGPT and Gemini)
(task prompt + outputs, unedited)

## 8. Mini-challenge — Paris
(prompts, outputs, reference, 2 suspicions with test code)