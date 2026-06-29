#  Observability & Incident Response Platform

![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)
![OpenSearch](https://img.shields.io/badge/OpenSearch-005EB8?style=flat&logo=opensearch&logoColor=white)
![Jaeger](https://img.shields.io/badge/Jaeger-66CFE3?style=flat&logo=jaeger&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazonaws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)

We kept getting blindsided. Services would degrade, users would complain, and we'd spend the first 40 minutes of every incident just trying to figure out *where* the problem was - tailing logs on individual EC2 instances, guessing which service was involved, manually correlating timestamps across three different tools.

So I built this. The goal wasn't to use every shiny tool but it was to get metrics, logs, and traces into one place where you can actually move between them quickly when something's on fire at 2am.

---

##  The Problem

Three microservices on AWS (Node.js API, Python backend, Go worker). No centralised logging. Alerts that fired only when CPU hit 100% - by which point the damage was already done. No distributed tracing, so when a request failed, we had no idea if it died in the API, the backend, or somewhere in between.

Mean time to resolution was hovering around **45–60 minutes** for anything non-trivial. Most of that time was just investigation, not fixing.

---

##  Architecture


| Component | Role |
|---|---|
| **Prometheus** | Scrapes and stores time-series metrics |
| **Grafana** | Single pane of glass for metrics, logs, and traces |
| **Fluent Bit** | Collects and ships logs from all services |
| **OpenSearch** | Centralised log storage and search |
| **Jaeger** | Distributed tracing across services via OpenTelemetry |
| **Alertmanager** | Alert routing to Slack and AWS SNS |
| **AWS Lambda** | Auto-remediation for known failure patterns |

Deployed on **AWS EC2 (t3.xlarge)**, with Prometheus metrics retained for 30 days locally and shipped to S3 via Thanos for longer-term storage.

---

##  Getting Started

### Prerequisites

- Docker `>= 24.x` and Docker Compose `>= 2.x`
- AWS CLI configured (`aws configure`)
- 4 vCPU / 16 GB RAM minimum

### Clone the repo

```bash
git clone https://github.com/your-username/observability-platform.git
cd observability-platform
```

### Spin up the core stack

```bash
docker compose up -d
```

### Add logging and tracing

```bash
docker compose up -d opensearch opensearch-dashboards fluent-bit jaeger
```

### Verify everything is running

```bash
docker compose ps
```

---

##  Service URLs

| Service | URL | Credentials |
|---|---|---|
| Grafana | `http://localhost:3000` | `admin / admin123` |
| Prometheus | `http://localhost:9090` | — |
| Alertmanager | `http://localhost:9093` | — |
| Jaeger UI | `http://localhost:16686` | — |
| OpenSearch Dashboards | `http://localhost:5601` | — |
| OpenSearch API | `http://localhost:9200` | — |
| Node Exporter | `http://localhost:9100/metrics` | — |

> ⚠️ Change the Grafana admin password before exposing this outside `localhost`.

---

## 📊 Metrics — Prometheus + Grafana

Prometheus scrapes your app's `/metrics` endpoint every `15s`. Node Exporter handles host-level metrics — CPU, memory, disk, network.

**CPU usage %**

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**p95 response latency**

```promql
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

**Error rate %**

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) * 100
```

**SLO burn rate** - values above `1` mean the error budget will be exhausted before month end

```promql
(1 - job:http_success_rate:ratio_rate1h) / (1 - 0.999)
```

**Community dashboards to import into Grafana:**

| Dashboard | ID |
|---|---|
| Node Exporter Full | `1860` |
| Prometheus 2.0 Overview | `3662` |

---

## 🪵 Logging - Fluent Bit + OpenSearch

Fluent Bit tails Docker container logs and app log files, enriches them with `hostname` and `environment` metadata, and ships to OpenSearch. Logs are indexed into daily indices (`app-logs-YYYY.MM.DD`) and automatically deleted after **30 days** via an ISM lifecycle policy.

Core output block in `fluent-bit/fluent-bit.conf`:

```text
[OUTPUT]
    Name            opensearch
    Match           *
    Host            opensearch
    Port            9200
    Logstash_Format On
    Logstash_Prefix app-logs
    Retry_Limit     False
```

---

##  Distributed Tracing - Jaeger + OpenTelemetry

Instrument once, get traces for everything — HTTP, Express, database calls, gRPC.

**Node.js**

```bash
npm install @opentelemetry/sdk-node \
            @opentelemetry/auto-instrumentations-node \
            @opentelemetry/exporter-trace-otlp-http
```

**Python**

```bash
pip install opentelemetry-sdk opentelemetry-instrumentation
```

### The piece that made the biggest difference

Injecting trace IDs into every log line. When you find an error in OpenSearch, you click the `trace_id` and land directly in Jaeger with the full request journey across all services. Before this, cross-service failures meant manually correlating timestamps and guessing. After, usually **under 2 minutes**.

```python
class TraceIdFilter(logging.Filter):
    def filter(self, record):
        span = trace.get_current_span()
        ctx  = span.get_span_context()
        record.trace_id = format(ctx.trace_id, '032x') if ctx.is_valid else 'no-trace'
        record.span_id  = format(ctx.span_id,  '016x') if ctx.is_valid else 'no-span'
        return True
```

---

##  Alerting - Alertmanager

The problem with our old alerting wasn't the tooling - alerts fired too late and went to everyone, so they were effectively ignored. Rebuilt it around three principles:

- **Alert on symptoms, not causes.** High error rate matters. High CPU might not.
- **Route to the right person.** DB alerts → DBA channel. Infra → ops. Nothing to everyone.
- **Give context in the alert.** Summary, affected service, runbook link - on-call engineers shouldn't have to go hunting.

Routing config in `alertmanager/alertmanager.yml`:

```yaml
route:
  routes:
    - match:
        severity: critical
      receiver: slack-critical      # Immediate Slack + PagerDuty

    - match:
        service: database
      receiver: dba-team            # DBA channel only

    - match_re:
        alertname: ^(NodeDown|DiskFull|HighCPU)$
      receiver: ops-team
```

### SLO burn rate alerting

Instead of alerting after breaching the error budget, the burn rate alert fires when we're consuming it fast enough to exhaust it within **5 days**. That's the difference between proactive and reactive.

```yaml
- alert: SLOBudgetBurnRateHigh
  expr: |
    (1 - job:http_success_rate:ratio_rate1h) > (1 - 0.999) * 14.4
  for: 2m
  labels:
    severity: critical
```

---

##  Auto-Remediation — AWS Lambda

Two Lambda functions handle the most common failure patterns automatically:

- **`auto_remediate_ecs.py`** - Forces a new ECS deployment when a `ServiceDown` alert fires. Notifies Slack either way; escalates to a human if remediation itself fails.
- **`auto_scale_ec2.py`** - Adds two instances to the ASG on high-CPU alert, scales back in on resolve.

These cover roughly **30% of incidents**. Not everything can be auto-remediated, but the ones that can shouldn't require waking someone up.

### Deploy

```bash
cd lambda
zip auto_remediate_ecs.zip auto_remediate_ecs.py
```

```bash
aws lambda create-function \
  --function-name auto-remediate-ecs \
  --runtime python3.11 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/lambda-remediation-role \
  --handler auto_remediate_ecs.lambda_handler \
  --zip-file fileb://auto_remediate_ecs.zip \
  --timeout 60
```

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:${ACCOUNT_ID}:ops-alerts \
  --protocol lambda \
  --notification-endpoint ${LAMBDA_ARN}
```

---

##  Useful Commands

Reload Prometheus config without restarting:

```bash
curl -X POST http://localhost:9090/-/reload
```

Test the full alert pipeline end to end:

```bash
curl -X POST http://localhost:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels":      { "alertname": "TestAlert", "severity": "warning" },
    "annotations": { "summary": "Pipeline check — ignore this" }
  }]'
