# Metrics and alerting with Prometheus, Alertmanager and Grafana

Watch CPU, memory, disk and network for your Docker host and every container on
it, and get an alert when something crosses a line. cAdvisor and node-exporter
gather the numbers, Prometheus stores them and runs the rules, Alertmanager
sends the notifications, Grafana draws the graphs.

This README is a build-it-yourself guide. Follow the steps to add this stack to
your own project by copying each file in order. If you just want it running,
clone this repo and skip to [Run it](#run-it).

## Good for

Docker containers on one or a few hosts. One Prometheus scrapes the local
cAdvisor and node-exporter. For a large dynamic fleet you would move to service
discovery, but this is the same starting shape.

## Prerequisites

- Docker and Docker Compose on the host
- Ports 3000 (Grafana), 9090 (Prometheus) and 9093 (Alertmanager) free

## What you will create

```
your-project/
  prometheus/
    prometheus.yml
    alerts.yml
  alertmanager/
    alertmanager.yml
  grafana/provisioning/datasources/
    prometheus.yaml
  .env
  docker-compose.yml
```

Build the files in the order below. Each step says what the file does and which
file in this repo to copy it from.

## Step 1: Prometheus scrape config

Create `prometheus/prometheus.yml`. Copy it from
[`prometheus/prometheus.yml`](prometheus/prometheus.yml).

This is the heart of the setup. It lists what to scrape (itself, cAdvisor,
node-exporter, and a commented example for your own service), tells Prometheus
where Alertmanager is, and loads the alert rules from step 2. You will add your
own services here later. Everything else can stay.

## Step 2: Alert rules

Create `prometheus/alerts.yml`. Copy it from
[`prometheus/alerts.yml`](prometheus/alerts.yml).

The rules Prometheus evaluates: high container CPU, high container memory, and
the metrics pipeline going down. Each rule is a query, a threshold, and how long
it must hold before firing. Edit the thresholds to taste. `prometheus.yml`
already points at this file.

## Step 3: Alertmanager routing

Create `alertmanager/alertmanager.yml`. Copy it from
[`alertmanager/alertmanager.yml`](alertmanager/alertmanager.yml).

Decides where a firing alert goes. It ships with an empty default receiver, so
nothing is sent until you fill one in. There are commented blocks for Slack,
email and AWS SNS. Uncomment one when you are ready.

## Step 4: Grafana data source

Create `grafana/provisioning/datasources/prometheus.yaml`. Copy it from
[`grafana/provisioning/datasources/prometheus.yaml`](grafana/provisioning/datasources/prometheus.yaml).

Tells Grafana about Prometheus on first boot, pointing at
`http://prometheus:9090`. Leave it as is.

## Step 5: Environment file

Copy [`.env.example`](.env.example) to `.env` and set a Grafana password:

```bash
cp .env.example .env
```

## Step 6: Compose file

Create `docker-compose.yml`. Copy it from
[`docker-compose.yml`](docker-compose.yml).

Last, because it ties everything together. It runs Prometheus, Alertmanager,
cAdvisor, node-exporter and Grafana, and mounts each config you created. It joins
an external Docker network called `observability-net` so Prometheus can reach
your app containers by name. You should not need to edit this.

## Run it

Create the shared network first, then start up:

```bash
docker network create observability-net
docker compose up -d
```

Open Grafana at `http://localhost:3000` (Prometheus is already connected). Check
what is being scraped at `http://localhost:9090/targets`.

## Scrape your own service

Container and host metrics come for free. For app metrics (request rate,
latency):

1. Expose metrics from your app. Spring Boot Actuator serves
   `/actuator/prometheus`, most other stacks serve `/metrics`.
2. Put your app on `observability-net` and add it as a target in
   `prometheus/prometheus.yml`:

   ```yaml
   - job_name: my-service
     metrics_path: /actuator/prometheus
     static_configs:
       - targets: ['my-service:8080']
   ```

3. Reload Prometheus: `curl -X POST http://localhost:9090/-/reload`

## Dashboards

Grafana starts with no dashboards so you pick your own. In Grafana go to
Dashboards, New, Import, and enter an ID:

- `1860` Node Exporter Full (host metrics)
- `193` Docker and container metrics (cAdvisor)

## Troubleshooting

- A target shows down? Confirm the app is on `observability-net` and the port
  and path are right. From inside Prometheus:
  `wget -qO- http://my-service:8080/actuator/prometheus`.
- No alert arrives? Check it is firing at `http://localhost:9090/alerts`, then
  that Alertmanager got it at `http://localhost:9093`. If it fires but nothing
  arrives, the receiver is still empty.

## License

MIT, see [LICENSE](LICENSE).
