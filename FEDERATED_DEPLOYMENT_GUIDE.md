# ISAC Radar - Federated Server Deployment Guide

## Overview

Transform your ISAC Radar from a single-node system to a **federated multi-node architecture** with:
- **4+ Edge Nodes** with cameras, radar, LiDAR, GPS
- **Central Hub Server** coordinating all nodes
- **Unified Database** for detection history
- **REST API** for querying and alerts
- **Real-time Dashboard** with WebSocket updates

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   CENTRAL SERVER                             │
│  ├─ Database (SQLite/PostgreSQL/MongoDB)                    │
│  ├─ REST API (Port 5000)                                    │
│  ├─ WebSocket Dashboard (Port 8000)                         │
│  ├─ MQTT Broker (Port 1883)                                 │
│  └─ Analytics & Reporting                                   │
└─────────────────┬───────────────────────────────────────────┘
                  │ MQTT / REST
      ┌───────────┼───────────┬───────────┐
      │           │           │           │
      ▼           ▼           ▼           ▼
   ┌────┐      ┌────┐      ┌────┐      ┌────┐
   │Node│      │Node│      │Node│      │Node│
   │ 1  │      │ 2  │      │ 3  │      │ 4  │
   └────┘      └────┘      └────┘      └────┘
  (Camera)    (Camera)    (Camera)    (Radar)
   +Radar      +Radar      +Radar       +GPS
   +GPS        +GPS        +LiDAR
```

## Quick Start - Docker (5 Minutes)

### Prerequisites
- Docker & Docker Compose installed
- 4GB RAM minimum
- Network connectivity between nodes

### Step 1: Create Project Structure
```bash
mkdir isac-federated && cd isac-federated
mkdir central edge config
```

### Step 2: Create docker-compose.yml
```yaml
version: '3.8'

services:
  mqtt-broker:
    image: eclipse-mosquitto:latest
    ports:
      - "1883:1883"
    volumes:
      - ./mosquitto.conf:/mosquitto/config/mosquitto.conf

  central-server:
    build:
      context: .
      dockerfile: Dockerfile.central
    ports:
      - "5000:5000"
      - "8000:8000"
    environment:
      - MQTT_BROKER=mqtt-broker
      - DATABASE_TYPE=sqlite
    depends_on:
      - mqtt-broker

  edge-node-1:
    build:
      context: .
      dockerfile: Dockerfile.edge
    environment:
      - NODE_ID=edge-1
      - NODE_NAME=Camera-North
      - MQTT_BROKER=mqtt-broker
      - CENTRAL_SERVER=http://central-server:5000
      - CAMERA_INDEX=0
    depends_on:
      - central-server
    devices:
      - /dev/video0:/dev/video0

  edge-node-2:
    build:
      context: .
      dockerfile: Dockerfile.edge
    environment:
      - NODE_ID=edge-2
      - NODE_NAME=Camera-South
      - MQTT_BROKER=mqtt-broker
      - CENTRAL_SERVER=http://central-server:5000
      - CAMERA_INDEX=1
    depends_on:
      - central-server
    devices:
      - /dev/video1:/dev/video1
```

### Step 3: Build and Deploy
```bash
# Build images
docker-compose build

# Start all services
docker-compose up -d

# View logs
docker-compose logs -f central-server
docker-compose logs -f edge-node-1
```

### Step 4: Verify Deployment
```bash
# Check central server
curl http://localhost:5000/api/status

# View all nodes
curl http://localhost:5000/api/nodes
```

## Manual Installation (Python)

### Step 1: Environment Setup
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# or
venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

### Step 2: Configure .env
```env
MQTT_BROKER=localhost
CENTRAL_SERVER_HOST=0.0.0.0
CENTRAL_SERVER_PORT=5000
DATABASE_TYPE=sqlite
DATABASE_URL=sqlite:///isac_detections.db
```

### Step 3: Start Central Server
```bash
python central_server.py
```

### Step 4: Start Edge Nodes (in separate terminals)
```bash
# Node 1
python edge_node.py --node-id=edge-1 --camera=0

# Node 2
python edge_node.py --node-id=edge-2 --camera=1
```

## REST API Reference

### Core Endpoints

#### 1. System Status
```bash
GET /api/status
```
Response:
```json
{
  "server_name": "ISAC-Central-Hub",
  "total_nodes": 4,
  "online_nodes": 3,
  "total_detections": 142
}
```

#### 2. Register Node
```bash
POST /api/nodes/register
Content-Type: application/json

{
  "node_name": "Camera-North",
  "location": "Highway North Gate",
  "sensors": ["camera", "radar", "gps"]
}
```

#### 3. Get All Nodes
```bash
GET /api/nodes
```

#### 4. Submit Detection
```bash
POST /api/detections
Content-Type: application/json

{
  "node_id": "edge-1",
  "label": "person",
  "confidence": 0.92,
  "bbox": [100, 200, 50, 80]
}
```

#### 5. Query Detections
```bash
GET /api/detections?label=person&confidence=0.8&limit=50
```

#### 6. Get Route History
```bash
GET /api/routes/edge-1
```

## Node Registration

### Automatic Registration
Each edge node auto-registers on startup:
```python
# edge_node.py
response = requests.post(
    f"{CENTRAL_SERVER}/api/nodes/register",
    json={
        "node_name": os.getenv("NODE_NAME"),
        "location": os.getenv("NODE_LOCATION"),
        "sensors": ["camera", "radar", "gps"]
    }
)
node_id = response.json()["node_id"]
```

### Manual Registration
```bash
curl -X POST http://localhost:5000/api/nodes/register \
  -H "Content-Type: application/json" \
  -d '{
    "node_name": "Camera-North",
    "location": "Highway North Gate",
    "sensors": ["camera", "radar", "gps"]
  }'
