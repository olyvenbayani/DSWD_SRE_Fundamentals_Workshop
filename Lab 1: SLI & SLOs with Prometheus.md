## Lab 1: SLI/SLOs with Prometheus
Welcome to Lab 1 of the SRE Workshop! This hands-on activity introduces you to Service Level Indicators (SLIs) and Service Level Objectives (SLOs) using a simple, fun web application monitored by Prometheus. We'll build a Python app that tells jokes, containerize it with Docker, set up monitoring, define SLIs and SLOs, and validate them with simulated traffic.
This guide is designed for beginners. Each step includes explanations, why we're doing it, and troubleshooting tips. We'll assume you've completed Lab 0 (setup with Docker, Docker Compose, etc.). If not, go back and install those tools first.
**Time Estimate:** 45-60 minutes.
**Goals:**
- Understand SLIs (measurable aspects of service quality, like latency).
- Set SLOs (targets for SLIs, like "99% of requests under 200ms").
- Use Prometheus to monitor and query metrics.
- Simulate real-world scenarios to check if SLOs are met.

## For EC2 Users: Additional Setup Notes (Skip if you're doing this on local machine)
If you're running this lab on an AWS EC2 instance (as described in Lab 0's alternative cloud setup), follow these extra steps to handle port access and remote connectivity. This ensures you can access the app and Prometheus from your local browser while keeping the instance secure.

### 1. Configure Security Group for Ports
**Why?** EC2 instances block inbound traffic by default. You need to allow ports 3000 (app) and 9090 (Prometheus) so you can access them remotely via the instance's public IP.

