# Failure and Rollback

Deployment facts used below: images are tagged with the commit SHA, the pipeline replaces containers over SSH, and a health check on `/actuator/health` runs after each deploy. Containers use `--restart unless-stopped`.

## 1. A faulty version is deployed. How do I roll back?

Every build is kept in Docker Hub under its commit SHA, so rollback means running an older tag.

- **Fast, through Git:** `git revert <bad-commit>` and push. The pipeline rebuilds and redeploys the previous behaviour.
- **Fastest, on the server:** SSH in, then:
  ```bash
  docker rm -f backend frontend
  docker run -d --name backend  --network appnet --restart unless-stopped --env-file /home/ubuntu/app/.env -p 8080:8080 <user>/todo-backend:<previous-sha>
  docker run -d --name frontend --network appnet --restart unless-stopped -p 3000:8080 <user>/todo-frontend:<previous-sha>
  ```
- Because the health check fails the pipeline when the new backend is not `UP`, most bad releases are noticed at deploy time.
- Keep database changes backward compatible, so the previous version can still run against the current schema.

## 2. The application crashes after deployment. What happens?

- Docker restarts the container automatically (`--restart unless-stopped`).
- If the backend never becomes healthy, the pipeline's health check fails after about 200 seconds, prints the last 30 log lines and marks the run red.
- Prometheus marks the `app` target down, and the `ServiceDown` alert fires after 1 minute.
- Action: read `docker logs backend`, then roll back as in scenario 1 while debugging.

## 3. The CI/CD tool is unavailable. Can I still deploy?

Yes. The images already exist in Docker Hub, and the server only needs Docker:

1. SSH to EC2.
2. `docker pull <user>/todo-backend:<sha>` and the same for the frontend.
3. Recreate the containers with the `docker run` commands in scenario 1.

If Docker Hub is also down, build the image on a laptop or on the server from the Git repository. Keep the deploy commands in this document so they do not depend on the CI tool.

## 4. Secrets are leaked. What steps do I take?

1. **Rotate first**, clean up second. Revoke the leaked item: Cohere API key, Slack webhook, DB password, Docker Hub token, SSH key, or AWS credentials.
2. Update the new values in `/home/ubuntu/app/.env` and GitHub Secrets, then redeploy.
3. Change the RDS master password and restart the backend.
4. Remove the secret from the repository and from Git history (`git filter-repo` or BFG). Treat it as compromised anyway, because the repository may have been cloned.
5. Review logs for misuse (AWS CloudTrail, Slack, Cohere usage).
6. Prevention: `.gitignore` for `.env`, secret scanning in GitHub, and no real values in `.env.example`.

## 5. The EC2 instance fails. How do I recover?

- **State:** the app is stateless and the data lives in RDS, so losing the instance does not lose data.
- **Recovery:** launch a new instance with the same security group and IAM role, install Docker, recreate `/home/ubuntu/app/.env` from a secure backup or secret store, and recreate the monitoring files from the `monitoring/` folder. Then push or re-run the pipeline, which pulls the images and deploys.
- If the host is only unhealthy, reboot it. Containers come back through the restart policy.
- Update the `EC2_HOST` secret if the public IP changes (an Elastic IP avoids this).
- Improvement: an AMI snapshot of a prepared server, or Terraform, to shorten the recovery time.
- Grafana data is not persisted. The dashboard is restored by importing `monitoring/grafana-dashboard.json`.

## 6. The RDS database becomes unavailable. What is the impact and recovery plan?

- **Impact:** the backend cannot read or write todos, so the API returns errors, the frontend cannot save or load, and `HighErrorRate` fires. The health endpoint may report DOWN.
- **Recovery:**
  - Check the RDS console and events. A reboot or failover often resolves a transient problem.
  - Automated backups with point-in-time recovery: restore to a new instance, then point `DB_URL` in `.env` at it and restart the backend.
  - Manual snapshots before risky changes.
  - **Multi-AZ** keeps a standby in another availability zone and fails over automatically, normally within a few minutes. It is not enabled in this low-cost setup, but it is the production recommendation.
- The security group rule and the endpoint name stay the same when the instance is restored in place.
