# 10 — Troubleshooting

## The diagnostic order

Work down this list. Almost every problem is identified within the first three
commands.

```powershell
docker compose ps          # 1. is it running at all?
docker compose logs <svc>  # 2. what did it say before it broke?
docker compose exec <svc> sh   # 3. go inside and look
```

Guessing is slower than looking. Look.

## Reading the status column

| Status | Meaning |
|---|---|
| `Up 5 minutes` | Healthy and running |
| `Restarting (1)` | Crashing on startup, in a loop — check logs immediately |
| `Exited (0)` | Stopped cleanly |
| `Exited (1)` | Stopped with an error |
| `Created` | Never started |

`Restarting` is the loud one. `restart: unless-stopped` is faithfully
restarting a container that dies every time.

---

## Problem: InfluxDB exits immediately

**Log:**

```
Error: failed to setup instance: 400 Bad Request:
passwords must be between 8 and 72 characters long
```

**Cause:** `DOCKER_INFLUXDB_INIT_PASSWORD` is shorter than 8 characters. The
obvious choice of `admin` is exactly 5.

**Fix:** in `.env`, use `INFLUXDB_PASSWORD=admin12345`.

This is the one place in this course where we could not honour the
"admin / admin everywhere" rule. The image enforces it and there is no flag to
turn it off — a good reminder that an image's own validation outranks your
compose file.

---

## Problem: MQTT clients cannot connect

**Symptom:** the container is `Up`, port 1883 is published, and every client is
refused.

**Cause:** Mosquitto 2.x binds only to loopback inside the container unless a
listener is declared.

**Fix:** confirm the config actually reached the container:

```powershell
docker compose exec mosquitto cat /mosquitto/config/mosquitto.conf
```

You must see `listener 1883`. If the file is empty or missing, the bind mount
path in `docker-compose.yml` is wrong.

---

## Problem: "Connection refused" between containers

**Cause:** you used `localhost` in a container-to-container address. This is
the most frequent error in the whole course.

**Fix:**

| Wrong | Right |
|---|---|
| `mqtt://localhost:1883` | `mqtt://mosquitto:1883` |
| `http://localhost:8086` | `http://influxdb:8086` |

**Verify the path directly:**

```powershell
docker compose exec nodered wget -qO- http://influxdb:8086/health
```

JSON back means the network is fine and your problem is credentials. Refused
means you are on the wrong address or the target is not up.

**Check DNS resolves at all:**

```powershell
docker compose exec nodered getent hosts influxdb
```

---

## Problem: port is already allocated

**Log:**

```
Error starting userland proxy: listen tcp 0.0.0.0:3000: bind: address already in use
```

**Cause:** something else on your machine owns that port. Port 3000 is
contested by React, Next.js, and Rails.

**Find the culprit:**

```powershell
Get-NetTCPConnection -LocalPort 3000 -State Listen | Select-Object LocalPort, OwningProcess
Get-Process -Id (Get-NetTCPConnection -LocalPort 3000 -State Listen).OwningProcess
```

The first line tells you the port is taken and which process id owns it; the
second turns that id into a name you recognise.

**Fix — move the host side only:**

```yaml
grafana:
  ports:
    - "3001:3000"     # browser uses 3001; Grafana still serves 3000
```

Then `docker compose up -d`. Nothing inside the container changes, and no other
container needs updating — they reach Grafana by service name, not host port.

---

## Problem: config edits have no effect (Windows)

**Cause:** your editor saved `mosquitto.conf` with CRLF line endings and the
parser choked, or the file was never mounted.

**Check what the container sees:**

```powershell
docker compose exec mosquitto cat -A /mosquitto/config/mosquitto.conf | Select-Object -First 5
```

`^M$` at the end of lines means CRLF.

**Fix:** save the file as LF in your editor (VS Code shows `CRLF` / `LF` in the
status bar — click it to switch). The `.gitattributes` in this repository
enforces LF for `.conf` files on checkout.

---

## Problem: changing `.env` does nothing

**Cause:** InfluxDB `setup` mode only runs when `influxdb_config` is empty. On
every later start the volume already holds a configuration and the variables
are ignored.

**Fix — only if you are willing to lose the data:**

```powershell
docker compose down -v
docker compose up -d
```

**Check what Compose resolved before starting:**

```powershell
docker compose config
```

This prints the fully substituted file. If a variable shows as empty, `.env` is
not being read — usually because you ran the command from a different directory.

---

## Problem: Node-RED palette node disappeared

