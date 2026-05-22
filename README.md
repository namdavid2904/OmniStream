# Endless Eight — Scalable IoT Ingestion Infrastructure & Fault-Tolerant Alerting Engine




**Team:** Nam Pham, Eric Gu, Zaeem Chaudhary, Justin Manoj, Sean Young, Srikar Kopparapu, William Hammond, Nathan Chen
**Repository:** https://github.com/Sri200519/team-8-iot.git




---




## Team and Service Ownership




| Team Member | Services / Components Owned                            |
| ----------- | ------------------------------------------------------ |
| Zaeem Chaudhary       | `Sensor Registry Service`     |
| Srikar Kopparapu      | `Device Management Service`               |
| Nathan Chen           | `Ingestion Service`             |
| Justin Manoj          | `Report Generation Worker`               |
| Sean Young            | `Alert Service`               |
| Nam Pham              | `Storage Worker`           |
| William Hammond       | `Anomaly Worker`               |
| Eric Gu               | `Dashboard API` |




> Ownership is verified by `git log --author`. Each person must have meaningful commits in the directories they claim.




---




## How to Start the System




```bash
# Start everything (builds images on first run)
docker compose up --build




# Start with service replicas (Sprint 4)
docker compose up --scale dashboard-api=3 --scale ingestion=3 --scale sensor-registry-service=3 –-build




# Verify all services are healthy
docker compose ps




# Stream logs
docker compose logs -f




# Open a shell in the holmes investigation container
docker compose exec holmes bash
```


## Service Access


### Public entrypoint through Caddy


- `http://localhost:80`


### Caddy routes


- `/readings*` -> `ingestion-service:3001`
- `/dashboard*` -> `dashboard-api:3000`
- `/alerts*` -> `alert-service:3000`
- `/sensors*` -> `sensor-registry-service:3000`
- `/devices*` -> `device-management-service:3000`


### Internal-only endpoints


- `anomaly-worker:3002` (health endpoint)
- `report-gen-worker:3000` (health endpoint)
- `storage-worker:3004` (health endpoint)




### Base URLs (development)




```
dashboard-api         http://dashboard-api:3000
ingestion             http://ingestion:3001
alert-service         http://alert-service:3000
sensor-registry       http://sensor-registry:3000
Device-management     http://device-management:3000
anomaly-worker        http://anomaly-worker:3002   (health endpoint only)
report-gen-worker     http://report-gen-worker:3000 (health endpoint only)
storage-worker         http://storage-worker:3004   (health endpoint only)
holmes                 (no port — access via exec)
```




> From inside holmes, services are reachable by name:
> `curl http://dashboard-api:3000/health`
>
> See [holmes/README.md](holmes/README.md) for a full tool reference.




---




## System Overview


The system ingests IoT climate readings (temperature, humidity, pressure), validates and enqueues them in Redis, and processes them with worker pipelines. `storage-worker` consumes the queue and batch-persists readings to Postgres. `anomaly-worker` consumes readings, fetches threshold metadata from `sensor-registry-service`, detects violations, and publishes alert events. `alert-service` subscribes to those events and stores alerts in the alert database. `dashboard-api` exposes read/query endpoints for latest readings and report requests. Dead-letter queues are used by workers for malformed or unprocessable messages.


## Seed Data Steps (Required Before Demo/k6)


Run these after `docker compose up` so endpoints and k6 tests have data.


### 1) Create a sensor in Sensor Registry


```bash
curl -X POST http://localhost:80/sensors/sensors \
 -H "Content-Type: application/json" \
 -d '{
   "sensor_id": "sensor-1",
   "location": "lab-a",
   "type": "climate",
   "min_temp": 15,
   "max_temp": 30,
   "min_humidity": 20,
   "max_humidity": 70,
   "min_pressure": 980,
   "max_pressure": 1040
 }'
```


### 2) Push at least one reading through ingestion


```bash
curl -X POST http://localhost:80/readings/sensor \
 -H "Content-Type: application/json" \
 -d '{
   "sensor_id": "sensor-1",
   "timestamp": "2026-05-04T18:00:00Z",
   "temperature": 24.1,
   "pressure": 1012.4,
   "humidity": 45.8
 }'
```


