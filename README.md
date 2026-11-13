# Real-Time PM2.5 Stream Intelligence (Astana)

Endterm project for **Real-Time Data Analysis**: real-time monitoring, time-series analysis and stream intelligence.
Author: Yerlan Utepbergenov, BDA-2401.

The project simulates an air-quality sensor network in four Astana districts (Yesil, Almaty, Saryarka, Baikonur) and processes the 1-minute PM2.5 readings as a live stream. Each event goes through anomaly detection and a 5-minute-ahead forecast, and a dashboard updates while the data arrives.

📄 **Technical report:** [`report/report.pdf`](report/report.pdf)

## Architecture

```
 data generator ──► producer ──► partitioned queues ──► worker pool (N threads) ──► results queue ──► live dashboard
 (4 districts,       (key =       (crc32(sensor_id)     per event: anomaly          (sink)            + metrics
  1-min readings)    sensor_id)    % N partitions)      detection + online ML)
```

- **Producer:** replays events in time order at 1,500 events/s and routes each one by `crc32(sensor_id) % N`, so every sensor always goes to the same partition.
- **Workers:** one consumer per partition. Each keeps per-sensor state (a rolling window and pending forecasts), which preserves per-sensor order without locks. This is the same idea as Kafka partitions with a consumer group.
- **Sink:** collects results, samples the backlog and redraws the dashboard.

## What it does

| Requirement | Implementation |
|---|---|
| Continuously updated visualization | Live 4-panel dashboard: PM2.5 with forecast and anomalies, anomalies per district, throughput and backlog, KPIs |
| Time-series analysis | Resampling (15 min, 1 h), rolling windows (1 h, 24 h), trend + daily seasonality decomposition |
| Anomaly detection | Online rolling **MAD** (robust) vs. rolling **z-score**, against the previous 30 minutes only |
| Real-time ML | Online `SGDRegressor` (Huber loss, `partial_fit`) forecasting PM2.5 at t+5 min, test-then-train |
| No leakage | Features use only values available at time t; labels are hidden from the stream |
| Distributed processing | Producer, key-partitioned streams, multiple consumer workers, scaling run with 1/2/4 workers |
| Operational metrics | Throughput, end-to-end latency (p50/p95/p99), backlog, events per worker |

## Key results

| Component | Result |
|---|---|
| Anomaly detection, rolling MAD (k=4) | precision 0.79, **recall 1.00**, **F1 0.88**, all 32 episodes detected with 0 min delay |
| Anomaly detection, rolling z-score (k=3) | F1 0.36 (masking: a spike inflates the mean/std and hides itself) |
| 5-min forecast, online SGD vs. persistence | MAE **1.67** vs. 1.76 on normal points (**5.3% better**, 15,459 forecasts) |
| Throughput / latency | 1,495 events/s, latency p50 2.9 ms, p95 7.9 ms |
| Time-series analysis | Recovers daily peaks at 07–08 h and 18–19 h and a trend of ≈ +3.7 µg/m³ per day |

## How to run

Requires Python 3.10+.

```bash
git clone https://github.com/utepbergenoff/realtime_data.git
cd realtime_data
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute --inplace main.ipynb   # or open main.ipynb and Run All (~20 s)
```

All parameters are in the `CONFIG` dict at the top of the notebook, which is also saved to `config.json`. With `seed = 42` the generated dataset is identical on every run. Timing numbers (throughput, latency) depend on the machine.

## Files

| Path | Contents |
|---|---|
| `main.ipynb` | Full implementation and results |
| `config.json` | All parameters |
| `data/astana_pm25_stream.csv` | Generated stream: 4 sensors × 3 days × 1 min = 17,280 events, with ground-truth anomaly labels |
| `report/report.pdf` | Technical report (`report.tex` source and `figures/`) |
| `requirements.txt` | Dependencies |
