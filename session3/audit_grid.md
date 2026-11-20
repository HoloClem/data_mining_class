# audit_grid.md - Session 3

Legend: ✓ matches baseline, ~ partly matches, ✗ does not match. Items marked (verify) are inferred from the printed output only; confirm them by running the model's code.

## Round 1 (Run 1, framing version [TODO: 1 or 2])

| Check | My baseline | ChatGPT6.1_Sol | Gemini3.5_Flash |
|---|---|---|---|
| Raw rows loaded | 7,000 | 7,000 ✓ | 7,000 ✓ |
| Missing end_station_id | 407, kept and flagged | 407 found, not dropped ✓ (flag column: verify) | 407 found and dropped ✗ |
| Vehicle labels after cleaning | 3 | Spelling problem not reported, probably 6 ✗ (verify) | Only casing fixed (2,257 rows), probably 5 ✗ (verify) |
| Negative durations | 109 removed | 109 detected, not removed ~ | 106 removed ~ (verify) |
| Trips of 24 hours or more | 96 removed | Not flagged as invalid ✗ | 93 removed as "outliers" ~ (verify) |
| Rows after cleaning | 6,795 | 7,000 ✗ | 6,394 ✗ |
| IQR flagged rides (k = 1.5) | 160 | 233, computed on 7,000 rows ✗ | Not reported; used k = 3 by vehicle_type ✗ |
| **Tally** | | 2 ✓, 1 ~, 4 ✗ | 1 ✓, 2 ~, 4 ✗ |

### Mismatches and responsible code

| Model | Mismatch | Line of code responsible | Why the model probably chose it |
|---|---|---|---|
| ChatGPT | Vehicle labels not unified | [TODO] | No rule about valid vehicle types in the framing (verify) |
| ChatGPT | Final count 7,000, nothing removed | [TODO] | [TODO] |
| ChatGPT | IQR 233 vs 160 | [TODO] | Computed before removing invalid durations [TODO: verify] |
| Gemini | 407 end stations dropped | [TODO] | Treated an empty station as an invalid format |
| Gemini | Only 5 labels | [TODO] | Lowercase/strip without mapping electric_bike and trottinette |
| Gemini | 6,394 final rows | [TODO] | 3 x IQR removal by vehicle_type on duration, distance and price |

Which model was closer: [TODO, my suggestion: ChatGPT on the 407 and the 109 negatives]
Which model was safer: [TODO, my suggestion: ChatGPT, since Gemini deletes data]

## Round 2 (Run 2, framing version 2)

| Check | My baseline | ChatGPT | Gemini |
|---|---|---|---|
| Raw rows loaded | 7,000 | [TODO] | [TODO] |
| Missing end_station_id | 407, kept and flagged | [TODO] | [TODO] |
| Vehicle labels after cleaning | 3 | [TODO] | [TODO] |
| Negative durations | 109 removed | [TODO] | [TODO] |
| Trips of 24 hours or more | 96 removed | [TODO] | [TODO] |
| Rows after cleaning | 6,795 | [TODO] | [TODO] |
| IQR flagged rides (k = 1.5) | 160 | [TODO] | [TODO] |