### 3) Optional: create a device record


```bash
curl -X POST http://localhost:80/devices/devices/register \
 -H "Content-Type: application/json" \
 -d '{
   "deviceId": "device-001",
   "sensorId": "sensor-1",
   "status": "active",
   "idempotencyKey": "seed-key-001"
 }'
```










---




## API Reference












<!--
  Document every endpoint for every service.
  Follow the format described in the project documentation: compact code block notation, then an example curl and an example response. Add a level-2 heading per service, level-3 per endpoint.
-->




—




### Dashboard API Service


### POST /reports/request
```
POST /reports/request




  Publishes report generation requests for the report generation worker to read.




  Required Parameters:
    sensor_id INT
    start_time TIMESTAMP
    end_time TIMESTAMP




  Responses:
    400  Validation failed, missing or invalid fields
    202  Successful report request submission
    500  Failed to submit report request
```




**Example request:**




```bash
curl -X POST http://localhost:80/dashboard/reports/request \
 -H "Content-Type: application/json" \
 -d '{
   "sensor_id": "sensor-1",
   "start_time": "2026-05-04T17:00:00Z",
   "end_time": "2026-05-04T18:30:00Z"
 }'
```




**Example response (202):**




```json
{
  "status": "queued",
  "request_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "queue": "report-gen:queue",
  "request": {
	"request_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
	“sensor_id”: “sensor-1”,
	"start_time": "2026-05-04T17:00:00Z",
   	"end_time": "2026-05-04T18:30:00Z"
  }
}
```




### GET /latest-readings/:sensor_id




```
GET /latest-readings/:sensor_id




  Gets latest reading related to a sensor from its sensor_id, will check for cached result.




  Required Parameters:
    sensor_id INT




  Responses:
    404  No Readings Found
    200  Reading Info Returned
```




**Example request:**




```bash
curl http://dashboard-api:3000/latest-readings/sensor_one
```




**Example response (201):**




```json
{
  "readingId": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-04-09T15:10:00Z",
  "sensorId": "sensor-abc-001",
  "temperature": 22.5,
  "pressure": 1013.2,
  "humidity": 48.7
}
```




### POST /dashboard




```
POST /dashboard




  Adds a new sensor reading into the sensor registry DB.




  Required Parameters:
    sensor_id INT
    timestamp STRING
    temperature DOUBLE
    pressure DOUBLE
    humidity DOUBLE




  Responses:
    400  Validation failed, invalid ID and/or missing input fields
    201  Successful sensor reading submission
    500  Failed to submit sensor reading
```




**Example request:**




```bash
curl -X POST http://localhost:80/dashboard/dashboard \
 -H "Content-Type: application/json" \
 -d '{
   "sensor_id": "sensor-1",
   "timestamp": "2026-05-04T18:05:00Z",
   "temperature": 23.9,
   "pressure": 1012.0,
   "humidity": 46.1
 }'
```




**Example response (201):**




```json
{
  "status": "healthy",
  "reading_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479"
}
```




### GET /health




```
GET /health




  Returns the health status of this service and its dependencies.




  Responses:
    200  Service and all dependencies healthy
    503  One or more dependencies unreachable
```




**Example request:**




```bash
curl http://dashboard-api:3000/health
```




**Example response (200):**




```json
{
  "status": "healthy",
  "db": "ok",
  "redis": "ok"
}
```




**Example response (503):**




```json
{
  "status": "unhealthy",
  "db": "ok",
  "redis": "error: connection refused"
}
```




---




### Ingestion Service




### POST /sensor




```
POST /sensor




  Publishes validated sensor reading data for redis subscribers, avoids publishing any duplicate data based on id and timestamp




  Required Parameters:
    sensor_id INT
    timestamp TIMESTAMPTZ
    temperature DOUBLE
    pressure DOUBLE
    humidity DOUBLE




  Responses:
    400  Missing sensor_id
    200  Duplicate Data, nothing is published
    202  Successfully returned sensor information
    500  Failure to retrieve sensor information
```




**Example request:**




```bash
curl http://ingestion:3001/sensor
```




