# Weather Streaming Pipeline 
A small real-time data pipeline that pulls weather data for Pakistani cities, streams it through Kafka, and lands it in PostgreSQL, ready for dashboards in Tableau.

I built this while learning Kafka and stream processing. The goal was simple: take a messy, always-changing data source and turn it into something clean and useful, end to end.


## How it works

```
Open-Meteo API
   │   live polling + historical backfill
   ▼
Kafka topic: weather-raw
   │
   ▼
Stream processor (consumer.py)
   ├── saves every reading        → staging.weather_raw
   ├── 5-minute window summaries  → staging.weather_aggregates  (+ weather-aggregates topic)
   └── threshold alerts           → staging.weather_alerts      (+ weather-alerts topic)
   │
   ▼
SQL views (insights.sql)  →  Tableau
```

The data comes from [Open-Meteo](https://open-meteo.com/), which is free and needs no API key. The producer can run in two modes: **live** (what's the weather right now?) and **backfill** (give me the last few weeks of hourly history).

## What's in the repo
| File | What it's for |
|---|---|
| `common.py` | Shared settings, message schemas, and Kafka/Postgres helpers |
| `producer.py` | Fetches weather data from Open-Meteo and publishes it to Kafka |
| `consumer.py` | Reads from Kafka, builds window summaries, raises alerts, writes to Postgres |
| `schema.sql` | Creates the staging tables |
| `insights.sql` | Creates the views Tableau connects to |
| `docker-compose.yml` | Runs Kafka and PostgreSQL locally |
| `requirements.txt` | Python dependencies |

## Getting started

You'll need Docker and Python 3.10 or newer.
**1. Start Kafka and Postgres**
```bash
docker compose up -d
```
The tables and views are created automatically the first time Postgres starts.

**2. Install the Python packages**
```bash
pip install -r requirements.txt
```

**3. Start the consumer** (leave this terminal open)
```bash
python consumer.py
```

**4. Load some history** (in a second terminal)
```bash
python producer.py backfill --days 30
```

**5. Switch to live data**
```bash
python producer.py live
```

By default, live mode checks for new data every 15 minutes. You can change that:
```bash
python producer.py live --interval 600 --batch-size 5
```

## Tweaking it
Everything you'd want to change lives in `common.py`:

- **Cities**: edit the `CITIES` dictionary (name → latitude, longitude)
- **Alert rules**: edit `THRESHOLDS`. Right now it flags heat above 40°C, freezing temperatures, winds above 50 km/h, and rain above 10 mm
- **Window size**: `WINDOW_SECONDS` (5 minutes by default)

## Views for Tableau
| View | What you get |
|---|---|
| `v_tableau_hourly_master` | Hourly trends for each city |
| `v_tableau_daily_climate` | Daily highs, lows, rainfall, and wind |
| `v_tableau_alert_analytics` | How often each alert type fires |
| `v_tableau_city_benchmarks` | How cities compare, with hottest and wettest rankings |
| `v_tableau_pipeline_health` | How fresh the data is and how much has arrived |

They live in the `insights` schema. Point Tableau at the `weather` database on `localhost:5432` (user and password are `postgres` by default).

- **Live data updates about every 15 minutes**, so a 5-minute window usually holds just one reading. The consumer closes quiet windows after 30 seconds so nothing gets stuck waiting.
- **Backfill skips the last couple of days** because Open-Meteo's archive takes a little while to fill in. If you see empty values near the end, raise `--end-lag`.
- **The API is free but fair-use.** The producer retries with a backoff if it gets rate-limited, and the defaults are gentle on purpose.
- **Writes are safe to repeat.** If the consumer restarts, it may see some messages twice, but upserts mean you won't end up with duplicate rows.

## What I'd like to add next
- A star-schema warehouse with an hourly Airflow job
- A live Tableau dashboard on top of it
- Tests for the window and alert logic

## Tech used
Python · Apache Kafka · PostgreSQL · Docker Compose · Pydantic · Open-Meteo API · Tableau

---

Questions or ideas? Feel free to open an issue. 💬
