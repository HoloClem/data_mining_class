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