```

## Data Flow

### Detection Flow
```
1. Edge Node detects object via camera
   └─> Confidence: 0.92, Label: person

2. Tracker maintains object across frames
   └─> Track ID: 42, Frames: 4

3. Edge node sends to central server
   └─> POST /api/detections

4. Central server records in database
   └─> Timestamp, Node ID, Coordinates

5. Central server publishes alert
   └─> Email to operator
   └─> MQTT broadcast
   └─> WebSocket to dashboard
```

### Aggregation Flow
```
Central Server queries all nodes:
├─ Node 1: 24 detections
├─ Node 2: 18 detections
├─ Node 3: 31 detections
└─ Node 4: 12 detections
    └─ Total: 85 detections

Analytics:
├─ Person: 32 (37.6%)
├─ Car: 35 (41.2%)
├─ Truck: 18 (21.2%)
└─ Average Confidence: 0.87
```

## Database Setup

### SQLite (Default - Development)
```bash
# Automatic setup on first run
python central_server.py
```

### PostgreSQL (Production)
```bash
# Create database
createdb isac_detections

# Set connection string
export DATABASE_URL=postgresql://user:password@localhost/isac_detections

# Run migrations
python manage.py migrate
```

### MongoDB (High Volume)
```bash
# Set database type
export DATABASE_TYPE=mongodb
export MONGODB_URI=mongodb://localhost:27017/isac_detections

# Start server
python central_server.py
```

## Monitoring & Logging

### View System Logs
```bash
# Central server logs
docker-compose logs central-server

# Edge node logs
docker-compose logs edge-node-1

# Follow real-time logs
docker-compose logs -f
```

### Access Dashboard
```
http://localhost:5000/dashboard
```

### WebSocket Real-time Stream
```bash
# Using wscat
npm install -g wscat
wscat -c ws://localhost:8000/ws/dashboard
```

## Troubleshooting

### Issue: Nodes Not Connecting
**Symptoms**: Central server shows `"online_nodes": 0`

**Solutions**:
1. Check MQTT broker is running
2. Verify network connectivity between nodes and central
3. Check firewall ports (1883, 5000, 8000)
4. Review edge node logs for errors

```bash
# Test MQTT connectivity
docker exec mqtt-broker mosquitto_sub -t "isac/+/detections"
```

### Issue: No Detections Recorded
**Symptoms**: `/api/detections` returns empty array

**Solutions**:
1. Verify camera is connected to edge node
2. Check CONFIDENCE threshold in `.env`
3. Verify YOLO model loaded successfully
4. Check disk space on edge node

```bash
# Check edge node logs
docker-compose logs edge-node-1 | grep -i "detection\|error"
```

### Issue: Database Full
**Symptoms**: Errors writing detections, slow queries

**Solutions**:
1. Archive old detections
2. Upgrade to PostgreSQL/MongoDB
3. Increase storage

```sql
-- Archive detections older than 30 days
DELETE FROM detections WHERE timestamp < NOW() - INTERVAL 30 DAY;
```

## Performance Optimization

### For High Volume (1000+ detections/second)
1. Use PostgreSQL or MongoDB instead of SQLite
2. Deploy Redis for caching
3. Separate API server from database
4. Use Kubernetes for auto-scaling
5. Implement detection batching

### For Real-time (Sub-100ms latency)
1. Use gRPC instead of REST
2. Deploy edge caching
3. Optimize MQTT QoS settings
4. Use GPU on edge nodes

### For Reliability (99.9% uptime)
1. Deploy in HA mode
2. Use managed database
3. Implement circuit breakers
4. Setup automated backups
5. Configure alerting

## Security

### Enable SSL/TLS
```yaml
# docker-compose.yml
central-server:
  environment:
    - SSL_CERT=/etc/ssl/certs/server.crt
    - SSL_KEY=/etc/ssl/private/server.key
  volumes:
    - ./certs:/etc/ssl:ro
```

### API Authentication
```bash
# Add API key
curl http://localhost:5000/api/nodes \
  -H "X-API-Key: your-secret-key-here"
```

### Database Security
```env
# Use environment variables for credentials
DATABASE_URL=postgresql://user:${DB_PASS}@localhost/isac_db
```

## Deployment Checklists

### Pre-Deployment
- [ ] All edge nodes physically connected
- [ ] Cameras tested and working
- [ ] Network connectivity verified
- [ ] Firewall rules configured
- [ ] SSL certificates prepared
- [ ] Database initialized
- [ ] Email alerts configured

### Post-Deployment
- [ ] Central server responding on port 5000
- [ ] All nodes registered and online
- [ ] Test detection submission via API
- [ ] Dashboard accessible
- [ ] WebSocket streaming working
- [ ] Emails sending correctly
- [ ] Logs being recorded

### Maintenance
- [ ] Daily: Check node status
- [ ] Weekly: Review detection trends
- [ ] Monthly: Archive old data
- [ ] Quarterly: Update dependencies
- [ ] Annually: Security audit

## Support & Troubleshooting

For issues, check:
1. System logs: `docker-compose logs`
2. API status: `curl http://localhost:5000/api/status`
3. Node connectivity: `curl http://localhost:5000/api/nodes`
4. Database: Check disk space and connections

---
**ISAC Radar Federated System Ready!** 🚀
