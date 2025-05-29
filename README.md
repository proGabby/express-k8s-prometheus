# Express.js Application with Prometheus Monitoring

This is a Node.js application built with Express.js that includes Prometheus metrics monitoring. The application exposes metrics that can be scraped by Prometheus, allowing you to monitor various aspects of your application's performance.

## Prerequisites

Before you begin, ensure you have the following installed:

### 1. Node.js
- Install [Node.js](https://nodejs.org/) (v18 or later)
- Verify installation: `node --version`

### 2. Docker Desktop
- Download [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop)
- Install by dragging Docker.app to your Applications folder
- Open Docker Desktop from Applications
- Wait for Docker Desktop to start (whale icon appears in menu bar)
- Add Docker to your PATH:
  ```bash
  # Open .zshrc in nano
  nano ~/.zshrc
  
  # Add this line to the file
  export PATH="/Applications/Docker.app/Contents/Resources/bin:$PATH"
  
  # Save and exit nano:
  # 1. Press Ctrl + X
  # 2. Press Y to save
  # 3. Press Enter to confirm
  
  # Reload your shell configuration
  source ~/.zshrc
  
  # Verify Docker is accessible
  docker --version
  ```

### 3. Kubernetes Tools
- Install [kubectl](https://kubernetes.io/docs/tasks/tools/)
- Install [Helm](https://helm.sh/docs/intro/install/)
- Enable Kubernetes in Docker Desktop:
  1. Open Docker Desktop
  2. Click the gear icon (Settings)
  3. Go to "Kubernetes"
  4. Check "Enable Kubernetes"
  5. Click "Apply & Restart"

## Project Structure

```
.
├── Dockerfile              # Container definition
├── index.js               # Main application code
├── package.json           # Node.js dependencies
├── k8s/                   # Kubernetes manifests
│   ├── deployment.yaml    # Application deployment
│   └── service.yaml       # Service definition
└── podmonitor.yaml        # Prometheus PodMonitor configuration
```

## Application Features

- Express.js web server running on port 3000
- Prometheus metrics endpoint at `/metrics`
- Health check endpoint at `/health`
- Custom metrics for HTTP request duration and total requests
- Kubernetes deployment with 3 replicas
- Prometheus monitoring integration

## Setup and Installation

### 1. Install Dependencies

```bash
npm install
```

### 2. Build and Run Locally

To run the application locally:

```bash
node index.js
```

The application will be available at `http://localhost:3000`

### 3. Build Docker Image

Make sure Docker Desktop is running, then:

```bash
# Verify Docker is accessible
docker --version

# Build the application image
docker build -t express-app:latest .
```

### 4. Deploy to Kubernetes

#### Install Prometheus Stack

First, add the Prometheus Helm repository and install the kube-prometheus-stack:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

#### Deploy the Application

Apply the Kubernetes manifests:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f podmonitor.yaml
```

### 5. Verify Deployment

Check if the pods are running:

```bash
kubectl get pods -l app=express-app
```

Check if the service is created:

```bash
kubectl get svc express-app-service
```

## Accessing the Application

### Local Access

To access the application locally, port-forward the service:

```bash
kubectl port-forward svc/express-app-service 3000:3000
```

Then visit:
- Application: http://localhost:3000
- Health check: http://localhost:3000/health
- Metrics: http://localhost:3000/metrics

### Prometheus Dashboard

To access the Prometheus dashboard and view your application metrics:

1. Port-forward the Prometheus service to your local machine:
```bash
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring
```

2. Open your browser and navigate to:
   - Prometheus UI: http://localhost:9090

3. In the Prometheus UI, you can:
   - Go to Status -> Targets to verify your application is being scraped
   - Go to Graph to query metrics
   - Try these example queries:
     - `http_requests_total` - Total number of HTTP requests
     - `http_request_duration_seconds` - Request duration histogram
     - `nodejs_memory_heap_used_bytes` - Node.js memory usage

4. To verify your application metrics are being collected:
   - Click on "Status" -> "Targets"
   - Look for targets with label `app=express-app`
   - Status should show as "UP"
   - Last scrape time should be recent

Note: Keep the port-forward running in your terminal while accessing the dashboard. To stop port-forwarding, press Ctrl+C in the terminal.

## Available Metrics

The application exposes the following metrics:

- `http_request_duration_seconds`: Histogram of HTTP request durations
  - Labels: method, route, status
- `http_requests_total`: Counter of total HTTP requests
  - Labels: method, route, status
- Default Node.js metrics (memory, CPU, etc.)

## Monitoring Configuration

The `podmonitor.yaml` file configures Prometheus to scrape metrics from the application:

- Scrape interval: 30 seconds
- Scrape timeout: 10 seconds
- Metrics path: /metrics
- Port: metrics (3000)

## Querying Metrics in Prometheus

### Basic Queries

1. Access the Prometheus UI (http://localhost:9090) and click on "Graph" in the top menu

2. To query `http_requests_total`:
   - Type `http_requests_total` in the query box
   - Click "Execute" or press Enter
   - This shows the total count of all HTTP requests

3. Filter by specific labels:
   - `http_requests_total{method="GET"}` - Only GET requests
   - `http_requests_total{route="/health"}` - Requests to health endpoint
   - `http_requests_total{status="200"}` - Successful requests
   - `http_requests_total{method="GET", route="/metrics"}` - GET requests to metrics endpoint

4. Time-based queries:
   - `rate(http_requests_total[5m])` - Requests per second over 5 minutes
   - `increase(http_requests_total[1h])` - Total requests in the last hour

### Example Queries

1. Request rate by endpoint:
```
rate(http_requests_total[5m])
```

2. Success rate (percentage of 200 responses):
```
sum(rate(http_requests_total{status="200"}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

3. Request duration percentiles:
```
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

4. Top 5 endpoints by request count:
```
topk(5, sum(http_requests_total) by (route))
```

### Tips for Querying

1. Use the "Graph" tab to visualize metrics over time
2. Use the "Table" tab to see current values
3. Adjust the time range using the controls above the graph
4. Use the "Add Panel" button to compare multiple metrics
5. Click on a metric in the graph to see its exact values

### Common Issues

1. If you see "No data":
   - Check if the target is "UP" in Status -> Targets
   - Verify the metric name is correct
   - Check if the time range is appropriate

2. If the graph is empty:
   - Try a longer time range
   - Check if the application is receiving traffic
   - Verify the scrape interval in podmonitor.yaml

## Troubleshooting

### Docker Issues
1. If `