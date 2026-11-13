# Launchpad

A small internal developer platform. A developer opens one page and can:

1. **Create a service** from a template (Python or Node). Launchpad scaffolds a git repo with app code, a Dockerfile, a manifest and Terraform.
2. **Provision its infrastructure** (database, object storage) on AWS, GCP or Azure. Terraform is generated per service.
3. **Deploy it** as versioned releases, with health checks, automatic rollback on a bad release, and one-click rollback to any earlier release.
4. **See logs and metrics** in the same UI: live log tail with level filter and search, plus request rate, error rate, latency and availability charts.

## Run it

```bash
pip install -r requirements.txt
./run.sh                      # http://127.0.0.1:8080
python3 smoke_test.py         # end-to-end check against the running server
```

Needs Python 3.10+. Node is only needed if you deploy a Node service. All state lives in `data/` (override with `LAUNCHPAD_DATA`).

## What is real and what is a stand-in

| Step | What happens by default | Real-cloud path |
|---|---|---|
| Scaffold | Real git repo in `data/repos/<name>` | Same |
| Provision | **Local provisioner**: creates a real SQLite file and directory and injects `DATABASE_URL` / `STORAGE_PATH`. Terraform is generated but **not applied**. | `LAUNCHPAD_PROVISIONER=terraform ./run.sh` runs `terraform init/apply` in the service's `infra/` using your own credentials, and injects non-sensitive outputs as env vars. |
| Deploy | **Local process deployer**: snapshots the repo to `data/releases/<name>/<n>`, runs it on a port from 9100-9199, waits for `/healthz`. | Implement a Docker/Kubernetes deployer by replacing `_start` / `_kill` in `app/deploy.py`; the release, health-check and rollback flow around them stays. |
| Logs | Service stdout/stderr go to `data/logs/<name>.log`; the UI tails the file | Ship the same JSON lines to your log backend and point `read_logs` at it |
| Metrics | Scrapes each service's `/metrics` (Prometheus text) every 5 s into SQLite, kept 24 h | Replace with Prometheus queries in `query_metrics` |

The Terraform path (and the generated Azure/GCP HCL) was **not executed or `terraform validate`d** while building this, because Terraform was not available in the build environment. Run `terraform init && terraform validate` in a service's `infra/` before the first real apply. Terraform mode is opt-in so a developer machine never touches a cloud account by accident.

## Service contract

Any service the platform deploys must:

- serve `GET /healthz` returning 200,
- serve `GET /metrics` in Prometheus text format (`http_requests_total`, `http_request_duration_seconds_sum/_count`, `process_uptime_seconds`),
- write one JSON object per line to stdout (`ts`, `level`, `msg`, plus any fields).

The templates in `scaffolds/` already do all three. Add a template by creating a folder there and registering it in `app/scaffold.py`.

## Layout

```
app/main.py        API routes and startup reconciliation
app/scaffold.py    repo creation, release snapshots
app/infra_gen.py   Terraform generation for aws / gcp / azure
app/provision.py   local and terraform provisioners
app/deploy.py      release lifecycle: deploy, rollback, stop, restart
app/observe.py     log reading, metric scraping and aggregation, supervisor loop
app/core.py        SQLite state, events, per-service operation locks
static/index.html  the whole UI (no build step)
scaffolds/         service templates
```

## Known limits

- **No authentication.** Anyone who can reach the port can create, deploy and delete services. Put it behind your SSO proxy before sharing it.
- Single host: the deployer runs services on the machine Launchpad runs on, and redeploys have a few seconds of downtime (stop old, start new).
- Infrastructure is chosen at creation time; changing resources afterwards means editing `infra/main.tf` yourself.
- Deleting a service stops it, destroys its infrastructure, and moves its repo to `data/trash/`.
- Sensitive Terraform outputs (for example database passwords) are deliberately not stored or injected.
