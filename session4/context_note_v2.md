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

## Modelling

### 1. Target and features
- I want to predict price_eur, the price paid for one trip, in euros.
- The data is df_clean, my cleaned table from Session 3 (6,795 trips). It can be loaded with pd.read_csv("../data/trips_lyon_cleaned.csv").
- The features are duration_minutes (length of the trip in minutes), distance_km (distance in kilometres) and vehicle_type (e-bike, mechanical bike or scooter). All three are known once the trip is over, so they can be used.
- vehicle_type is missing for 1,118 trips. This needs to be handled and reported.
- For the PCA, battery_start_pct (battery level at the start, in percent) can also be used.

### 2. Columns to leave out
- applied_rate_per_min, because it is part of how the price is calculated and would give away the answer.
- distance_meters, because it is the same information as distance_km.
- trip_id, user_id and the station ids, because they are identifiers and not measurements.

### 3. How success is measured
- The model learns on 80% of the trips and is tested on the other 20%, with random_state=42.
- I want R² and MAE (in euros) on the test set, with the same scores on the training set next to them.
- A result is trustworthy if the train and test scores are close and the coefficients make sense, for example the number of euros one extra minute adds.

### 4. What I found in my own run
- With PCA on duration, distance and battery, two components keep about 99% of the information.
- The price seems to follow a fixed rule based on duration and vehicle type. A score close to perfect is therefore expected, and it should be explained rather than presented as a success.