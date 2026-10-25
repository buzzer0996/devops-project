# DevOps Infrastructure and Monitoring Project

## 1. Project Overview

This project demonstrates Infrastructure as Code (IaC), automated Terraform validation, containerized monitoring, system metrics collection, and web service availability testing.

The project combines Terraform, GitHub Actions, Docker Compose, Prometheus, Grafana, Node Exporter, Blackbox Exporter, and Nginx in a local AWS-compatible testing environment.

The main objectives are to:
- Define and validate infrastructure using Terraform.
- Automate Terraform checks through GitHub Actions.
- Collect and visualize system metrics.
- Monitor web service availability.
- Test monitoring behavior during service outages and recovery.
- Maintain version-controlled configuration and project documentation.

## 2. Architecture

The local environment consists of the following components:

- **Terraform:** Defines the infrastructure configuration and validates it.
- **AWS provider:** Connects Terraform to the local AWS-compatible API endpoint.
- **GitHub Actions:** Runs automated Terraform formatting and validation checks.
- **Docker Compose:** Runs the monitoring services.
- **Nginx:** Serves the test web page inside the simulated Linux environment.
- **Node Exporter:** Exposes system metrics, including CPU, memory, and filesystem statistics.
- **Prometheus:** Collects metrics from configured targets and stores time-series data.
- **Blackbox Exporter:** Probes the Nginx HTTP endpoint and reports whether the request succeeds.
- **Grafana:** Visualizes collected metrics in the DevOps Monitoring dashboard.
- **SSH tunnels:** Forward the Nginx and Node Exporter ports to the macOS host so the monitoring containers can reach them.

### Monitoring data flow

1. Node Exporter exposes system metrics on port `9100` in the Linux environment.
2. An SSH tunnel forwards host port `19100` to Node Exporter.
3. Prometheus scrapes Node Exporter at `host.docker.internal:19100`.
4. A separate SSH tunnel exposes Nginx on host port `18080`.
5. Blackbox Exporter probes `http://host.docker.internal:18080`.
6. Prometheus collects the probe result and makes it available for queries.
7. Grafana queries Prometheus and displays monitoring metrics.

## 3. Repository Structure

```text
.
├── .github/
│   └── workflows/
│       └── terraform.yml
├── monitoring/
│   ├── docker-compose.yml
│   └── prometheus/
│       └── prometheus.yml
├── terraform/
│   ├── .terraform.lock.hcl
│   └── main.tf
├── .gitignore
└── README.md
```

## 4. Services and Local URLs

| Service | URL | Purpose |
|---|---|---|
| AWS-compatible API | `http://localhost:4566` | Local infrastructure API |
| Environment dashboard | `http://localhost:9877` | Local environment interface |
| Nginx | `http://localhost:18080` | Test web server |
| Prometheus | `http://localhost:9090` | Metrics and queries |
| Prometheus Targets | `http://localhost:9090/targets` | Check scrape target status |
| Grafana | `http://localhost:3001` | Monitoring dashboards |
| Blackbox Exporter | `http://localhost:9115` | Probe and exporter endpoints |

These addresses are for the local development environment and are not public service endpoints.

## 5. Prerequisites

- macOS
- Docker with Docker Compose support
- Terraform CLI
- Git
- A running local AWS-compatible testing environment
- SSH access to the simulated Linux environment
- Nginx and Node Exporter installed in that environment

## 6. Starting the Monitoring Stack

Run the following commands from the repository root.

Start Prometheus, Grafana, and Blackbox Exporter:

```bash
docker compose -f monitoring/docker-compose.yml up -d
```

Check container status:

```bash
docker compose -f monitoring/docker-compose.yml ps
```

View logs:

```bash
docker compose -f monitoring/docker-compose.yml logs
```

Stop the monitoring stack:

```bash
docker compose -f monitoring/docker-compose.yml down
```

The standard `down` command preserves named data volumes. Avoid adding `-v` unless you intentionally want to remove the stored Prometheus and Grafana data.

## 7. SSH Tunnels

The tunnels must remain open while Prometheus needs to reach the corresponding services.

### Nginx tunnel

Run in a dedicated macOS terminal:

```bash
ssh -i ~/devops-local-key.pem \
  -p 2200 \
  -o IdentitiesOnly=yes \
  -N -L 18080:127.0.0.1:8080 \
  root@127.0.0.1
```

Open `http://localhost:18080` to access Nginx.

### Node Exporter tunnel

Run in another dedicated macOS terminal:

```bash
ssh -i ~/devops-local-key.pem \
  -p 2200 \
  -o IdentitiesOnly=yes \
  -N \
  -L 19100:127.0.0.1:9100 \
  root@127.0.0.1
```

Prometheus reaches Node Exporter through `host.docker.internal:19100`.

The private key is referenced locally and must not be committed to the repository.

## 8. Terraform Configuration and Validation

The Terraform configuration is located in `terraform/main.tf`. The provider is configured for a local AWS-compatible API endpoint rather than a public AWS endpoint.

Run formatting checks:

```bash
cd terraform
terraform fmt -check
```

Validate the configuration:

```bash
terraform validate
```

Inspect proposed infrastructure changes:

```bash
terraform plan
```

Review the plan before performing any infrastructure-changing operation. Validation does not create, modify, or destroy infrastructure.

The configuration has been checked successfully with `terraform fmt -check` and `terraform validate`.

## 9. Continuous Integration

The workflow in `.github/workflows/terraform.yml` automates Terraform checks through GitHub Actions.

The workflow checks Terraform formatting, initializes Terraform without configuring a remote backend, and validates the configuration.

This provides an automated check of the Terraform configuration when the workflow is triggered by repository changes.

## 10. Prometheus Configuration

The Prometheus configuration is located at:

`monitoring/prometheus/prometheus.yml`

The configured scrape jobs are:

- **prometheus:** Monitors the Prometheus service itself.
- **node-exporter:** Collects system metrics through the Node Exporter SSH tunnel.
- **nginx-blackbox:** Checks Nginx availability using Blackbox Exporter and the `http_2xx` probe module.

The global scrape interval is 15 seconds.

Open `http://localhost:9090/targets` to inspect the status of configured targets.

## 11. Grafana Dashboard

Grafana is available at `http://localhost:3001`.

The **DevOps Monitoring** dashboard contains panels for:

1. Prometheus target status.
2. Available memory in GiB.
3. CPU usage percentage.
4. Filesystem usage percentage.

The Prometheus data source is configured with the internal Docker Compose service URL:

`http://prometheus:9090`

### Example PromQL queries

Target status:

```promql
up
```

Available memory in GiB:

```promql
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

CPU usage percentage:

```promql
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
```

Filesystem usage percentage:

```promql
100 * (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes)
```

Nginx probe success:

```promql
probe_success{job="nginx-blackbox"}
```

A `probe_success` value of `1` means the probe succeeded; `0` means it failed.

## 12. Availability and Recovery Test

The Nginx availability test verifies that monitoring detects an outage and observes recovery.

Test procedure:

1. Confirm that Nginx is accessible at `http://localhost:18080`.
2. Query `probe_success{job="nginx-blackbox"}` in Prometheus.
3. Stop Nginx in the simulated Linux environment.
4. Confirm that the probe reports `0` after the next probe.
5. Restart Nginx.
6. Confirm that the probe returns to `1` after the service becomes reachable and the next probe completes.

The test was completed successfully: the probe changed from `1` to `0` during the outage and returned to `1` after Nginx restarted.

## 13. Security and Repository Hygiene

The `.gitignore` file excludes common local Terraform state files, private keys, and local variable files.

Security practices for this project include:

- Do not commit private SSH keys.
- Do not commit Terraform state files or files containing secrets.
- Do not publish access tokens, passwords, or real cloud credentials.
- Use test credentials only in the intended local testing environment.
- Review changes before committing and pushing them to GitHub.

## 14. Environment Limitations

- The project runs in a local simulation and does not represent a deployment to real AWS infrastructure.
- Service URLs use localhost and are intended for local access.
- Prometheus depends on the SSH tunnels to reach Nginx and Node Exporter.
- The simulated Linux environment does not run a normal `systemd` process. Services started manually may need to be restarted if the environment is recreated.
- Monitoring data collection depends on Docker, the local environment, the monitored services, and the required tunnels being available.

## 15. Useful Commands

From the repository root:

Check Git status:

```bash
git status -sb
```

Start monitoring:

```bash
docker compose -f monitoring/docker-compose.yml up -d
```

Check monitoring containers:

```bash
docker compose -f monitoring/docker-compose.yml ps
```

Validate Terraform:

```bash
cd terraform
terraform fmt -check
terraform validate
```

Return to the repository root:

```bash
cd ..
```

---

This project demonstrates a local DevOps workflow combining infrastructure configuration, automated validation, monitoring, observability, availability testing, and version control.
