# Kafka Integration Setup Guide

This guide explains how to connect CodeXray to Kafka when deploying both the main CodeXray service and the Anomaly Service in a VM.

## Prerequisites

- Docker and Docker Compose installed
- Both docker-compose files ready to deploy
- Access to CodeXray UI to get the session cookie and project ID

## Deployment Steps

### Step 1: Start the Main CodeXray Services

```bash
cd /path/to/codexray-deploy
sudo docker-compose up -d
```

This will start:
- CodeXray (port 8080)
- Prometheus (port 9090)
- ClickHouse (port 8043)
- PostgreSQL (port 8004)
- Node Agent

### Step 2: Start the Anomaly Service Stack

```bash
cd /path/to/codexray-deploy/anomaly-service
sudo docker-compose -f docker-compose up -d
```

This will start:
- Anomaly Service (ports 8082, 8000)
- Kafka (ports 29092, 9093)
- Kafka UI (port 8083)
- Zookeeper (port 22181)
- Redis (port 6379)

### Step 3: Connect CodeXray to Anomaly Service Network

The CodeXray and Anomaly Service containers run on separate Docker networks by default. To allow CodeXray to communicate with Kafka, connect the CodeXray container to the anomaly service network:

```bash
sudo docker network connect anomaly-service_anomaly-service-network codexray-deploy_codexray_1
```

### Step 4: Get Your Session Cookie and Project ID

1. **Login to CodeXray UI**: Open http://localhost:8080 (or your VM's IP:8080)
2. **Open Browser DevTools**: Press F12
3. **Go to Application/Storage tab**: Find Cookies
4. **Copy the `codexray_session` cookie value**
5. **Get your Project ID**: Navigate to your project and copy the ID from the URL (e.g., `f1gr424o`)

### Step 5: Configure Kafka Integration

Send a PUT request to configure the Kafka integration:

```bash
curl -X PUT http://localhost:8080/api/project/<PROJECT_ID>/integrations/kafka \
  -H "Content-Type: application/json" \
  -H "Cookie: codexray_session=<YOUR_SESSION_COOKIE>" \
  -d '{
    "brokers": ["kafka:29092"],
    "global": false,
    "metrics_topic": "codexray-metrics"
  }'
```

**Replace:**
- `<PROJECT_ID>` with your actual project ID
- `<YOUR_SESSION_COOKIE>` with the cookie value you copied

**Example:**
```bash
curl -X PUT http://localhost:8080/api/project/f1gr424o/integrations/kafka \
  -H "Content-Type: application/json" \
  -H "Cookie: codexray_session=eyJpZCI6MX0=.oOycxgOplsKenRdpFw81w88x4HdSlXW36l_vvo99YJI=" \
  -d '{
    "brokers": ["kafka:29092"],
    "global": false,
    "metrics_topic": "codexray-metrics"
  }'
```

### Step 6: Verify the Connection

#### 6.1 Check the API Response
The curl command should return `HTTP/1.1 200 OK`

#### 6.2 Verify Configuration
```bash
curl -X GET http://localhost:8080/api/project/<PROJECT_ID>/integrations/kafka \
  -H "Cookie: codexray_session=<YOUR_SESSION_COOKIE>"
```

Expected response:
```json
{
  "global": false,
  "brokers": ["kafka:29092"],
  "auth": {"user": "", "password": ""},
  "tls_enable": false,
  "tls_skip_verify": false,
  "metrics_topic": "codexray-metrics"
}
```

#### 6.3 Check Kafka Topic
Verify the `codexray-metrics` topic exists:
```bash
sudo docker exec kafka kafka-topics --bootstrap-server localhost:29092 --list
```

You should see:
```
__consumer_offsets
codexray-metrics
```

#### 6.4 Check CodeXray Logs
```bash
sudo docker logs codexray-deploy_codexray_1 --tail 20 | grep -i kafka
```

Expected output:
```
I... successfully connected to kafka brokers kafka:29092, found 2 topics
```

#### 6.5 Monitor Kafka UI
Open http://localhost:8083 (or your VM's IP:8083) to view:
- Kafka brokers status
- Topics and their messages
- Consumer groups

## Important Notes

### Network Configuration
- **CodeXray Network**: `codexray-network`
- **Anomaly Service Network**: `anomaly-service-network`
- **Bridge Connection**: Required via Step 3 to allow cross-network communication

### Broker Addresses
- **Inside Docker Network**: Use `kafka:29092` (container hostname)
- **From Host Machine**: Use `localhost:9093` or `<VM_IP>:9093`
- **Never use** `localhost:9093` in the integration config - it won't work from inside the CodeXray container

### ClickHouse Connection
The Anomaly Service connects to ClickHouse using:
- **Host**: `host.docker.internal` (configured in docker-compose)
- **Port**: `8043`
- **User**: `default`
- **Password**: `vizares`

This is already configured in the anomaly-service docker-compose file.

## Troubleshooting

### Connection Refused Error
If you see `connection refused` errors:
1. Ensure Step 3 (network connection) was completed
2. Verify Kafka is running: `sudo docker ps | grep kafka`
3. Use `kafka:29092` not `localhost:9093` in the integration config

### Topic Not Created
If the `codexray-metrics` topic doesn't exist:
1. Check Kafka logs: `sudo docker logs kafka --tail 50`
2. Create manually if needed:
   ```bash
   sudo docker exec kafka kafka-topics --create \
     --bootstrap-server localhost:29092 \
     --topic codexray-metrics \
     --partitions 1 \
     --replication-factor 1
   ```

### Anomaly Service Not Consuming
Check anomaly service logs:
```bash
sudo docker logs anomalies-service --tail 50
```

Verify it's connected to both Kafka and ClickHouse.

### Session Cookie Expired
If you get authentication errors:
1. The session cookie may have expired
2. Login to CodeXray UI again
3. Get a fresh cookie from DevTools
4. Retry the curl command

## Persistence After Restart

The network connection created in Step 3 is **not persistent** across Docker restarts. 

### Option 1: Reconnect After Restart
After restarting Docker or the containers, run Step 3 again:
```bash
sudo docker network connect anomaly-service_anomaly-service-network codexray-deploy_codexray_1
```

### Option 2: Automate with a Script
Create a startup script:
```bash
#!/bin/bash
cd /path/to/codexray-deploy
sudo docker-compose up -d

cd /path/to/codexray-deploy/anomaly-service
sudo docker-compose -f docker-compose up -d

# Wait for containers to be ready
sleep 10

# Connect networks
sudo docker network connect anomaly-service_anomaly-service-network codexray-deploy_codexray_1

echo "All services started and connected!"
```

## Service URLs

After successful deployment:

- **CodeXray UI**: http://localhost:8080
- **Prometheus**: http://localhost:9090
- **Kafka UI**: http://localhost:8083
- **Anomaly Service Health**: http://localhost:8082/health
- **Anomaly Service Alt Port**: http://localhost:8000

Replace `localhost` with your VM's IP address when accessing from outside the VM.

## Stopping Services

To stop all services:

```bash
# Stop main CodeXray services
cd /path/to/codexray-deploy
sudo docker-compose down

# Stop anomaly services
cd /path/to/codexray-deploy/anomaly-service
sudo docker-compose -f docker-compose down
```

To stop and remove all data (volumes):
```bash
sudo docker-compose down -v
```

