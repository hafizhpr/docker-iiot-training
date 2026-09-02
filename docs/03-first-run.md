# 03 — First Run

You have installed Docker and you understand what the stack does. Now you start
it — and, more importantly, learn to tell whether it actually worked.

## Read the file before you run it

Open `docker-compose.yml`. It looks long, but it is the same short pattern
repeated four times.

### The outer shape

```yaml
name: iiot-training      # project name — prefixes volumes and networks

services:                # the containers
  mosquitto: ...
  influxdb: ...
  nodered: ...
  grafana: ...

volumes:                 # every named volume must be declared here
  mosquitto_data:
  ...

networks:                # the private network they share
  iiot-net:
    driver: bridge
```

Three top-level blocks. Everything else is detail.

### One service, key by key

```yaml
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: ${INFLUXDB_USERNAME}
    volumes:
      - influxdb_data:/var/lib/influxdb2
    networks:
      - iiot-net
```

| Key | What it does |
|---|---|
| `image` | Which image to run. Pinned to `2.7`, not `latest` |
| `container_name` | A fixed, readable name instead of a random one |
| `restart: unless-stopped` | Restart automatically unless you stopped it yourself |
| `ports` | `host:container` — publishes the port to your machine |
| `environment` | Configuration passed in at startup; `${...}` is read from `.env` |
| `volumes` | `volume_name:/path/in/container` — what survives the container |
| `networks` | Which network to join, so DNS by service name works |

Two more keys appear elsewhere in the file:

| Key | Where | What it does |
|---|---|---|
| `depends_on` | `nodered`, `grafana` | Start order only — **not** readiness |
| `extra_hosts` | `nodered` | Adds `host.docker.internal` so flows can reach your host machine — Lesson 04 |

That is the entire vocabulary of this compose file. Four services, same keys.

## Check before you start

```powershell
docker compose config
```

This prints the file with every `${VARIABLE}` substituted, and fails loudly on
a syntax error. Two things to look for:

- Passwords and tokens show real values, not empty strings. Empty means `.env`
  was not found — usually because you are in the wrong directory
- No `variable is not set` warnings

Getting into the habit of running this first will save you a confusing debug
session later.

## Start the stack

```powershell
docker compose up -d
```

`-d` is detached — it returns your prompt instead of streaming logs forever.

**The first run pulls roughly 2.5 GB of images** and takes a few minutes on a
normal connection. The download itself is smaller — layers travel compressed
and expand on disk — so watch the progress, not the clock. You will see
layer-by-layer progress:

```
mosquitto Pulling
 6e771e15690e Pull complete
 ...
mosquitto Pulled
influxdb Pulling
```

Every later start skips all of that, because the images are already on disk.

Then the creation order:

```
 Volume iiot-training_influxdb_data  Created
 Network iiot-training_iiot-net      Created
 Container mosquitto                 Created
 Container influxdb                  Created
 Container nodered                   Created
 Container grafana                   Created
 Container mosquitto                 Started
 ...
```

Note what happens first. Volumes and the network are created **before** any
container, because containers attach to them.

## Verify

```powershell
docker compose ps
```

```
NAME        STATUS          PORTS
grafana     Up 6 seconds    0.0.0.0:3000->3000/tcp
influxdb    Up 6 seconds    0.0.0.0:8086->8086/tcp
mosquitto   Up 6 seconds    0.0.0.0:1883->1883/tcp, 0.0.0.0:9001->9001/tcp
nodered     Up 6 seconds    0.0.0.0:1880->1880/tcp
```

Four rows, all `Up`. Anything showing `Restarting` or `Exited` means read the
logs now:

```powershell
docker compose logs <service-name>
```

Lesson 10 covers what the common failures look like.

## Open each service

| Service | URL | Login |
|---|---|---|
| Node-RED | <http://localhost:1880> | none |
| InfluxDB | <http://localhost:8086> | `admin` / `admin12345` |
| Grafana | <http://localhost:3000> | `admin` / `admin` |

Open all three now. If a page loads, that service is genuinely working —
container running, port published, application listening. Three green ticks
beat any amount of log reading.

Mosquitto has no web UI. Test it from the command line instead:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'test' -m 'hello' -r
docker compose exec mosquitto mosquitto_sub -h localhost -t 'test' -C 1 -W 5
```

`hello` comes back. The broker works.

Publishing with `-r` makes the broker retain the message, so the subscriber
receives it even though it connected afterwards. That is what lets you test a
broker from a single PowerShell window instead of juggling two.

### Why InfluxDB did not ask you to set anything up

Normally InfluxDB greets you with a setup wizard. Ours skipped it, because
`DOCKER_INFLUXDB_INIT_MODE: setup` plus the variables from `.env` performed the
whole initialisation on first boot — organisation, bucket, admin user, and API
token.

That is worth noticing as a pattern, not just a convenience: a container that
configures itself from the environment can be created and destroyed by a
script, with no human clicking through a wizard. It is the difference between
a machine you set up and a machine you can recreate.

## What was actually created

```powershell
docker compose ps       # 4 containers
docker volume ls | Select-String iiot-training    # 6 volumes
docker network ls | Select-String iiot-training   # 1 network
```

Everything carries the `iiot-training` prefix — the `name:` at the top of the
compose file. That prefix is what keeps this project from colliding with any
other Docker project on your machine.

## Stopping

```powershell
docker compose stop     # pause; start again with: docker compose start
docker compose down     # remove containers + network, keep all data
```

Both are safe. Lesson 11 explains the third one — and why it is not.

## Checkpoint

1. What does `docker compose config` show you that reading the file does not?
2. A service shows `Restarting (1)`. What is your next command?
3. Why do volumes and the network get created before any container?
4. Why did InfluxDB never show you a setup wizard?

---

**Previous:** [02 — The Stack](02-the-stack.md) · **Next:** [04 — Networking](04-networking.md)
