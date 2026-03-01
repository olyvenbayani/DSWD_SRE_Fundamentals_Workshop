# Lab 2: Exploring Prometheus Metric Types with the Joke API

Welcome to Lab 2 of the SRE Workshop! This hands-on activity builds on Lab 1, using our funny Joke API to demonstrate the four core Prometheus metric types: Counters, Gauges, Histograms, and Summaries. We'll add custom metrics to the app, observe them in Prometheus, and query each type to understand their behaviors and use cases—all while keeping the humor theme alive (e.g., tracking "joke deliveries" and "humor levels").

This guide is for beginners. Each step explains why we're doing it, how it relates to the metric types, and troubleshooting tips. We'll reuse files from Lab 1 and add custom code using the `prometheus_client` library.

**Time Estimate:** 45-60 minutes.  
**Goals:**  
- Learn the four metric types: Counters (cumulative increases), Gauges (fluctuating values), Histograms (value distributions), and Summaries (precise percentiles).  
- Add and query custom metrics in the app.  
- Simulate scenarios to see how each type behaves.  
- Connect to Lab 1: Extend the Joke API with metrics like "total jokes told" (counter) or "current humor level" (gauge).

## Step 1: Set Up the Lab Directory
**Why?** We'll build on Lab 1 files to show how metrics evolve in a real app.

1. In your terminal, create a new directory and copy Lab 1 files:  
   ```
   mkdir sre-lab2
   cp -r ../sre-lab1/* sre-lab2/
   cd sre-lab2
   ```

2. We'll update some files below. Use your text editor (e.g., VS Code).

## Step 2: Update the App to Include All Metric Types
**Why?** The `prometheus-flask-exporter` from Lab 1 already provides counters (e.g., request totals) and histograms (e.g., latency buckets). We'll add custom ones using `prometheus_client` for gauges and summaries, plus extra counters/histograms for demonstration. This lets us explore all types in one app.

First, update `requirements.txt` to include `prometheus_client` (for custom metrics):

```
flask==3.0.3
prometheus-flask-exporter==0.23.1
prometheus-client==0.20.0  # Add this
```

Update `app.py` (replace the existing one):

```python
from flask import Flask, jsonify
import time
import random
from prometheus_flask_exporter import PrometheusMetrics
from prometheus_client import Counter, Gauge, Histogram, Summary

app = Flask(__name__)
metrics = PrometheusMetrics(app)

# Export default metrics (includes counters and histograms from exporter)
metrics.info('app_info', 'Application info', version='1.0.0')

# Custom metrics for demonstration
joke_counter = Counter('joke_total', 'Total number of jokes told')  # Counter: Cumulative, only increases
humor_gauge = Gauge('humor_level', 'Current humor level (fluctuates)')  # Gauge: Can go up/down
joke_length_histogram = Histogram('joke_length_seconds', 'Distribution of joke "delivery" lengths', buckets=[10, 20, 30, 40, 50, 60])  # Histogram: Buckets for distribution
latency_summary = Summary('request_latency_summary_seconds', 'Summary of request latencies')  # Summary: Percentiles over time

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
    joke_counter.inc()  # Counter: Increment total jokes
    humor_gauge.set(random.uniform(50, 100))  # Gauge: Set fluctuating humor level
    joke_length_histogram.observe(len(joke))  # Histogram: Observe joke length (chars as "seconds" for demo)
    latency_summary.observe(time.time() - start_time)  # Summary: Observe latency
    
    return jsonify(message=f"Success! Here's a joke: {joke}"), 200

@app.route('/failure')
def failure():
    start_time = time.time()
    delay = random.uniform(0.2, 0.5)
    time.sleep(delay)
    joke = random.choice(jokes)
    
    # Update custom metrics (even on failure)
    joke_counter.inc()  # Still counts as a joke attempt
    humor_gauge.set(random.uniform(0, 50))  # Lower humor on failure
    joke_length_histogram.observe(len(joke))
    latency_summary.observe(time.time() - start_time)
    
    return jsonify(message=f"Oops, failure! But hey: {joke}"), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```

**Explanation:**  
- **Counter (`joke_total`):** Tracks total jokes—only increases (cumulative).  
- **Gauge (`humor_level`):** Fluctuates randomly (e.g., high on success, low on failure).  
- **Histogram (`joke_length_seconds`):** Distributes joke lengths into buckets (using char count as demo "seconds").  
- **Summary (`request_latency_summary_seconds`):** Tracks latencies with built-in percentiles.  
- These tie into the fun theme while demonstrating types.

