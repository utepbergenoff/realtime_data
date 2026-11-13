# Endterm: Real-Time PM2.5 Stream Intelligence (Astana)

Everything is in `main.ipynb`. It contains the data generator, the partitioned producer/worker pipeline, the live dashboard, time-series analysis, anomaly detection and its evaluation, the online ML forecaster and its evaluation, and the operational metrics.

## Run
```bash
cd projects-for-uni && source venv/bin/activate
pip install -r realtime_endterm/requirements.txt   # already installed in this venv
cd realtime_endterm
jupyter nbconvert --to notebook --execute --inplace main.ipynb   # or open the notebook and Run All (~20 s)
```

## Files
- `main.ipynb`: implementation and results
- `config.json`: all parameters (written by the notebook; `seed=42` makes the data reproducible)
- `data/astana_pm25_stream.csv`: generated stream (4 sensors × 3 days × 1-min, with ground-truth anomaly labels)
- `requirements.txt`: dependencies