1. Go to the AWS EC2 console: [https://console.aws.amazon.com/ec2/](https://console.aws.amazon.com/ec2/).
2. Select your instance from the list.
3. In the instance details, scroll to **Security** and click the linked **Security group** (e.g., sg-XXXXXX).
4. In the Security Groups view, select the group and go to the **Inbound rules** tab.
5. Click **Edit inbound rules**.
6. Add two new rules:
   - **Type**: Custom TCP, **Port range**: 3000, **Source**: 0.0.0.0/0 (for testing; restrict to your IP for security, e.g., My IP).
   - **Type**: Custom TCP, **Port range**: 9090, **Source**: 0.0.0.0/0 (same as above).
7. Save the rules.

**Security Tip:** After the lab, remove or restrict these rules to avoid exposing ports unnecessarily. Use your public IP (find it via [whatismyip.com](https://whatismyip.com)) instead of 0.0.0.0/0 in production scenarios.

### 2. Running Commands on EC2
- Use VS Code's Remote-SSH connection (from Lab 0) to open the lab directory on the EC2 instance.
- Run all terminal commands (e.g., `docker-compose up`) in VS Code's integrated terminal, which executes them remotely on EC2.
- Find your EC2 public IP/DNS: In the EC2 console, select your instance and copy the **Public IPv4 address** or **Public IPv4 DNS** (e.g., ec2-XXX-XXX-XXX-XXX.compute-1.amazonaws.com).

### 3. Accessing Services
- Replace `localhost` in URLs with your EC2 public IP/DNS (e.g., http://ec2-XXX-XXX-XXX-XXX.compute-1.amazonaws.com:3000/success).
- If using a public IP, it might look like http://XXX.XXX.XXX.XXX:3000/success.
- For internal scripts like `simulate_traffic.sh`, `localhost` is fine when run on EC2—it refers to the instance itself.

**Troubleshooting:**
- Connection refused? Double-check security group rules and ensure the instance is running.
- Timeout? Verify your local firewall isn't blocking outbound connections.
- No public IP? Ensure your instance is in a public subnet (or launch a new one with auto-assign public IP enabled).

Proceed with the main steps below, using the EC2-specific URLs where noted.

## Step 1: Create the Lab Directory
**Why?** We need an organized folder to hold all files for the app, configs, and docs. This keeps everything tidy.
1. Open your terminal (Command Prompt on Windows, Terminal on macOS/Linux). For EC2 users: Use VS Code's remote terminal.
2. Create a new directory:
   ```
   mkdir sre-lab1
   cd sre-lab1
   ```
3. Inside this directory, we'll create the files below. Use a text editor like VS Code to make them.

## Step 2: Build the Sample App
**Why?** This is a simple Python web app using Flask. It has two endpoints: `/success` (works fine) and `/failure` (simulates errors). We've added funny SRE-themed jokes to make testing enjoyable. The app exposes metrics for Prometheus to monitor requests, errors, and latency.
Create these files in your `sre-lab1` directory:
### File: `app.py` (The Main App Code)
This script runs the web server. It includes random delays to simulate real latency and exports metrics.
```python
from flask import Flask, jsonify
import time
import random
from prometheus_flask_exporter import PrometheusMetrics
app = Flask(__name__)
metrics = PrometheusMetrics(app)
# Export default metrics
metrics.info('app_info', 'Application info', version='1.0.0')
# List of funny jokes/puns (SRE-themed where possible)
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
    # Simulate variable latency: 50-150ms
    delay = random.uniform(0.05, 0.15)
    time.sleep(delay)
    joke = random.choice(jokes)
    return jsonify(message=f"Success! Here's a joke: {joke}"), 200
@app.route('/failure')
def failure():
    # Simulate error with higher latency: 200-500ms
    delay = random.uniform(0.2, 0.5)
    time.sleep(delay)
    joke = random.choice(jokes)
    return jsonify(message=f"Oops, failure! But hey: {joke}"), 500
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```
**Explanation:**
- `Flask` creates the web app.
- `prometheus_flask_exporter` adds monitoring (e.g., tracks request counts and times).
- Endpoints return JSON with jokes. Success is fast and reliable; failure is slow and errors out.
- Port 3000 avoids common conflicts (like default Flask port 5000).
### File: `requirements.txt` (Dependencies)
Lists Python packages needed.
```
flask==3.0.3
prometheus-flask-exporter==0.23.1
```
**Explanation:** Docker will install these when building the container. No need to install them locally.
### File: `Dockerfile` (Build Instructions for the App Container)
Defines how to package the app into a Docker image.
```
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 3000
CMD ["python", "app.py"]
```
**Explanation:**
- Starts from a lightweight Python image.
- Copies files and installs dependencies.
- Exposes port 3000 for access.
### File: `docker-compose.yml` (Run Multiple Containers)
Sets up the app and Prometheus together.
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
    container_name: prometheus
    depends_on:
      - app
```
**Explanation:**
- `app`: Builds from your Dockerfile.
- `prometheus`: Uses a pre-built image, mounts a config file.
- Ports: App on 3000 (local and container), Prometheus UI on 9090.
- No `version` line to avoid warnings.
### File: `prometheus.yml` (Prometheus Config)
Tells Prometheus what to monitor.
```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'sample-app'
    static_configs:
      - targets: ['app:3000']
```
**Explanation:**
- Scrapes metrics from the app every 15 seconds.
- Targets the app container at its internal port 3000.

## Step 3: Deploy the App
**Why?** This starts the containers so you can interact with the app and monitor it.
1. In your terminal (in the `sre-lab1` directory). For EC2 users: Use VS Code's remote terminal:
   ```
   docker-compose up -d --build
   ```
   - `--build` builds the app image.
   - `-d` runs in the background.
2. Wait a few seconds, then check if it's running:
   ```
   docker ps
   ```
   - You should see `sample-app` and `prometheus`.
 
<img width="1241" height="650" alt="Lab1-01" src="https://github.com/user-attachments/assets/431887ee-1a0d-4d82-a9f0-bd0065b44f95" />
**Troubleshooting:**
- If it fails (e.g., port conflict), run `docker-compose down`, then try again. Or change the host port in `docker-compose.yml` (e.g., "3001:3000").
- View logs: `docker logs sample-app`.
- For EC2: If Docker fails to start, ensure you followed Lab 0's installation steps (e.g., `sudo systemctl start docker`).

## Step 4: Verify the App
**Why?** Ensures everything works before monitoring.
1. Open a browser:
   - http://localhost:3000/success – See a success message with a joke (HTTP 200). For EC2 users: Use http://<EC2-PUBLIC-IP>:3000/success or http://<EC2-PUBLIC-DNS>:3000/success.
   - http://localhost:3000/failure – See a failure message with a joke (HTTP 500). For EC2 users: Same as above, replace localhost.
 
<img width="1639" height="538" alt="Lab1-02" src="https://github.com/user-attachments/assets/fadb5413-6b1d-44c6-94dd-baa5828dde14" />
2. Check metrics: http://localhost:3000/metrics – Raw data for Prometheus. For EC2: Use http://<EC2-PUBLIC-IP>:3000/metrics.
<img width="1160" height="770" alt="Lab1-05" src="https://github.com/user-attachments/assets/3c0c263e-6089-4a44-ae90-8ba6d85cb569" />
3. Open Prometheus UI: http://localhost:9090. For EC2: http://<EC2-PUBLIC-IP>:9090.
   - In the "Expression" box, query `http_server_requests_total` and click "Execute". You should see request counts.
 
<img width="3342" height="842" alt="Lab1-03" src="https://github.com/user-attachments/assets/2b7cdde7-4592-4d54-b84f-bc0293d7f61c" />
**Explanation:** These tests confirm the app responds and exposes metrics.
**Explore More Queries:**
To get comfortable with Prometheus, try these additional queries in the Expression box. Click "Execute" after each one, and observe the results in the table or graph view (switch tabs above the results). This helps you understand the metrics before defining SLIs:
- flask_http_server_requests_total{status="200"}: Counts only successful requests (HTTP 200).
- http_server_requests_total{status="500"}: Counts error requests (HTTP 500).
- rate(http_server_requests_total[5m]): Shows the rate of requests per second over the last 5 minutes.
- histogram_quantile(0.50, sum(rate(http_server_requests_seconds_bucket[5m])) by (le)): Median (50th percentile) latency—how long half of requests take.
- histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le)): 99th percentile latency—how long the slowest 1% of requests take.
- up: Checks if the app is up (1 = up, 0 = down).
Why explore? These build on the basics and prepare you for Step 5. Experiment by making more requests to the app (e.g., reload the browser pages) and see how numbers change. If a query returns "No data," generate traffic by visiting the endpoints multiple times.
Explanation: These tests confirm the app responds and exposes metrics.

## Step 5: Define SLIs
**Why?** SLIs are the "what" we measure (e.g., error rate). We'll document them.
Create `slis.md`:
```
# SLIs for Sample API
1. Availability: Percentage of requests that return HTTP 200.
   - Why? Measures if the service is up and working.
   - Formula: (successful_requests / total_requests) * 100
   - Prometheus Query: sum(rate(flask_http_request_total{status="200"}[5m])) / sum(rate(flask_http_request_total[5m])) * 100
2. Latency: 99th percentile of request times.
   - Why? Shows how fast the service is for most users.
   - Formula: 99% of requests under a threshold (e.g., 200ms).
   - Prometheus Query: histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))
```
**Explanation:** Use these queries in Prometheus to see real values. Adjust based on your needs.

## Step 6: Set SLOs
**Why?** SLOs are targets (e.g., "99.9% availability"). They make SLIs actionable.
Create `slos.yaml`:
```yaml
service: sample-api
slos:
  - name: availability
    objective: 99.9%
    window: 28 days # Use 5 minutes for lab testing
    sli: proportion of HTTP 200 responses
    justification: Users expect the service to be reliable.
  - name: latency
    objective: 99% of requests < 200ms
    window: 28 days
    sli: 99th percentile request duration
    justification: Fast responses keep users happy.
```
**Explanation:** In production, windows are long; here, short for quick tests.

## Step 7: Simulate Traffic and Validate
**Why?** Real traffic generates data to check if SLOs hold.
Create `simulate_traffic.sh`:
```bash
#!/bin/bash
ab -n 800 -c 10 http://localhost:3000/success
ab -n 200 -c 10 http://localhost:3000/failure
```

1. Make it runnable: `chmod +x simulate_traffic.sh`
2. Run: `./simulate_traffic.sh`. For EC2 users: Run this in the VS Code remote terminal—`localhost` works as it's internal to the instance.
<img width="1160" height="770" alt="Lab1-05" src="https://github.com/user-attachments/assets/04c36c1a-7b28-41b8-9741-51faa9a4d7f1" />


For Windows:
```
@'
# --- load_test.ps1 Script Content ---
$TotalRequests = 800
$TargetUrl = "http://localhost:3000/success"
$Successes = 0
$Failures = 0

Write-Host "Starting load test: $TotalRequests requests to $TargetUrl"
Write-Host "----------------------------------------------------"

1..$TotalRequests | ForEach-Object {
    try {
        # Send the request and store the result
        $Response = Invoke-WebRequest -Uri $TargetUrl -UseBasicParsing -ErrorAction Stop

        # Check for success (e.g., HTTP status 200)
        if ($Response.StatusCode -eq 200) {
            $Successes++
        } else {
            $Failures++
        }
    } catch {
        # Catch connection errors, timeouts, or stopped errors
        $Failures++
    }

    # Optional: Print progress every 100 requests
    if ($_ % 100 -eq 0) {
        Write-Host "Progress: $_ requests sent."
    }
}

Write-Host "----------------------------------------------------"
Write-Host "--- Test Results ---"
Write-Host "Total Requests: $TotalRequests"
Write-Host "Successful Requests: $Successes"
Write-Host "Failed Requests: $Failures"
'@ | Set-Content -Path .\load_test.ps1 -Force


3. In Prometheus UI: Re-run SLI queries. For EC2: Access the UI at http://<EC2-PUBLIC-IP>:9090.
   - Availability: Should be ~80% (due to failures). Is it above 99.9%? (No—discuss why and adjust script for passing tests.)
```

   Run prom query:
```
sum(rate(flask_http_request_total{status="200"}[5m])) /
sum(rate(flask_http_request_total[5m]))
```
**How to check it:**
Run the query
If it's ≥ 0.999 → SLO met
If it's < 0.999 → SLO violated
<img width="3310" height="1470" alt="Lab1-07" src="https://github.com/user-attachments/assets/bc09ad45-d8fe-467f-af0c-87e86d18a410" />
    
   - Latency: Check if most are under 200ms.
```
histogram_quantile(0.99, sum(rate(flask_http_request_duration_seconds_bucket[1d])) by (le))

```
**How to check it:**
Run the query
Look at the output
If value is < 0.2 seconds → SLO met
If value is ≥ 0.2 seconds → SLO violated
**Explanation:** `ab` sends requests. Adjust numbers for different scenarios.

## Step 8: Cleanup and Reflect
1. Stop: `docker-compose down`. For EC2: Run in the remote terminal.
2. Think: How would you change SLOs for a real app? What if latency spikes?
**Troubleshooting Tips for Beginners:**
- Errors? Check logs with `docker logs <container>`.
- No metrics? Ensure the app is hit a few times.
- Stuck? Search "Docker [error message]" online.
- For EC2: If ports aren't accessible, verify security groups and public IP. Restart the instance if needed (EC2 console > Actions > Reboot).

You've completed Lab 1! If issues arise, ask your instructor. Ready for Lab 2?
