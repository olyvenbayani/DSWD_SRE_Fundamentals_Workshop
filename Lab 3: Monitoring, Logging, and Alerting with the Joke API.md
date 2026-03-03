# Lab 3: Monitoring, Logging, and Alerting with the Joke API

Welcome to Lab 3 of the SRE Workshop! This hands-on activity builds directly on Lab 2 (or Lab 1), extending our funny Joke API to demonstrate monitoring (with Prometheus), logging (adding structured logs to the app), and alerting (using Prometheus Alertmanager). We'll simulate scenarios where the service "breaks" (e.g., high error rates), trigger alerts, and inspect logs to debug—keeping the humor alive with joke-themed log messages.

This guide is for beginners. Each step explains why we're doing it and includes troubleshooting. Assume you've done previous labs; we'll reuse and modify files.

**Time Estimate:** 60-75 minutes.  
**Goals:**  
- Enhance monitoring from previous labs with alerting for SLO breaches.  
- Add logging to track events and errors.  
- Simulate issues, trigger alerts, and use logs to investigate.  
- Connect to previous: The same Joke API now logs jokes and alerts when "too many bad jokes" (high failures) occur.

## Step 1: Set Up the Lab Directory
**Why?** We'll build on previous files to show progression—copy them and make updates.

1. In your terminal, create a new directory and copy files from Lab 2 (or 1):  
   ```
   mkdir sre-lab3
   cp -r ../sre-lab2/* sre-lab3/
   cd sre-lab3
   ```

2. We'll update some files below. Use your text editor (e.g., VS Code).

## Step 2: Update the Sample App for Logging
**Why?** Logging captures what happens inside the app (e.g., requests, errors). We'll use Python's logging module to output structured messages to stdout, which Docker can capture. For fun, logs include the jokes!

Update `app.py` (add logging; keep custom metrics from Lab 2):

```python
from flask import Flask, jsonify
import time
import random
from prometheus_flask_exporter import PrometheusMetrics
from prometheus_client import Counter, Gauge, Histogram, Summary
import logging

# Set up logging to stdout with INFO level
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

app = Flask(__name__)
metrics = PrometheusMetrics(app)

# Export default metrics
metrics.info('app_info', 'Application info', version='1.0.0')

# Custom metrics from Lab 2
joke_counter = Counter('joke_total', 'Total number of jokes told')
humor_gauge = Gauge('humor_level', 'Current humor level (fluctuates)')
joke_length_histogram = Histogram('joke_length_seconds', 'Distribution of joke "delivery" lengths', buckets=[10, 20, 30, 40, 50, 60])
latency_summary = Summary('request_latency_summary_seconds', 'Summary of request latencies')

# List of funny jokes/puns
jokes = [
    "Why did the SRE go to therapy? Too many incidents!",
    "Error budgets are like diets: easy to set, hard to stick to.",
    "Why don't programmers like nature? Too many bugs.",
    "SLOs: Because 'best effort' isn't measurable.",
    "Why was the computer cold? It left its Windows open!",
    "Failure is not an option—it's a feature in beta.",
    "Why did the database administrator leave his wife? She had one too many relationships.",
    "Alert: Your coffee is low— that's the real outage.",
    "Why do SREs make great musicians? They handle scales well.",
    "404: Joke not found. Wait, that's not funny."
]

@app.route('/success')
def success():
    start_time = time.time()
    delay = random.uniform(0.05, 0.15)
    time.sleep(delay)
    joke = random.choice(jokes)
    
    # Update custom metrics
    joke_counter.inc()
    humor_gauge.set(random.uniform(50, 100))
    joke_length_histogram.observe(len(joke))
    latency_summary.observe(time.time() - start_time)
    
    logging.info(f"Successful request! Joke delivered: {joke}")
    return jsonify(message=f"Success! Here's a joke: {joke}"), 200

@app.route('/failure')
def failure():
    start_time = time.time()
    delay = random.uniform(0.2, 0.5)
    time.sleep(delay)
    joke = random.choice(jokes)
    
    # Update custom metrics
    joke_counter.inc()
    humor_gauge.set(random.uniform(0, 50))
    joke_length_histogram.observe(len(joke))
    latency_summary.observe(time.time() - start_time)
    
    logging.error(f"Failure request! Bad joke attempted: {joke}")
    return jsonify(message=f"Oops, failure! But hey: {joke}"), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```

**Explanation:**  
- Added `import logging` and basic config—logs go to console (stdout).  
- In endpoints: Log info for success, error for failure, including the joke for fun.  
- This demonstrates logging: Track events without changing app behavior.


Update your prometheus.yml
```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load rules once and periodically evaluate them
rule_files:
  - "/etc/prometheus/alert-rules.yml"

# Scrape configuration
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'flask-app'
    static_configs:
      - targets: ['app:3000']
```

No changes to `requirements.txt` (already has prometheus-client), `Dockerfile`.

## Step 3: Add Alertmanager to the Setup
**Why?** Alerting notifies when metrics (from monitoring) go bad, like SLO breaches. We'll add Alertmanager to handle alerts from Prometheus.

