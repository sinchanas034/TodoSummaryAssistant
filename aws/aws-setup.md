# AWS Setup

## Overview

| Component | Setting |
|-----------|---------|
| Region | us-east-1 (N. Virginia) |
| EC2 | t3.micro, Ubuntu, Docker installed |
| RDS | MySQL 8, db.t4g.micro, **not publicly accessible** |
| Registry | Docker Hub |
| Deploy method | GitHub Actions over SSH |

## Networking

- Default VPC. EC2 runs in a public subnet with a public IP.
- RDS runs in the same VPC, with **Publicly accessible = No**, so it has no route from the internet.

## Security groups

### `todo-ec2-sg` (EC2)

| Port | Source | Why |
|------|--------|-----|
| 22 | 0.0.0.0/0 (key-only login) | SSH for the CI deploy step and admin* |
| 3000 | 0.0.0.0/0 | Public website (frontend) |
| 8080 | 0.0.0.0/0 | Backend API, called by the visitor's browser |
| 3001 | My IP | Grafana |
| 9090 | My IP | Prometheus |

\*Port 22 is open to `0.0.0.0/0` because GitHub-hosted runners have changing IP addresses, so the pipeline cannot connect from a fixed address. Login requires the private SSH key, which is stored only in GitHub Secrets (password login is not used). This is a known trade-off. A production setup would use AWS SSM Session Manager (no open SSH port) or a self-hosted runner with a fixed IP, and this rule would be closed or restricted to that IP.

### RDS security group

| Port | Source | Why |
|------|--------|-----|
| 3306 | `todo-ec2-sg` (security group, not an IP range) | Only the EC2 instance can reach the database |

## IAM

- The EC2 instance has an **IAM role** attached (instance profile), so no AWS access keys are stored on the server or in the repository.
- The instance profile role is `todo-ec2-role`. It uses the AWS managed policy `AmazonSSMManagedInstanceCore` (Systems Manager access only). It does not use `AdministratorAccess`. No AWS access keys are stored on the server or in the repository.

## Secrets and configuration

| Secret | Where it lives |
|--------|----------------|
| DB URL, DB user and password, Cohere key, Slack webhook | `/home/ubuntu/app/.env` on EC2 (not in Git), passed to the container with `--env-file` |
| Docker Hub token, SSH key, EC2 host and user | GitHub Actions Secrets |

## Server layout

```
/home/ubuntu/app/.env          application secrets
/home/ubuntu/monitoring/       prometheus.yml, alert-rules.yml
Docker network: appnet         frontend, backend, prometheus, node-exporter, grafana
```

## Rebuilding from scratch

1. Create the security groups, the RDS MySQL instance (private) and the EC2 instance with the IAM role.
2. Install Docker on EC2 and create `/home/ubuntu/app/.env` from `.env.example`.
3. Copy `monitoring/prometheus.yml` and `monitoring/alert-rules.yml` to `~/monitoring/`, then start Node Exporter, Prometheus and Grafana on `appnet` (commands are in the README).
4. Add the GitHub Secrets and push to `main`. The pipeline deploys the app.
5. In Grafana, add the Prometheus data source (`http://prometheus:9090`) and import `monitoring/grafana-dashboard.json` and dashboard 1860.

## Cost control

Use free-tier instance types. Delete the EC2 instance, RDS instance (take a final snapshot only if needed), security groups and Elastic IPs after the review.
