# DXB Sky Intelligence: Data Dictionary

Power BI project on flight delays at Dubai International Airport (DXB). Flight and weather data is collected automatically every 2 hours, cleaned in Power Query, and modeled as a star schema.

## 1. Data sources

| Source | Used for | Notes |
| --- | --- | --- |
| AirLabs Schedules API (free plan) | Departures and arrivals at DXB: scheduled, estimated and actual times, status, terminal, gate, airline, codeshare fields | Returns up to 100 departures and 100 arrivals per call, so the data is a collected sample, not full airport traffic |
| Open-Meteo | Hourly weather at DXB | Free, no key |
| OpenFlights (airlines.dat, airports.dat) | Airline names and countries, airport names, cities, countries, latitude and longitude | Downloaded once |

Collection: two Python scripts run on GitHub Actions. A cron-job.org timer starts the workflow every 2 hours, and GitHub's own hourly schedule is a backup that skips when the last file is under 100 minutes old. One flights file is saved per run and one weather file per day, in a private GitHub repository.

## 2. Model overview (star schema)

| Table | Type | Grain |
| --- | --- | --- |
| flights | Fact | One row per real flight (direction + flight number + scheduled departure) |
| dim\_airline | Dimension | One row per active airline code |
| dim\_airport | Dimension | One row per airport code |
| dim\_date | Dimension | One row per calendar day (1 Oct to 31 Dec 2026) |
| weather\_hourly | Dimension | One row per hour (Dubai time) |
| \_Measures | Helper | Holds all DAX measures |
| Metric | Field parameter | Lets a slicer switch the measure shown in a chart |

### Relationships (all many-to-one, single direction)

| From (flights) | To | Meaning |
| --- | --- | --- |
| flight\_date | dim\_date\[Date\] | Day of the flight |
| airline\_iata | dim\_airline\[airline\_iata\] | Airline of the flight |
| other\_airport | dim\_airport\[airport\_iata\] | The airport at the other end of the route (Dubai is always one end) |
| weather\_hour | weather\_hourly\[time\] | Weather of the flight's scheduled hour |

## 3. flights (fact table)

### Columns from the source

| Column | Description |
| --- | --- |
| collected\_at\_utc | When the record was collected (UTC) |
| direction | dep = departure from DXB, arr = arrival at DXB |
| airline\_iata, flight\_iata, flight\_number | Airline code, full flight code, number |
| dep\_iata, arr\_iata | Departure and arrival airport codes |
| dep\_terminal, dep\_gate, arr\_terminal, arr\_gate | Terminal and gate |
| dep\_time, arr\_time | Scheduled times, Dubai time |
| dep\_time\_utc, arr\_time\_utc | Scheduled times, UTC |
| dep\_estimated, dep\_actual, arr\_estimated, arr\_actual | Estimated and actual times (empty until they happen) |
| status | scheduled, active, landed or cancelled |
| duration | Scheduled flight duration in minutes |
| delayed, dep\_delayed, arr\_delayed | Delay minutes reported by AirLabs. Not used, because AirLabs fills them only when a flight is late |
| cs\_airline\_iata, cs\_flight\_iata, cs\_flight\_number | Codeshare fields. Filled only on the partner copy of a flight |

Removed: arr\_baggage (mixed numbers and text caused load errors, and it is not used).

### Columns added in Power Query

| Column | Description |
| --- | --- |
| dep\_delay\_min | Actual departure minus scheduled departure, in minutes. Negative = left early. Empty if the flight has not departed |
| arr\_delay\_min | Actual arrival minus scheduled arrival, in minutes. Negative = arrived early |
| airline\_name, airline\_country | Joined from dim\_airline |
| other\_airport | Arrival airport for departures, departure airport for arrivals |
| airport\_name, city, country, latitude, longitude | Joined from dim\_airport for the other airport |
| weather\_hour | Scheduled time rounded down to the hour, used to join weather |
| temperature\_2m (C), relative\_humidity\_2m (%), precipitation (mm), visibility (m), wind\_speed\_10m (km/h), wind\_gusts\_10m (km/h), cloud\_cover (%), weather\_code | Weather of that hour, from weather\_hourly |

### Calculated columns (DAX)

| Column | Description |
| --- | --- |
| airline\_label | Airline name, or the airline code when the name is missing |
| flight\_date | Calendar date of weather\_hour |
| flight\_hour | Hour of day (0 to 23), Dubai time |
| wind\_band | Calm (under 15 km/h), Breezy (15 to 25), Windy (25+), based on gusts |
| temp\_band | Mild (under 30 C), Warm (30 to 34), Hot (34+) |
| visibility\_band | Poor (under 5 km), Hazy (5 to 10 km), Clear (10+ km) |
| dep\_status | On time (15 min or less), Delayed (more than 15 min) for departed flights |

## 4. Dimensions

### dim\_airline

| Column | Description |
| --- | --- |
| airline\_name | Airline name |
| airline\_iata | Two-character airline code (unique, active airlines only) |
| airline\_country | Country of the airline |
| is\_active | Y for all rows kept |

### dim\_airport

| Column | Description |
| --- | --- |
| airport\_name, city, country | Airport details |
| airport\_iata | Three-letter airport code (unique) |
| latitude, longitude | Map position |

### dim\_date

| Column | Description |
| --- | --- |
| Date | Calendar day |
| Year, MonthNo, Month, Day | Date parts |
| WeekdayNo, Weekday | Weekday number (Monday = 1) and short name |

### weather\_hourly

One row per hour with time, temperature\_2m, relative\_humidity\_2m, precipitation, visibility, wind\_speed\_10m, wind\_gusts\_10m, cloud\_cover and weather\_code.

## 5. Measures (\_Measures table)

| Measure | Definition |
| --- | --- |
| Total Flights | Number of flights |
| Departures, Arrivals | Flights by direction (scheduled, including flights not yet flown) |
| Departed Flights | Departures that have an actual time, so a delay value exists |
| Avg Dep Delay (min), Avg Arr Delay (min) | Average delay. Sensitive to extreme flights |
| Median Dep Delay (min) | Median departure delay. Preferred over the average |
| On-time Dep %, On-time Arr % | Share of flights within 15 minutes of schedule, among flights that have a delay value |
| Cancelled % | Cancelled flights divided by all flights |

## 6. Cleaning rules

1. All CSV files in the flights folder are combined into one table.
2. Repeated records of the same flight (direction, flight\_iata, scheduled departure) are reduced to the latest one, so the final status is kept.
3. Codeshare copies are removed: only rows where cs\_flight\_iata is empty (the operating airline) are kept.
4. Delays are calculated from the times, because AirLabs' own delay fields are often empty.
5. Airlines are matched on active airlines with unique codes. Airports are matched on unique codes. About 2% of airports are missing from the free list.

## 7. Security

Row-level security roles on dim\_airline: Emirates\_Manager (\[airline\_iata\] = "EK") and FlyDubai\_Manager (\[airline\_iata\] = "FZ"). Tested with View as.

## 8. Known limits

- The data is a collected sample (about 100 departures and 100 arrivals per call), not every flight at DXB. Counts per hour reflect collection, not true traffic.
- History is the days collected so far, so findings are early.
- No fog or haze has appeared yet. Weather findings need more days, including the December to February fog season.
- Delay averages can be pulled by a few rescheduled flights, so the report uses medians and minimum-flight filters.