Update `docker-compose.yml` (add the alertmanager service):

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    container_name: sample-app

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert-rules.yml:/etc/prometheus/alert-rules.yml  # New: Alert rules
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--web.enable-lifecycle'  # Allows reloading config
    container_name: prometheus
    depends_on:
      - app

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    container_name: alertmanager
```

**Explanation:**  
- Added `alertmanager` service with its own config.  
- Mounted alert rules to Prometheus.

Create new file: `alert-rules.yml` (defines when to alert, e.g., high error rate breaching SLO):

```yaml
groups:
  - name: joke-api-alerts
    interval: 15s  # How often to evaluate these rules
    rules:
      # Alert when error rate exceeds 10%
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(flask_http_request_total{status="500"}[5m]))
            /
            sum(rate(flask_http_request_total[5m]))
          ) > 0.1
        for: 1m
        labels:
          severity: warning
          service: joke-api
        annotations:
          summary: "High error rate in Joke API"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 10%). Check application logs immediately."
          dashboard: "http://localhost:9090/graph?g0.expr=rate(flask_http_request_total%7Bstatus%3D%22500%22%7D%5B5m%5D)"
      
      # Alert when latency is too high
      - alert: HighLatency
        expr: |
          histogram_quantile(
            0.99,
            sum(rate(flask_http_request_duration_seconds_bucket[5m])) by (le)
          ) > 0.2
        for: 1m
        labels:
          severity: critical
          service: joke-api
        annotations:
          summary: "High latency detected in Joke API"
          description: "99th percentile latency is {{ $value | humanizeDuration }} (threshold: 200ms). Users experiencing slow responses."
          dashboard: "http://localhost:9090/graph"
      
      # Additional useful alert: Service is down
      - alert: ServiceDown
        expr: up{job="flask-app"} == 0
        for: 30s
        labels:
          severity: critical
          service: joke-api
        annotations:
          summary: "Joke API is down"
          description: "The Flask application has been unreachable for 30 seconds."
      
      # Additional useful alert: High request rate
      - alert: HighRequestRate
        expr: sum(rate(flask_http_request_total[5m])) > 100
        for: 2m
        labels:
          severity: info
          service: joke-api
        annotations:
          summary: "High request rate detected"
          description: "Request rate is {{ $value | humanize }} req/s (threshold: 100 req/s). Possible traffic spike."
```

**Explanation:**  
- `HighErrorRate`: Alerts if errors > 10% (breaching availability SLO).  
- `HighLatency`: Alerts if slow requests > 200ms.  
- Fun annotations tie back to jokes.

Create new file: `alertmanager.yml` (basic config; for demo, alerts show in UI—no email/slack):

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: blackhole

receivers:
  - name: blackhole
```

**Explanation:** Simple setup—alerts will be visible in Alertmanager UI.

## Step 4: Deploy the Updated Setup
**Why?** Starts everything, including alerting.

1. Run:  
   ```
   docker-compose up -d --build
   ```

2. Verify:  
   - App: http://localhost:3000/success (joke response).  
   - Prometheus: http://localhost:9090 (metrics).  
   - Alertmanager: http://localhost:9093 (alerts UI—empty at first).

**Troubleshooting:** Check logs if fails: `docker logs prometheus`.

## Step 5: Explore Monitoring and Logging
**Why?** See baseline before issues.

1. Make a few requests (browser or curl).  
2. View logs: `docker logs sample-app` – See timed messages like "Successful request! Joke delivered: [joke]".

<img width="2023" height="562" alt="image" src="https://github.com/user-attachments/assets/2dc2ba35-3ae9-4566-9472-dae46ed00396" />


3. In Prometheus UI: Query metrics (from Lab 2 explorations). No alerts yet.

**Explanation:** Logging shows events; monitoring shows metrics.

## Step 6: Set Up Alerts in Prometheus
**Why?** Load alert rules.

1. In Prometheus UI > Alerts tab: Rules might not show yet.  
2. Reload config: `curl -X POST http://localhost:9090/-/reload` (needs `web.enable-lifecycle` in compose).  
3. Check Alerts tab: See defined rules.

<img width="3344" height="1339" alt="image" src="https://github.com/user-attachments/assets/fba559a6-78a8-45a0-b085-8ae1fdad8fac" />


## Step 7: Simulate Traffic and Trigger Alerts
**Why?** Test alerting/logging by breaching SLOs.

Update `simulate_traffic.sh` for more failures (to trigger HighErrorRate):

```bash
#!/bin/bash
ab -n 500 -c 10 http://127.0.0.1:3000/success
ab -n 500 -c 10 http://127.0.0.1:3000/failure  # 50% errors to trigger alert
```

1. Run: `./simulate_traffic.sh`  
2. Wait 1-2 minutes.  
3. In Prometheus > Alerts: See firing alerts.  
4. In Alertmanager UI (http://localhost:9093): View active alerts with fun descriptions.  
5. Check logs: `docker logs sample-app` – See error logs for failures.

<img width="2326" height="922" alt="image" src="https://github.com/user-attachments/assets/aa6ef217-5d38-4c62-9263-a3856f42d22f" />

**Explanation:** High failures trigger alert. Logs help debug (e.g., see which requests failed).

## Step 8: Resolve and Reflect
1. Stop bad traffic. Run mostly successes to resolve.  
2. Watch alerts resolve in UIs.  
3. Cleanup: `docker-compose down`  
4. Discuss: How do logs help with alerts? 

**Troubleshooting:** No alerts? Check expr in rules match your metrics (e.g., use Graph tab to test). Adjust thresholds.

You've completed Lab 3! Now your Joke API is monitored, logged, and alerted. 