**Cause:** `docker compose down -v` deleted `nodered_data`, which contained
`/data/node_modules`.

**Fix:** reinstall through Manage palette — or stop the problem recurring by
baking the node into a custom image ([Lesson 12](12-building-custom-images.md)).

---

## Problem: service refers to undefined volume

**Log:**

```
service "influxdb" refers to undefined volume influxdb_data
```

**Cause:** the volume is used in a service but not declared in the top-level
`volumes:` block.

**Fix:** add it:

```yaml
volumes:
  influxdb_data:
```

---

## Problem: a service starts before its dependency is ready

**Cause:** `depends_on` orders startup; it does not wait for readiness.

**Fix:** add a healthcheck and depend on it:

```yaml
influxdb:
  healthcheck:
    test: ["CMD", "influx", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5

nodered:
  depends_on:
    influxdb:
      condition: service_healthy
```

We left this out of the course stack because Node-RED reconnects by itself.
Add it when a service crashes rather than retries.

---

## Problem: palette install fails with EBADENGINE

**Log (from Manage palette, or `docker compose logs nodered`):**

```
npm error code EBADENGINE
npm error engine Unsupported engine
npm error notsup Not compatible with your version of node/npm: node-red-contrib-modbus@5.60.2
npm error notsup Required: {"node":">=22"}
npm error notsup Actual:   {"npm":"10.9.8","node":"v20.20.2"}
```

**Cause:** the palette node requires a newer Node.js runtime than the one baked
into your Node-RED image. Nothing is wrong with the node or with npm — the
image is simply older than the package expects.

Read the last two lines of the error. They tell you exactly what is required
and what you have.

**Check what you are running:**

```powershell
docker compose exec nodered node --version
```

**Fix:** switch to an image variant built on the required Node major. Node-RED
publishes tags with the Node version as a suffix:

| Tag | Node runtime |
|---|---|
| `nodered/node-red:latest` | Node 20 |
| `nodered/node-red:4.1.0-22` | Node 22 |

In `docker-compose.yml`:

```yaml
nodered:
  image: nodered/node-red:4.1.0-22
```

Then recreate just that service:

```powershell
docker compose up -d nodered
docker compose exec nodered node --version
```

Your flows and previously installed palette nodes are untouched — they live in
the `nodered_data` volume, and only the image changed. This is the payoff of
Lesson 05: the runtime is disposable, the data is not.

Retry the install from Manage palette.

> **Why this is an argument for pinning.** `latest` moved to Node 20 at some
> point without telling anyone, and will move again. A pinned tag like
> `4.1.0-22` states the runtime you tested against, so a package that installed
> yesterday still installs tomorrow.

> **Native modules.** Most palette nodes are pure JavaScript and survive a Node
> version change. A node with compiled bindings may need reinstalling after the
> switch — uninstall and reinstall it from Manage palette if it misbehaves.

---

## Commands worth memorising

```powershell
docker compose ps                      # status
docker compose logs -f --tail=50 grafana   # follow recent logs
docker compose exec influxdb sh        # shell inside
docker compose config                  # resolved configuration
docker compose restart mosquitto       # restart one service
docker compose up -d --force-recreate nodered   # rebuild one container
docker stats                           # live CPU and memory per container
docker network inspect iiot-training_iiot-net   # who is on the network
```

---

## Making this stack safe for real use

Every shortcut in this course, and what replaces it:

| Shortcut | Production replacement |
|---|---|
| `allow_anonymous true` | `password_file` plus per-device credentials, and TLS on 8883 |
| Hardcoded InfluxDB token | A generated token with least-privilege scope, injected as a secret |
| Grafana `admin` / `admin` | A strong password, or SSO / LDAP |
| `.env` committed to git | `.gitignore` the file; use a secret manager |
| `latest` tags | Pinned versions, upgraded deliberately |
| Ports published to `0.0.0.0` | Bind to `127.0.0.1`, or put a reverse proxy in front |

None of these are hard. They are all skipped here for one reason: a student who
spends the first hour on TLS certificates never gets to the Docker lesson.

## Checkpoint

1. A container shows `Restarting (1)`. What is your first command?
2. Grafana cannot reach InfluxDB. What single command distinguishes a network
   problem from a credentials problem?
3. You changed the InfluxDB bucket name in `.env` and nothing happened. Why?

---

**Previous:** [09 — Lab: Grafana](09-lab-grafana.md) · **Next:** [11 — Cleanup](11-cleanup.md)