**Example response (200):**




```json
{
  "readingId": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-04-09T15:10:00Z",
  "sensorId": "sensor-abc-001",
  "temperature": 22.5,
  "pressure": 1013.2,
  "humidity": 48.7
}
```




### GET /data




```
GET /data




  Gets all sensor data from a sensor, given its sensor_id




  Required Parameters:
    sensor_id STRING




  Responses:
    400  Missing sensor_id
    200  Successfully returned sensor information
    500  Failure to retrieve sensor information
```




**Example request:**




```bash
curl http://ingestion:3001/data?sensor_id=1
```




**Example response (200):**




```json
{
  "sensor_id": "sensor-7a8b9c",
  "created_at": "2024-05-20T14:30:00Z",
  "updated_at": "2024-05-20T14:30:00Z",
  "location": "Warehouse Alpha - Sector 4",
  "sensor_type": "climate_monitor",
  "min_temp": 18.5,
  "max_temp": 26.0,
  "min_humidity": 35.0,
  "max_humidity": 60.5,
  "min_pressure": 1005.2,
  "max_pressure": 1022.8
}
```




### GET /health




```
GET /health




  Returns the health status of this service and its dependencies.




  Responses:
    200  Service and all dependencies healthy
    503  One or more dependencies unreachable
```




**Example request:**




```bash
curl http://ingestion:3001/health
```




**Example response (200):**




```json
{
  "status": "healthy",
  "db": "ok",
  "redis": "ok"
}
```




**Example response (503):**




```json
{
  "status": "unhealthy",
  "db": "ok",
  "redis": "error: connection refused"
}
```


---


### Sensor Registry Service




### GET /sensors/:id




```
GET /sensors/:id




  Returns sensor metadata and threshold information for a given sensor.




  Required Parameters:
    id STRING — the sensor ID




  Responses:
    200  Successfully returned sensor information
```




**Example request:**




```bash
curl http://sensor-registry-service:3000/sensors/sensor-042
```




**Example response (200):**




```json
{
  "status": 200,
  "sensor_id": "sensor-042",
  "location": "lab",
  "threshold": 50
}
```


### POST /sensors




```
POST /sensors


  Registers a new sensor and its threshold configuration.




  Required Parameters:
    sensor_id STRING
    location STRING
    type STRING
    min_temp DOUBLE
    max_temp DOUBLE
    min_humidity DOUBLE
    max_humidity DOUBLE
    min_pressure DOUBLE
    max_pressure DOUBLE




  Responses:
    201  Sensor created
    400  Invalid input
    500  Failed to create sensor
```




**Example request:**




```bash
```bash
curl -X POST http://sensor-registry-service:3000/sensors \
-H "Content-Type: application/json" \
-d '{
"sensor_id": "sensor-1",
"location": "lab-a",
"type": "climate",
"min_temp": 15,
"max_temp": 30,
"min_humidity": 20,
"max_humidity": 70,
"min_pressure": 980,
"max_pressure": 1040
}'
```




**Example response (201):**




```json
{
"sensor_id": "sensor-1",
"location": "lab-a",
"type": "climate",
"min_temp": 15,
"max_temp": 30,
"min_humidity": 20,
"max_humidity": 70,
"min_pressure": 980,
"max_pressure": 1040
}
```


### GET /health




```
GET /health




  Returns the health status of this service and its dependencies.




  Responses:
    200  Service and all dependencies healthy
    503  One or more dependencies unreachable
```




**Example request:**




```bash
curl http://sensor-registry:3000/health
```




**Example response (200):**




```json
{
  "status": "healthy",
  "db": "ok",
  "redis": "ok"
}
```




**Example response (503):**




```json
{
  "status": "unhealthy",
  "db": "ok",
  "redis": "error: connection refused"
}
```




---




### Alert Service




### GET /alerts




```
GET /alerts




  Returns all alerts currently stored in the Alert DB.




  Responses:
    200  Successfully returned alerts
