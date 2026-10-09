# Todo Summary Assistant: DevOps Delivery Pipeline

A full-stack app (Spring Boot backend, React frontend, MySQL) that manages to-do items, summarises pending tasks with Cohere, and posts the summary to Slack.
This repository adds a production-style DevOps setup: Docker, GitHub Actions CI/CD, AWS EC2 + RDS, and Prometheus + Grafana monitoring.

## Architecture (short)

GitHub push -> GitHub Actions (test -> build images tagged by commit SHA -> push to Docker Hub -> deploy to EC2 over SSH -> health check).
On EC2: `frontend`, `backend`, `prometheus`, `node-exporter`, `grafana` containers on one Docker network (`appnet`). The backend talks to a private MySQL RDS instance.
Diagram: `aws/architecture-diagram.png`. AWS details: `aws/aws-setup.md`.

## Repository layout

| Path | Purpose |
|------|---------|
| `Backend/todo-summary-assistant/` | Spring Boot app, `Dockerfile`, `.dockerignore` |
| `Frontend/todo/` | React app, `Dockerfile`, `.dockerignore` |
| `.github/workflows/ci-cd.yml` | CI/CD pipeline |
| `aws/` | Architecture diagram and AWS setup guide |
| `monitoring/` | `prometheus.yml`, `alert-rules.yml`, `grafana-dashboard.json` |
| `screenshots/` | Grafana dashboard, Prometheus targets and alerts |
| `docker-compose.yml` | Local run (app + monitoring) |
| `.env.example` | All environment variables (no real values) |
| `FAILURE_AND_ROLLBACK.md` | Failure scenarios and recovery |
| `MONITORING_AND_OPERATIONS.md` | Metrics, logs and alerting design |

## 4.1 Run locally

**Requirements:** Java 17, Maven (wrapper included), Node.js 18+, MySQL 8, Docker (optional).

1. Clone the repo and copy the env template:
   ```bash
   git clone https://github.com/sinchanas034/TodoSummaryAssistant.git
   cd TodoSummaryAssistant
   cp .env.example .env      # then fill in your own values
   ```
2. Backend:
   ```bash
   cd Backend/todo-summary-assistant
   ./mvnw spring-boot:run
   ```
3. Frontend:
   ```bash
   cd Frontend/todo
   npm install
   npm start
   ```

### Environment variables

| Variable | Meaning |
|----------|---------|
| `DB_URL` | JDBC URL of the MySQL database |
| `DB_USER` / `DB_PASSWORD` | Database credentials |
| `COHERE_API_KEY` | Cohere API key |
| `SLACK_WEBHOOK_URL` | Slack incoming-webhook URL |
| CORS allowed origins variable | Browser origins allowed to call the API (see `.env.example` for the exact name) |
| `REACT_APP_API_URL` | Backend URL, passed to the frontend as a build argument |

Real values are never committed. On the server they live in `/home/ubuntu/app/.env` (not in Git). In CI they come from GitHub Secrets.

## 4.2 Docker design choices

- **Multi-stage builds:** build tools (Maven, Node) stay in the build stage; the final image holds only the runtime. This keeps images small.
- **Non-root:** the backend runs as `appuser` and the frontend as `nginx` (verified with `docker exec <container> whoami`).
- **Configuration through environment variables:** no secrets or environment-specific values baked into images.
- **`.dockerignore`:** excludes build output, `node_modules`, `.git` and local env files.
- **Frontend:** static files served by nginx on port 8080 inside the container (mapped to 3000 on the host).

## 4.3 CI/CD pipeline (`.github/workflows/ci-cd.yml`)

Triggers: push to `main` and pull requests.

| Stage | What it does |
|-------|--------------|
| `test` | Starts a MySQL service container, runs `./mvnw test` with dummy values. Fails fast on any error |
| `build-and-push` | Logs in to Docker Hub, builds backend and frontend images, tags each with the commit SHA and `latest`, pushes them. Runs only on push to `main` |
| `deploy` | SSH to EC2, pulls the SHA-tagged images, replaces the containers on the `appnet` network, then polls `/actuator/health` for up to ~200 s. If it never reports `UP`, the job prints logs and fails |

No secrets are in the file. These GitHub Secrets are used: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY`.

## 4.4 / 4.5 AWS and monitoring

See `aws/aws-setup.md` and `MONITORING_AND_OPERATIONS.md`.
Prometheus scrapes the backend (`/actuator/prometheus`) and Node Exporter. Grafana shows request rate, error rate, latency, uptime, restarts (custom dashboard) and CPU, memory, disk (Node Exporter Full, dashboard ID 1860). Four alerts are defined: ServiceDown, HighCPU, HighErrorRate, LowDisk.

## Changes to application code (DevOps enablement only)

1. Added Spring Boot Actuator and Micrometer Prometheus registry to expose `/actuator/health` and `/actuator/prometheus`.
2. Moved DB URL, DB credentials, Cohere key and Slack webhook from `application.properties` to environment variables.
3. CORS allowed origins are read from an environment variable.

No business logic was changed.

## Assumptions

- A single EC2 instance runs the app and the monitoring stack (small, free-tier friendly footprint).
- The browser calls the backend directly on port 8080, so that port is reachable from the internet.
- Grafana data is not persisted in a volume; the dashboard is stored as JSON in `monitoring/` and can be re-imported.
- Docker Hub is used as the registry.

## Known limitations

- HTTP only (no TLS/domain). Production would add HTTPS through a load balancer or reverse proxy.
- Single EC2 instance, so no high availability.

## Monitoring setup on the server

Run once on the EC2 instance (the deploy pipeline does not start these containers):

```bash
mkdir -p ~/monitoring
# copy monitoring/prometheus.yml and monitoring/alert-rules.yml from this repo into ~/monitoring

docker network create appnet 2>/dev/null || true

docker run -d --name node-exporter --network appnet --restart unless-stopped \
  -v /:/host:ro,rslave prom/node-exporter --path.rootfs=/host

docker run -d --name prometheus --network appnet --restart unless-stopped \
  -p 9090:9090 \
  -v ~/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v ~/monitoring/alert-rules.yml:/etc/prometheus/alert-rules.yml \
  prom/prometheus

docker run -d --name grafana --network appnet --restart unless-stopped \
  -p 3001:3000 grafana/grafana
```

Then open Grafana on port 3001, add the Prometheus data source `http://prometheus:9090`, and import `monitoring/grafana-dashboard.json` and dashboard ID 1860.
