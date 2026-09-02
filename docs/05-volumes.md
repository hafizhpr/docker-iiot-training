# 05 — Volumes and Persistence

## Why containers forget

A container's writable layer is deleted when the container is deleted. Without
volumes, every `docker compose down` would wipe your database, your Node-RED
flows, and your Grafana dashboards.

That is not a bug. Containers are designed to be disposable — which only works
if the valuable data lives somewhere else.

## Two ways to keep data

### Named volume

```yaml
volumes:
  - influxdb_data:/var/lib/influxdb2
```

Docker creates and manages the storage. You do not choose where on disk it
lives, and you interact with it only through Docker commands.

**Use for:** databases, application state, anything the container owns.

### Bind mount

```yaml
volumes:
  - ./mosquitto/config:/mosquitto/config
```

A directory on **your** machine is mapped into the container. Edit the file in
your normal editor and the container sees the change immediately.

**Use for:** configuration you want to edit, source code during development.

### Choosing between them

| | Named volume | Bind mount |
|---|---|---|
| Managed by | Docker | You |
| Location | Docker's internal storage | A path you pick |
| Editable from host | Awkward | Trivial |
| Permission problems | Rare | Common |
| Portable across machines | Yes | Depends on the path |

Our stack uses both on purpose, so you can feel the difference.

## What our stack persists

| Service | Volume | Contents |
|---|---|---|
| Mosquitto | `mosquitto_data` | Retained messages, queued QoS 1/2 messages |
| Mosquitto | `mosquitto_log` | Broker log file |
| Mosquitto | `./mosquitto/config` *(bind)* | `mosquitto.conf` |
| InfluxDB | `influxdb_data` | All measurement data |
| InfluxDB | `influxdb_config` | Org, bucket, and token setup |
| Node-RED | `nodered_data` | Flows, credentials, installed palette nodes |
| Grafana | `grafana_data` | Dashboards, users, datasources |

Every named volume must also be declared at the bottom of the file:

```yaml
volumes:
  mosquitto_data:
  mosquitto_log:
  influxdb_data:
  influxdb_config:
  nodered_data:
  grafana_data:
```

Forget one and Compose refuses to start with `service refers to undefined
volume`.

## Prove persistence

Write a data point:

```powershell
docker compose exec influxdb influx write  --bucket sensors --org training --token training-token-do-not-use-in-production  --precision s "temperature,line=line1 value=42.5"
```

Destroy every container:

```powershell
docker compose down
docker compose ps -a
```

Nothing left. Now bring it back and look for the data:

```powershell
docker compose up -d
Start-Sleep 10
docker compose exec influxdb influx query --org training --token training-token-do-not-use-in-production 'from(bucket:\"sensors\") |> range(start:-1h) |> filter(fn:(r) => r._measurement == \"temperature\")'
```

> **Why the `\"` escapes?** Flux requires double quotes around string
> literals, but Windows PowerShell 5.1 strips embedded double quotes when it
> hands arguments to a native program. Without the backslashes, InfluxDB
> receives `bucket: sensors` and answers `undefined identifier sensors`.
> Escaping them is the reliable form on PowerShell.

The reading is still there. The containers were deleted and recreated; the
volume was not touched.

## `down` vs `down -v`

```powershell
docker compose down       # removes containers + network. VOLUMES SURVIVE.
docker compose down -v    # removes containers + network + VOLUMES. Data gone.
```

Run the persistence experiment again with `down -v` and the query comes back
empty — and InfluxDB re-runs its whole setup, because `influxdb_config` was
deleted too.

`-v` is the flag that ends careers. There is no undo.

## Inspecting volumes

```powershell
docker volume ls
docker volume inspect iiot-training_influxdb_data
docker system df -v
```

Volume names are prefixed with the Compose project name, exactly like networks.

`docker system df -v` shows how much disk each volume consumes — useful once a
time-series database has been collecting for a while.

## Environment-driven configuration

Volumes persist data; environment variables configure behaviour. Our InfluxDB
service initialises itself entirely from `.env`:

```yaml
environment:
  DOCKER_INFLUXDB_INIT_MODE: setup
  DOCKER_INFLUXDB_INIT_USERNAME: ${INFLUXDB_USERNAME}
  DOCKER_INFLUXDB_INIT_PASSWORD: ${INFLUXDB_PASSWORD}
  DOCKER_INFLUXDB_INIT_ORG: ${INFLUXDB_ORG}
  DOCKER_INFLUXDB_INIT_BUCKET: ${INFLUXDB_BUCKET}
  DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: ${INFLUXDB_TOKEN}
```

Compose reads `.env` from the same directory automatically.

`setup` mode only runs when `influxdb_config` is empty. On the second start
the volume already holds a configuration, so the variables are ignored — which
is why changing a password in `.env` appears to do nothing until you
`down -v`.

Confirm what Compose actually resolved before starting anything:

```powershell
docker compose config
```

## Checkpoint

1. Which volume type would you use for a Postgres data directory, and which for
   an `nginx.conf` you plan to tweak?
2. What is the difference between `down` and `down -v`?
3. You change `INFLUXDB_PASSWORD` in `.env` and restart. Nothing happens. Why?

---

**Previous:** [04 — Networking](04-networking.md) · **Next:** [06 — Lab: MQTT](06-lab-mqtt.md)
