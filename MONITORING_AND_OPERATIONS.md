# Monitoring and Operations

## Stack

Prometheus scrapes every 15 seconds: the Spring Boot backend (`/actuator/prometheus`) and Node Exporter (host metrics). Grafana reads from Prometheus. All run as containers on `appnet`. Config: `monitoring/prometheus.yml`, `monitoring/alert-rules.yml`, `monitoring/grafana-dashboard.json`. Screenshots are in `screenshots/`.

## Which metrics I monitor and why

| Metric | Why |
|--------|-----|
| Request rate (`http_server_requests_seconds_count`) | Shows traffic, and a sudden drop to zero means users cannot reach the app |
| Error rate (5xx share of requests) | The clearest signal of user-facing failure |
| Latency (`_sum / _count`) | Slow responses are felt before errors appear |
| Uptime (`process_uptime_seconds`) | Confirms how long the process has been stable |
| Restarts (`changes(process_start_time_seconds[1h])`) | Detects crash loops without cAdvisor |
| `up{job="app"}` | Is the backend scrapeable at all |
| CPU, memory, disk (Node Exporter) | Capacity and resource exhaustion on the single host |

Possible additions: JVM heap usage and database connection pool metrics.

## Critical logs

- Backend container logs (`docker logs backend`): startup errors, database connection errors, uncaught exceptions, failed Cohere or Slack calls.
- Pipeline logs in GitHub Actions: failed tests, failed deploys, failed health checks.
- Frontend (nginx) logs: 5xx responses and unexpected traffic spikes.
- System: SSH login attempts and Docker daemon logs on the host.
- AWS: RDS events and CloudTrail for IAM and security group changes.

Production would ship these logs to a central place (CloudWatch Logs or Loki), so they survive the loss of a container or instance.

## Alerts that matter

| Alert | Condition | Severity |
|-------|-----------|----------|
| ServiceDown | `up{job="app"} == 0` for 1 min | critical |
| HighErrorRate | 5xx ratio above 5% for 5 min | critical |
| HighCPU | CPU above 80% for 5 min | warning |
| LowDisk | Root filesystem under 15% free for 10 min | warning |

Each alert has a `for:` duration so that short spikes do not page anyone. The error-rate rule uses `clamp_min` so that an idle app (no requests) does not divide by zero.

## What should not alert (to avoid noise)

- Single 4xx responses (user mistakes) and short latency spikes.
- A brief CPU peak during a deploy.
- Container restarts during a planned deployment.
- Memory use that is high but stable, since the JVM and Linux cache use memory by design.
- Small side mounts (`/boot`, `/etc/hosts`), which is why the disk rule is limited to `mountpoint="/"`.

Informational items go on dashboards, not to a pager.

## How issues are detected early

1. The pipeline health check catches a bad deploy within minutes.
2. `ServiceDown` fires within about 1 minute of an outage.
3. Dashboards show trends (latency creeping up, disk filling) before alerts trigger.
4. Restart and uptime panels reveal crash loops.
5. Tests run on every push and pull request.
6. Next step for production: route alerts to Slack or email through Alertmanager or Grafana contact points, and add an external uptime check against the public URL, since monitoring on the same host cannot report that host being down.
