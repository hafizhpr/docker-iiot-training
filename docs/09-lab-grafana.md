# 09 — Lab: Grafana

**Goal:** connect Grafana to InfluxDB and build a live dashboard.

Make sure data is still flowing — either the inject-node sensor from Lesson 07
or the Modbus poll from Lesson 08. Either way the topic is the same, so this
lesson does not care which one is feeding it.

## Step 1 — Log in

Open <http://localhost:3000>.

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `admin` |

Grafana may offer to change the password — **Skip** is fine here.

Those credentials came from the environment, not from a manual setup wizard:

```yaml
environment:
  GF_SECURITY_ADMIN_USER: ${GRAFANA_USER}
  GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
```

Every Grafana setting can be driven this way. The pattern is
`GF_<SECTION>_<KEY>` in uppercase — `GF_SERVER_HTTP_PORT`,
`GF_USERS_ALLOW_SIGN_UP`, and so on. This is how Grafana is configured in
production too; the only thing wrong with ours is the value `admin`.

## Step 2 — Add the InfluxDB datasource

**☰ → Connections → Data sources → Add new data source → InfluxDB**

| Field | Value |
|---|---|
| Name | `InfluxDB` |
| Query language | **Flux** |
| URL | `http://influxdb:8086` |
| Auth: Basic auth | off |
| Organization | `training` |
| Token | `training-token-do-not-use-in-production` |
| Default Bucket | `sensors` |

**URL is `http://influxdb:8086`.** You reached Grafana at `localhost:3000` in
your browser, but Grafana itself is a container — its `localhost` is itself.
This is the third time the same rule has decided the outcome. It will not be
the last.

Click **Save & test**. Expect `datasource is working`.

If you get a connection error, run this to confirm the path Grafana uses is
actually open:

```powershell
docker compose exec grafana wget -qO- http://influxdb:8086/health
```

JSON back means the network is fine and the problem is the token or the
organization name.

## Step 3 — Build a panel

**☰ → Dashboards → New → New dashboard → Add visualization →** pick `InfluxDB`.

Switch the query editor to the **Flux** text mode and enter:

```flux
from(bucket: "sensors")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "temperature")
  |> filter(fn: (r) => r._field == "value")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

Reading it line by line:

| Line | Meaning |
|---|---|
| `from(bucket:)` | Which bucket to read |
| `range(...)` | The time window — `v.timeRangeStart` follows the dashboard picker |
| `filter(_measurement)` | Only temperature rows |
| `filter(_field)` | Only the `value` field |
| `aggregateWindow(...)` | Average per interval, so a wide window stays readable |

The `v.` variables are supplied by Grafana. Using them is what makes the
dashboard time picker and zoom work.

Set the time range to **Last 15 minutes**, and the auto-refresh (top right) to
**5s**. Give the panel a title, then **Save dashboard**.

## Step 4 — Split by production line

Change the query to group by the tag we set in Node-RED:

```flux
from(bucket: "sensors")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "temperature")
  |> filter(fn: (r) => r._field == "value")
  |> group(columns: ["line"])
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

Publish from a second line and watch a new series appear on its own:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line2/press02/telemetry/temperature' -m '39.4'
```

This is why tags matter. `line` was a tag, so it is indexed and cheap to group
by. Had it been a field, this query would be far slower at scale.

## Step 5 — Add a threshold alert

On the panel: **Edit → Alert → New alert rule**.

| Setting | Value |
|---|---|
| Condition | `WHEN last() OF query IS ABOVE 41` |
| Evaluate every | `1m` for `1m` |

Save it, then push a value over the line:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/temperature' -m '55'
```

Within the evaluation interval the rule fires. Delivering that alert to email,
Slack, or Telegram is a contact-point configuration — outside this course, but
this is where it starts.

## Step 6 — Prove the dashboard persists

```powershell
docker compose down
docker compose up -d
```

Log back in. Your dashboard and datasource are intact, because
`grafana_data:/var/lib/grafana` holds them.

Then consider what `docker compose down -v` would do to the same dashboard, and
you will understand why that flag deserves respect.

## Where each thing lives

| Item | Stored in |
|---|---|
| Measurements | `influxdb_data` |
| Dashboards, users, datasources | `grafana_data` |
| Flows and palette nodes | `nodered_data` |
| Retained MQTT messages | `mosquitto_data` |

Four containers, four independent lifecycles. Any one of them can be rebuilt
without touching the others — the core argument for containerising a stack
like this.

## Checkpoint

1. Grafana runs in your browser at `localhost:3000`, yet the datasource URL is
   `http://influxdb:8086`. Explain the difference to someone who has not read
   Lesson 04.
2. What do the `v.timeRangeStart` and `v.windowPeriod` variables do?
3. Why was making `line` a tag rather than a field a good decision?

---

**Previous:** [08 — Lab: Modbus to MQTT](08-lab-modbus.md) · **Next:** [10 — Troubleshooting](10-troubleshooting.md)
