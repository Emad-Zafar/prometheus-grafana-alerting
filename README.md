# Metrics and alerting with Prometheus, Alertmanager and Grafana

A self-contained metrics stack for Docker hosts. Prometheus collects metrics,
Alertmanager sends notifications when something crosses a threshold, and Grafana
draws the graphs. Container metrics come from cAdvisor and host metrics from
node-exporter, so you get useful numbers before your app exposes anything at
all.

This is not your application. It runs alongside your containers, scrapes them,
and alerts on them.

## What you get

- CPU, memory, disk and network per container out of the box (cAdvisor)
- Host level metrics such as load, memory and disk usage (node-exporter)
- Alert rules for high CPU, high memory, and the pipeline going down
- Notifications to Slack, email or AWS SNS when a rule fires
- Grafana ready to graph any of it

## Which setups this fits

Use this when:

- You run Docker containers on one or a few hosts and want to know when one is
  hot, leaking memory, or down.
- You want alerts without paying for a hosted monitoring product.
- Your services can expose Prometheus style metrics, or you are happy starting
  with container and host metrics only.

This is a single host or small fleet design. Each host runs its own cAdvisor and
node-exporter, and one Prometheus scrapes them. For a large dynamic fleet you
would move to service discovery or a hosted metrics backend, but the config here
is the same shape you would start from.

## What's inside

```
docker-compose.yml              five services
prometheus/prometheus.yml       what to scrape
prometheus/alerts.yml           the alert rules
alertmanager/alertmanager.yml   where alerts go
grafana/provisioning/...        Prometheus wired in as a data source
.env.example                    copy to .env and edit
```

## Requirements

- Docker and Docker Compose
- Ports 3000 (Grafana), 9090 (Prometheus) and 9093 (Alertmanager) free

## Quick start

This stack joins a shared Docker network called `observability-net` so
Prometheus can reach your app containers by name. Create it once:

```bash
docker network create observability-net
```

Then:

1. Copy the environment file and set a Grafana password.

   ```bash
   cp .env.example .env
   ```

2. Start the stack.

   ```bash
   docker compose up -d
   ```

3. Open Grafana at `http://localhost:3000`. The Prometheus data source is
   already connected.

4. Check what Prometheus is scraping at `http://localhost:9090/targets`. You
   should see `prometheus`, `cadvisor` and `node-exporter` all up.

## Scrape your own service

You get container and host metrics for free. To graph application metrics such
as request rate or latency, two things have to be true.

1. Your service exposes metrics. Spring Boot Actuator serves them at
   `/actuator/prometheus`. Most other stacks use a Prometheus client library
   that serves `/metrics`.

2. Your service is on the `observability-net` network so Prometheus can reach
   it, and you add it as a scrape target. Attach your app in its own compose
   file:

   ```yaml
   services:
     my-service:
       # ... your image and config ...
       networks:
         - observability-net

   networks:
     observability-net:
       external: true
   ```

   Then uncomment and edit the example job in `prometheus/prometheus.yml`:

   ```yaml
   - job_name: my-service
     metrics_path: /actuator/prometheus
     static_configs:
       - targets: ['my-service:8080']
   ```

3. Reload Prometheus without restarting it:

   ```bash
   curl -X POST http://localhost:9090/-/reload
   ```

Your service should now appear on the targets page.

## Alerts

The rules live in `prometheus/alerts.yml`. Out of the box:

- `HighContainerCPU` fires when a container sits above 80% CPU for 2 minutes.
- `HighContainerMemory` fires above 85% memory for 2 minutes.
- `MonitoringDown` fires if cAdvisor stops reporting.

Edit the thresholds and the `for` durations to taste. After editing, reload:

```bash
curl -X POST http://localhost:9090/-/reload
```

### Where alerts go

By default no notifications are sent, so the stack starts without needing any
account. Open `alertmanager/alertmanager.yml` and uncomment one receiver:

- Slack. Put your webhook in `.env` as `SLACK_WEBHOOK_URL` and uncomment the
  `slack_configs` block.
- Email. Fill in the `email_configs` block with your SMTP details.
- AWS SNS. Uncomment `sns_configs`, set the topic ARN and region. The container
  uses its IAM role or AWS credentials to publish.

Reload Alertmanager after editing:

```bash
docker compose restart alertmanager
```

## Dashboards

Grafana starts with the Prometheus data source connected but no dashboards, so
you pick what you want. Two well known community dashboards cover this stack.
In Grafana go to Dashboards, New, Import, and enter an ID:

- `1860` Node Exporter Full, for host metrics.
- `193` Docker and container metrics, for cAdvisor.

Importing by ID needs internet access from your browser. If the host is
offline, download the JSON elsewhere and import the file.

## Common settings

- Retention: `--storage.tsdb.retention.time` in `docker-compose.yml`, default
  15 days.
- Scrape frequency: `scrape_interval` in `prometheus.yml`.
- Grafana login: `GRAFANA_ADMIN_USER` and `GRAFANA_ADMIN_PASSWORD` in `.env`.

## Troubleshooting

- A target shows as down on the targets page. Confirm your service is on
  `observability-net` and that the port and metrics path are right. From inside
  the Prometheus container, `wget -qO- http://my-service:8080/actuator/prometheus`
  should return metrics text.
- No alerts arrive. Check the alert is firing at
  `http://localhost:9090/alerts`, then check Alertmanager received it at
  `http://localhost:9093`. If it fires but nothing arrives, the receiver is
  still empty or misconfigured.
- cAdvisor uses a lot of CPU. Raise `--housekeeping_interval` in the compose
  file.

## License

MIT, see [LICENSE](LICENSE).