No changes needed to `Dockerfile`, `docker-compose.yml`, or `prometheus.yml`—they scrape all metrics from `/metrics`.

## Step 3: Deploy the Updated App
**Why?** Starts the app with new metrics for exploration.

1. Run:  
   ```
   docker-compose up -d --build
   ```

2. Verify:  
   - App endpoints: http://localhost:3000/success and /failure.  
   - Metrics: http://localhost:3000/metrics (search for new ones like `joke_total`).  
   - Prometheus UI: http://localhost:9090 (query new metrics).

**Troubleshooting:** If metrics don't appear, hit endpoints a few times and wait 15-30s (scrape interval).

## Step 4: Explore Each Metric Type
**Why?** Hands-on querying teaches how each type works. Generate data first by visiting endpoints or running `simulate_traffic.sh` from Lab 1.

In Prometheus UI (http://localhost:9090), use the "Expression" box to query. Switch to "Graph" tab for trends.

### 1. Counters: Tracking Cumulative Values
   - **Why?** Counters increase only (e.g., total events). Useful for rates.  
   - Query examples:  
     - `joke_total`: Raw cumulative value (increases with each request).  
     - `rate(joke_total[5m])`: Rate of jokes per second over 5 minutes.  
     - `flask_http_request_total`: Built-in counter from exporter (total requests).  
   - Activity: Hit /success 10 times. Watch `joke_total` increase. Run simulation—see rate spike.  
   - Reflection: Why can't counters decrease? (Resets on restart for clean starts.)

### 2. Gauges: Measuring Current States
   - **Why?** Gauges fluctuate (up/down). Snapshot of now.  
   - Query examples:  
     - `humor_level`: Current value (changes per request).  
     - `avg_over_time(humor_level[5m])`: Average over 5 minutes.  
   - Activity: Alternate /success (high humor) and /failure (low). Graph it—see ups/downs.  
   - Reflection: Ideal for CPU/memory—why? (Captures real-time state.)

### 3. Histograms: Analyzing Distribution of Values
   - **Why?** Shows how values spread (buckets for approximations).  
   - Query examples:  
     - `joke_length_seconds_bucket`: Raw buckets (e.g., how many jokes <20 chars).  
     - `histogram_quantile(0.95, sum(rate(joke_length_seconds_bucket[5m])) by (le))`: 95th percentile joke length.  
     - `flask_http_request_duration_seconds_bucket`: Built-in for latency distribution.  
   - Activity: Run simulation—query percentile. Note: Buckets are predefined (tweak in code for real use).  
   - Reflection: Great for latency—why approximations? (Efficient for large data.)

### 4. Summaries: Calculating Percentiles and Averages
   - **Why?** Like histograms but precise percentiles over windows.  
   - Query examples:  
     - `request_latency_summary_seconds{quantile="0.99"}`: Direct 99th percentile latency.  
     - `request_latency_summary_seconds_sum / request_latency_summary_seconds_count`: Average latency.  
   - Activity: Simulate traffic—compare to histogram queries. See precise values.  
   - Reflection: Use when exactness matters—why higher resource use? (Stores more data.)

## Step 5: Simulate and Analyze
**Why?** See types in action under load.

1. Update/run `simulate_traffic.sh` (use 127.0.0.1 if needed).  
2. Query all types—note behaviors (e.g., counter always up, gauge varies).  
3. Experiment: Restart container (`docker-compose restart app`)—watch counters reset, gauges persist current state.

## Step 6: Cleanup and Reflect
1. Stop: `docker-compose down`  
2. Think: Which type for what? (E.g., counters for requests, gauges for memory.) How does this help SLOs from Lab 1?

**Troubleshooting Tips:**  
- No custom metrics? Check `/metrics` endpoint—ensure `prometheus_client` imported correctly.  
- Queries fail? Use auto-complete in Prometheus UI (type metric name).  
- High cardinality? Histograms/summaries can be heavy—fine for lab.

You've completed Lab 2! Now you know Prometheus metrics inside out. Save as `lab2-guide.md`. Fun fact: Imagine alerting on low "humor_level"! Ready for Lab 3?