```




**Example request:**




```bash
curl http://alert-service:3000/alerts
```




**Example response (200):**




```json
[
  {
    "alert_id": "550e8400-e29b-41d4-a716-446655440000",
    "sensor_id": "sensor-001",
    "message": "Temperature exceeded maximum threshold",
    "timestamp": "2026-04-28T10:01:00Z",
    "reading_value": 105.3,
    "alert_type": "temperature"
  },
  {
    "alert_id": "660e8400-e29b-41d4-a716-446655440111",
    "sensor_id": "sensor-002",
    "message": "Humidity below minimum threshold",
    "timestamp": "2026-04-28T09:58:00Z",
    "reading_value": 20.1,
    "alert_type": "humidity"
  }
]


```
### GET /health




```
GET /health




  Returns the health status of this service and its dependencies.




  Responses:
    200  Service and all dependencies healthy
    503  One or more dependencies unreachable
```




**Example request:**




```bash
curl http://alert-service:3000/health
```




**Example response (200):**




```json
{
  "status": "healthy",
  "db": "ok",
  "redis": "ok"
}
```




**Example response (503):**




```json
{
  "status": "unhealthy",
  "db": "ok",
  "redis": "error: connection refused"
}
```


---






### Device Management Service




### POST /devices/register




```
POST /devices/register




  Registers a new device. Idempotent — duplicate requests with the
  same idempotency key return the original record without creating a duplicate.




  Required fields:
    deviceId STRING
    status STRING
    idempotencyKey STRING




  Optional fields:
    sensorId STRING
    version STRING
    metadata OBJECT




  Responses:
    201  Device successfully registered
    200  Duplicate request — original record returned
    400  Missing required fields
    409  Constraint violation (e.g. sensor_id already in use)
    500  Internal server error
```




**Example request:**




```bash
curl -X POST http://device-management-service:3000/devices/register \
  -H "Content-Type: application/json" \
  -d '{"deviceId":"device-001","sensorId":"sensor-042","status":"active","idempotencyKey":"key-abc-123"}'
```




**Example response (201):**




```json
{
  "device_id": "device-001",
  "sensor_id": "sensor-042",
  "status": "active",
  "version": null,
  "registered_at": "2026-04-20T20:27:26.567Z",
  "last_seen_at": null,
  "metadata": {},
  "idempotency_key": "key-abc-123"
}
```


### GET /devices/:id


GET /devices/:id


Returns device information for a given device ID.
Required Parameters:
 id STRING
Responses:
 200 Device found
 404 Device not found


**Example request:**




```bash
curl http://device-management-service:3000/devices/device-001
```

Example response (200):


{
  "device_id": "device-001",
  "sensor_id": "sensor-1",
  "status": "active",
  "version": "1.0.0",
  "registered_at": "2026-05-04T18:00:00Z",
  "last_seen_at": "2026-05-04T18:05:00Z"
}


### POST /devices/:id/maintenance
POST /devices/:id/maintenance
Schedules a maintenance window for a device.
Required Parameters:
 id STRING
Body Parameters:
 startTime STRING
 endTime STRING
 reason STRING
Responses:
 200 Maintenance scheduled
 400 Invalid input


**Example request:**
```bash
curl -X POST http://device-management-service:3000/devices/device-001/maintenance \
 -H "Content-Type: application/json" \
 -d '{
   "startTime": "2026-05-05T10:00:00Z",
   "endTime": "2026-05-05T12:00:00Z",
   "reason": "calibration"
 }'
Example response (200):
{
  "status": "scheduled",
  "device_id": "device-001"
}
```


### POST /devices/:id/firmware




POST /devices/:id/firmware
Updates the firmware version of a device.
Required Parameters:
 id STRING
Body Parameters:
 version STRING
Responses:
 200 Firmware updated
 400 Invalid input


**Example request:**




```bash
curl -X POST http://device-management-service:3000/devices/device-001/firmware \
 -H "Content-Type: application/json" \
 -d '{
   "version": "1.0.2"
 }'
 ```


Example response (200):

```json
{
  "status": "updated",
  "device_id": "device-001",
  "version": "1.0.2"
}
```




---




### Report Generation Worker




### GET /health




```
GET /health




  Returns the health status of the report generation worker.




  Responses:
    200  Worker healthy, Redis reachable
    503  Redis unreachable
