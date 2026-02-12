# TIG Stack + Jenkins

Monitor Jenkins CI metrics using the TIG pipeline:

**Jenkins** (CI server) → **Telegraf** (scrapes metrics) → **InfluxDB 2.x** (stores) → **Grafana** (visualizes)

## Architecture

| Service  | Image                    | Port  | Purpose                          |
|----------|--------------------------|-------|----------------------------------|
| InfluxDB | `influxdb:2`             | 8086  | Time-series storage (Flux)       |
| Jenkins  | `jenkins/jenkins:lts`    | 8080  | CI server (metric source)        |
| Telegraf | `telegraf:latest`        | —     | Scrapes Jenkins → writes InfluxDB|
| Grafana  | `grafana/grafana:latest` | 3000  | Dashboards and visualization     |

Telegraf waits for both InfluxDB and Jenkins to pass health checks before starting. Grafana's InfluxDB datasource and a Jenkins Overview dashboard are auto-provisioned at startup.

## Prerequisites

- [Podman](https://podman.io/) with `podman compose` (or Docker with `docker compose`)

## Quick Start

### 1. Create the `.env` file

Copy the example below and adjust values as needed:

```env
# InfluxDB
INFLUXDB_ORG=tig-org
INFLUXDB_BUCKET=jenkins
INFLUXDB_ADMIN_USER=admin
INFLUXDB_ADMIN_PASSWORD=admin12345
INFLUXDB_ADMIN_TOKEN=my-super-secret-token

# Grafana
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=admin
```

### 2. Start the stack

```sh
podman compose up -d
```

First run pulls images and may take a few minutes. Telegraf will start automatically once InfluxDB and Jenkins are healthy.

### 3. Verify everything is running

```sh
podman compose ps
```

All 4 containers should show `Up`:

```
NAME       IMAGE                              STATUS
grafana    docker.io/grafana/grafana:latest   Up
influxdb   docker.io/library/influxdb:2       Up
jenkins    docker.io/jenkins/jenkins:lts      Up
telegraf   docker.io/library/telegraf:latest  Up
```

## Service UIs

| Service  | URL                        | Credentials        |
|----------|----------------------------|--------------------|
| Jenkins  | http://localhost:8080      | No auth (wizard disabled) |
| InfluxDB | http://localhost:8086      | admin / admin12345 |
| Grafana  | http://localhost:3000      | admin / admin      |

## Running a Test Jenkins Job

Jenkins has the setup wizard disabled, so you can create jobs immediately via the REST API.

### Fetch a CSRF crumb (required for all POST requests)

```sh
CRUMB=$(curl -sf -c /tmp/jenkins-cookies \
  'http://localhost:8080/crumbIssuer/api/json')
CRUMB_FIELD=$(echo "$CRUMB" | python3 -c "import sys,json; print(json.load(sys.stdin)['crumbRequestField'])")
CRUMB_VALUE=$(echo "$CRUMB" | python3 -c "import sys,json; print(json.load(sys.stdin)['crumb'])")
```

### Create the job

```sh
curl -s -b /tmp/jenkins-cookies \
  -H "Content-Type: application/xml" \
  -H "$CRUMB_FIELD: $CRUMB_VALUE" \
  -X POST 'http://localhost:8080/createItem?name=test-job' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<project>
  <description>Test job for TIG stack verification</description>
  <builders>
    <hudson.tasks.Shell>
      <command>echo "Hello from Jenkins!" &amp;&amp; sleep 5 &amp;&amp; echo "Done."</command>
    </hudson.tasks.Shell>
  </builders>
</project>'
```

### Trigger a build

Re-fetch the crumb (they are single-use), then trigger:

```sh
CRUMB=$(curl -sf -c /tmp/jenkins-cookies \
  'http://localhost:8080/crumbIssuer/api/json')
CRUMB_FIELD=$(echo "$CRUMB" | python3 -c "import sys,json; print(json.load(sys.stdin)['crumbRequestField'])")
CRUMB_VALUE=$(echo "$CRUMB" | python3 -c "import sys,json; print(json.load(sys.stdin)['crumb'])")

curl -s -b /tmp/jenkins-cookies \
  -H "$CRUMB_FIELD: $CRUMB_VALUE" \
  -X POST 'http://localhost:8080/job/test-job/build'
```

### Check build result

Wait a few seconds for the build to finish, then:

```sh
curl -sf 'http://localhost:8080/job/test-job/lastBuild/api/json?pretty=true' \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f\"Build #{d['number']}  Result: {d['result']}  Duration: {d['duration']}ms\")
"
```

### View console output

```sh
curl -sf 'http://localhost:8080/job/test-job/lastBuild/consoleText'
```

## Viewing Metrics in Grafana

1. Open http://localhost:3000 and log in (`admin` / `admin`)
2. Go to **Dashboards** → **Jenkins Overview**
3. Telegraf scrapes Jenkins every 30 seconds — wait up to a minute after a build for data to appear

The dashboard includes 6 panels:

- **Executor Usage** — busy vs total executors over time
- **Job Build Duration** — build time per job
- **Job Build Result** — last result color-coded (green=success, red=failure)
- **Build Number Over Time** — build numbers plotted chronologically
- **Node Status** — table of nodes with online/offline status
- **Node Response Time** — node response latency over time

## Stopping the Stack

```sh
podman compose down
```

To also remove stored data (volumes):

```sh
podman compose down -v
```

## File Structure

```
tig-stack/
├── docker-compose.yml
├── .env
├── .gitignore
├── telegraf/
│   └── telegraf.conf
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── influxdb.yml
        └── dashboards/
            ├── dashboard-provider.yml
            └── jenkins-overview.json
```
