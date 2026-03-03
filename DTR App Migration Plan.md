# Critical DTR App Migration POC

# Docker + EC2 + Auto Scaling + ALB Lab (Using Docker Hub - Option B)

## 🎯 Objective

In this lab, you will:

-   Deploy a Dockerized app to EC2
-   Use Docker Hub as the image repository
-   Attach EC2 to an Auto Scaling Group (ASG)
-   Place it behind an Application Load Balancer (ALB)
-   Stress test the application
-   Deploy a new version using `latest` tag
-   Trigger instance refresh for rolling update

------------------------------------------------------------------------

# 🏗 Architecture Overview

Internet → ALB → Target Group → Auto Scaling Group → EC2 (Docker running
app)

Scaling Trigger: CPU Utilization (Target Tracking 50%)

------------------------------------------------------------------------

# 🧑‍💻 Step 1: Create Simple App

## app.js

``` javascript
const express = require('express');
const os = require('os');
const app = express();

const VERSION = process.env.VERSION || "v1";

app.get('/', (req, res) => {
    res.send(`
        <h1>App Version: ${VERSION}</h1>
        <p>Hostname: ${os.hostname()}</p>
    `);
});

app.listen(8080, () => console.log("Running on 8080"));
```

------------------------------------------------------------------------

## Dockerfile

``` dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install express
EXPOSE 8080
CMD ["node", "app.js"]
```

------------------------------------------------------------------------

# 🐳 Step 2: Build & Push to Docker Hub

``` bash
docker login

docker build -t yourdockerhubusername/docker-asg-demo:latest .

docker push yourdockerhubusername/docker-asg-demo:latest
```

------------------------------------------------------------------------

# 🖥 Step 3: Create Launch Template

User Data Script:

``` bash
#!/bin/bash
yum update -y
yum install -y docker
service docker start

docker pull yourdockerhubusername/docker-asg-demo:latest

docker run -d -p 80:8080 -e VERSION=v1 --name demo yourdockerhubusername/docker-asg-demo:latest
```

------------------------------------------------------------------------

# ⚖ Step 4: Create Application Load Balancer

-   Internet-facing
-   Listener: HTTP 80
-   Target Group:
    -   Target type: Instance
    -   Port: 80
    -   Health check path: /

Attach ASG to Target Group.

------------------------------------------------------------------------

# 📈 Step 5: Create Auto Scaling Group

Settings:

-   Min: 1
-   Desired: 1
-   Max: 4
-   Scaling Policy:
    -   Target Tracking
    -   CPU Utilization: 50%

------------------------------------------------------------------------

# 🔥 Step 6: Verify Application

Open ALB DNS.

Expected output:

App Version: v1\
Hostname: ip-xxx-xxx-xxx

Refresh multiple times to observe hostname changes after scaling.

------------------------------------------------------------------------

# 💥 Step 7: Stress Test

From your machine:

``` bash
ab -n 10000 -c 200 http://<alb-dns>/
```

Observe:

-   CPU spike
-   ASG launches new instances
-   Multiple hostnames appear

------------------------------------------------------------------------

# 🚀 Step 8: Deploy New Version

Modify app.js:

``` javascript
const VERSION = process.env.VERSION || "v2";
```

Rebuild and push:

``` bash
docker build -t yourdockerhubusername/docker-asg-demo:latest .
docker push yourdockerhubusername/docker-asg-demo:latest
```

------------------------------------------------------------------------

# 🔄 Step 9: Trigger Rolling Update

Run:

``` bash
aws autoscaling start-instance-refresh   --auto-scaling-group-name docker-asg-demo-asg
```

Instances will rotate gradually.

Verify ALB now shows:

App Version: v2

------------------------------------------------------------------------

# 🧠 Key Concepts Learned

-   Immutable Infrastructure
-   Stateless Application Design
-   Horizontal Scaling
-   Rolling Deployment via Instance Refresh
-   ALB Health Checks
-   Target Tracking Scaling Policy

------------------------------------------------------------------------

# ⚠ Important Note

Using `latest` tag is acceptable for lab/demo purposes.

For production: - Use versioned tags - Update Launch Template
dynamically - Avoid mutable image references

------------------------------------------------------------------------

# 🎉 Lab Complete

You have successfully:

-   Deployed containerized app on EC2
-   Implemented Auto Scaling
-   Configured Load Balancing
-   Performed stress testing
-   Executed rolling deployment
