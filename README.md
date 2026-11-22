# ORBITERRA + ASTRA integrated backend

This folder combines the ORBITERRA FastAPI application and ASTRA data/AI services.
The original service filenames are retained.

## Architecture

- `orbiterra_backend/main.py` — REST API, SQLite persistence, dashboard, sensor ingestion and irrigation safety controls.
- `astra/ai_service.py` — cybersecurity/decision service. Receives MQTT telemetry over mutually authenticated TLS, applies anomaly checks, signs recommendations with HMAC-SHA256, and writes a tamper-evident SHA-256 audit chain.
- `astra/mosquitto/` — MQTT broker configuration and topic ACL.
- `astra/simulator.py` — optional MQTT device simulator.
- `astra/generate_data.py` / `astra/train_model.py` — seasonal training-data generation and model training.
- `astra/verify_chain.py` — audit-chain integrity verifier.
- `orbiterra_backend/static/` — existing dashboard assets, preserved unchanged.

## Run the complete stack (Docker)

Requirements: Docker Desktop with Docker Compose.

1. Copy `.env.example` to `.env`.
2. Replace both secrets in `.env` with long random values (at least 32 characters for `ASTRA_HMAC_KEY`).
3. From this project root run:

   ```bash
   docker compose up --build
   ```

4. Open:
   - Dashboard: http://localhost:8000
   - API docs: http://localhost:8000/docs
   - Health: http://localhost:8000/api/health

The default stack starts Mosquitto, the API and ASTRA AI/security service together.
To also start the test device simulator:

```bash
docker compose --profile simulation up --build
```

Elasticsearch and Kibana are optional and use the `observability` profile:

```bash
docker compose --profile observability up --build
```

## Data flow

1. The dashboard or a sensor gateway posts readings to `POST /api/sensors/ingest` with `X-Sensor-Key`.
2. The API stores the reading in SQLite and publishes a normalized telemetry message to `devices/pump-001/sensors`.
3. Mosquitto requires a trusted client certificate and enforces topic-level ACLs.
4. `ai_service.py` receives the telemetry over TLS, checks sensor anomalies, calculates an irrigation recommendation and publishes it to `devices/pump-001/commands`.
5. The API accepts the recommendation only if its HMAC signature is valid. Read it at `GET /api/security/latest-recommendation`.
6. Every AI decision or rejected telemetry event is appended to the ASTRA hash-chain audit log.

The AI service publishes **recommendations only**. It does not directly switch the pump. Pump control remains behind the existing API duration limit and hardware-disabled-by-default safeguards.

## Test sensor ingestion

PowerShell example:

```powershell
$body = @{
  soil_moisture = 32
  air_temperature = 29
  air_humidity = 61
  soil_temperature = 25
  rainfall_mm = 0
} | ConvertTo-Json

Invoke-RestMethod -Method Post `
  -Uri http://127.0.0.1:8000/api/sensors/ingest `
  -Headers @{ "X-Sensor-Key" = "YOUR_ORBITERRA_SENSOR_API_KEY" } `
  -ContentType "application/json" -Body $body
```

Then inspect:

```text
GET http://127.0.0.1:8000/api/health
GET http://127.0.0.1:8000/api/security/latest-recommendation
```

`mqtt_published` in the ingest response indicates whether the API had an active MQTT connection when it attempted to publish.

## Run data/model scripts locally

Create and activate a Python virtual environment, then install:

```bash
pip install -r orbiterra_backend/requirements.txt
```

Run from the project root:

```bash
python astra/generate_data.py
python astra/train_model.py
python astra/verify_chain.py
```

`generate_data.py` expects the regional CSVs under `astra/data/climate/`. The supplied datasets and existing trained model/data files are retained.

## TLS and secrets

- MQTT uses TLS on port `8883`, certificate authentication, CA validation and topic ACLs.
- The API uses the `pump-001` device certificate identity for its sensor publish/command subscription permissions.
- ASTRA uses its own `astra-ai` client certificate identity.
- The included certificates are development credentials from the supplied project (with a broker certificate regenerated for the Docker service DNS name). Replace/regenerate certificates and private keys from your own CA before deployment.
- The CA private signing key is intentionally not included in the merged archive. Never commit production private keys or `.env`.
- Change `ASTRA_HMAC_KEY` and `ORBITERRA_SENSOR_API_KEY` before use. The compose defaults are for local development only.
- Hardware remains disabled in Compose. Do not enable it until independent physical cutoff, emergency stop, relay isolation and fail-safe OFF behaviour have been tested.

## Existing API routes

The original ORBITERRA routes remain available: farm profile, sensor latest/ingest, dashboard, irrigation status/start/stop/history and report summary. Additional routes:

- `GET /api/security/latest-recommendation` — most recent HMAC-verified AI recommendation.
- `GET /api/health` — API and MQTT integration status.

## Notes

- Satellite/weather integrations, farmer authentication, and production hardware relay integration were not configured in either supplied project, so they are not fabricated here.
- The AI anomaly baseline and irrigation estimates remain demonstration logic; validate and calibrate against real devices and agronomic data before operational use.