```




**Example request:**




```bash
curl http://report-gen-worker:3000/health
```




**Example response (200):**




```json
{
  "status": "healthy",
  "service": "report-gen-worker",
  "checked_at": "2026-04-20T20:27:26.567Z",
  "dependencies": {
    "redis": {
      "status": "healthy",
      "latency_ms": 1
    }
  },
  "metrics": {
    "queue_depth": 0,
    "dlq_depth": 0,
    "last_successfully_processed_at": null,
    "jobs_processed_count": 0
  }
}
```




### Storage Worker




### GET /health




```
GET /health




  Returns the health status of the storage worker.




  Responses:
    200  Worker healthy, Redis reachable
    503  Redis unreachable
```




**Example request:**




```bash
curl http://storage-worker:3004/health
```




**Example response (200):**




```json
{
  "status": "healthy",
  "redis": "ok",
  "depth": 0,
  "dlq_depth": 0,
  "last_job_at": "2026-04-20T20:27:26.567Z",
  "jobs_processed": 10
}
```




**Example response (503):**




```json
{
  "status": "unhealthy",
  "redis": "error: connection refused",
  "depth": 0,
  "dlq_depth": 0,
  "last_job_at": null,
  "jobs_processed": 0
}
```








### Anomaly Worker




### GET /health




```
GET /health




  Returns the health status of the anomaly worker and its dependencies.




  Responses:
    200  Worker healthy, Redis reachable
    503  Redis unreachable
```




**Example request:**




```bash
curl http://anomaly-worker:3002/health
```




**Example response (200):**




```json
{
  "status": "healthy",
  "redis": "ok",
  "depth": 0,
  "dlq_depth": 0,
  "last_job_at": "2026-04-20T20:27:26.567Z",
  "jobs_processed": 10
}
```




**Example response (503):**




```json
{
  "status": "unhealthy",
  "redis": "error: connection refused",
  "depth": 0,
  "dlq_depth": 0,
  "last_job_at": null,
  "jobs_processed": 0
}
```


–--


## k6 Tests (Sprint 4)


### Scaling comparison (`k6/sprint-4-scale.js`)


```bash
# from host (if k6 installed)
k6 run --env BASE_URL=http://localhost:80 --env SCALE=single k6/sprint-4-scale.js
k6 run --env BASE_URL=http://localhost:80 --env SCALE=replicated k6/sprint-4-scale.js


# from Holmes
docker compose exec holmes bash
k6 run --env SCALE=single /workspace/k6/sprint-4-scale.js
k6 run --env SCALE=replicated /workspace/k6/sprint-4-scale.js
```


### Replica failure test (`k6/sprint-4-replica.js`)


```bash
# from host (if k6 installed)
k6 run --env BASE_URL=http://localhost:80 k6/sprint-4-replica.js


# from Holmes
docker compose exec holmes bash
k6 run /workspace/k6/sprint-4-replica.js
```


### Replica failure drill during test


```bash
# list replica containers
docker compose ps


# stop one replica during sustained phase
docker stop <container-id>


# verify survivors remain healthy
docker compose ps


# restore desired replica count
docker compose up --scale dashboard-api=3 -d
```




## Sprint History

| Sprint | Tag        | Plan                                              | Report                                    |
| ------ | ---------- | ------------------------------------------------- | ----------------------------------------- |
| 1      | `sprint-1` | [SPRINT-1-PLAN.md](sprint-plans/SPRINT-1-PLAN.md) | [SPRINT-1.md](sprint-reports/SPRINT-1.md) |
| 2      | `sprint-2` | [SPRINT-2-PLAN.md](sprint-plans/SPRINT-2-PLAN.md) | [SPRINT-2.md](sprint-reports/SPRINT-2.md) |
| 3      | `sprint-3` | [SPRINT-3-PLAN.md](sprint-plans/SPRINT-3-PLAN.md) | [SPRINT-3.md](sprint-reports/SPRINT-3.md) |
| 4      | `sprint-4` | [SPRINT-4-PLAN.md](sprint-plans/SPRINT-4-PLAN.md) | [SPRINT-4.md](sprint-reports/SPRINT-4.md) |