```

Check all scrape targets are healthy:

```bash
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool
```

Tail live logs for any service:

```bash
docker compose logs -f prometheus
docker compose logs -f fluent-bit
docker compose logs -f jaeger
```

---

##  Repo Structure

```
observability-platform/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
│       ├── alerts.yml          # Infrastructure + app + SLO alerts
│       └── slo.yml             # Error budget recording rules
├── grafana/
│   └── provisioning/
│       ├── datasources/        # Prometheus, OpenSearch, Jaeger auto-wired
│       └── dashboards/
├── alertmanager/
│   └── alertmanager.yml        # Routing + Slack + SNS config
├── fluent-bit/
│   ├── fluent-bit.conf
│   └── parsers.conf
├── lambda/
│   ├── auto_remediate_ecs.py
│   └── auto_scale_ec2.py
├── thanos/
│   └── objstore.yml            # S3 long-term metrics storage
└── runbooks/
    └── high-error-rate.md
```

---

##  Before Going to Production

-  Change the Grafana admin password (`GF_SECURITY_ADMIN_PASSWORD`)
-  Enable OpenSearch TLS — `plugins.security.disabled=true` is for local dev only
-  Add your Slack webhook URL to `alertmanager/alertmanager.yml`
-  Swap Jaeger `badger` storage to OpenSearch for production (`SPAN_STORAGE_TYPE=elasticsearch`)
-  Tighten Lambda IAM roles to least privilege
-  Update SLO targets in `prometheus/rules/slo.yml` to match real commitments

---

##  Stack Versions

| Tool | Version |
|---|---|
| Prometheus | `2.47.0` |
| Grafana | `10.2.0` |
| Alertmanager | `0.26.0` |
| OpenSearch | `2.11.0` |
| Fluent Bit | `2.2.0` |
| Jaeger | `1.51` |
| Node Exporter | `1.7.0` |

---

## 📄 License

MIT
