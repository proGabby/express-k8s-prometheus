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

Access the Prometheus dashboard:

```bash
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring
```

Then visit: http://localhost:9090

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

## Troubleshooting

### Docker Issues
1. If `docker` command is not found:
   - Ensure Docker Desktop is installed in Applications
   - Verify Docker Desktop is running (whale icon in menu bar)
   - Check if Docker is in your PATH: `echo $PATH`
   - Try adding Docker to PATH in `~/.zshrc` as shown in Prerequisites

2. If pods are in `ImagePullBackOff` state:
   - Ensure Docker Desktop is running
   - Verify the image exists: `docker images | grep express-app`
   - Check image name in deployment.yaml

### Prometheus Issues
1. If metrics are not visible in Prometheus:
   - Verify the PodMonitor is applied: `kubectl get podmonitor`
   - Check if pods have the correct labels
   - Verify the metrics endpoint is accessible
   - Check Prometheus targets: http://localhost:9090/targets

2. If the application is not accessible:
   - Check pod status: `kubectl get pods -l app=express-app`
   - Check service status: `kubectl get svc express-app-service`
   - Verify port-forwarding is working
   - Check pod logs: `kubectl logs -l app=express-app`

## Cleanup

To remove the application and monitoring stack:

```bash
kubectl delete -f k8s/deployment.yaml
kubectl delete -f k8s/service.yaml
kubectl delete -f podmonitor.yaml
helm uninstall prometheus -n monitoring
``` 