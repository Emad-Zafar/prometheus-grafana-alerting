# Metrics and alerting with Prometheus, Alertmanager and Grafana

Watch CPU, memory, disk and network for your Docker host and every container on
it, and get an alert when something crosses a line. cAdvisor and node-exporter
gather the numbers, Prometheus stores them and runs the rules, Alertmanager
sends the notifications, Grafana draws the graphs.

## What you get

- Per container metrics out of the box (cAdvisor)
- Host metrics: load, memory, disk (node-exporter)
- Alert rules for high CPU, high memory, and the pipeline going down
- Notifications to Slack, email or AWS SNS
- Grafana ready to graph all of it

## Good for

Docker containers on one or a few hosts. One Prometheus scrapes the local
cAdvisor and node-exporter. For a large dynamic fleet you would move to service
discovery, but the config here is the same starting shape.

## Run it

This stack shares a Docker network with your app containers so it can scrape
them. Create it once, then start up:

```bash
docker network create observability-net
cp .env.example .env       # set a Grafana password
docker compose up -d
```

Open Grafana at `http://localhost:3000` (Prometheus is already connected). Check
what is being scraped at `http://localhost:9090/targets`.

## Scrape your own service

Container and host metrics come for free. For app metrics (request rate,
latency), do two things:

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

Then reload: `curl -X POST http://localhost:9090/-/reload`.

## Alerts

Rules are in `prometheus/alerts.yml` (high CPU, high memory, pipeline down).
Edit the thresholds and reload Prometheus as above.

By default nothing is sent, so the stack starts with no accounts needed. Open
`alertmanager/alertmanager.yml`, uncomment the Slack, email or SNS block, fill
it in, then `docker compose restart alertmanager`.

## Dashboards

Grafana starts with no dashboards so you pick what you want. In Grafana go to
Dashboards, New, Import, and enter an ID:

- `1860` Node Exporter Full (host metrics)
- `193` Docker and container metrics (cAdvisor)

## Ports

Grafana 3000, Prometheus 9090, Alertmanager 9093.

## Troubleshooting

- A target shows down? Confirm the app is on `observability-net` and the port
  and path are right. Test from inside Prometheus:
  `wget -qO- http://my-service:8080/actuator/prometheus`.
- No alert arrives? Check it is firing at `http://localhost:9090/alerts`, then
  that Alertmanager got it at `http://localhost:9093`. If it fires but nothing
  arrives, the receiver is still empty.

## License

MIT, see [LICENSE](LICENSE).
