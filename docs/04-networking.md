# 04 — Networking

This is the lesson that saves you the most time later. Almost every "it works
on my machine but not in Docker" problem is a networking misunderstanding.

## Two numbers, two different worlds

```yaml
ports:
  - "8086:8086"
     │     │
     │     └── container port — the port INSIDE the container
     └──────── host port — the port on YOUR machine
```

They are usually written the same here because we kept the defaults, but they
are unrelated. This is equally valid:

```yaml
ports:
  - "9999:8086"    # reach InfluxDB at localhost:9999 from your browser
```

The container still believes it is serving on 8086. Only the host mapping moved.

## The private network

Compose creates a user-defined bridge network for the project. In our
`docker-compose.yml`:

```yaml
networks:
  iiot-net:
    driver: bridge
```

Every service joins it, and the network provides an embedded **DNS server**.
That is what makes this work:

```
mqtt://mosquitto:1883      not   mqtt://172.21.0.3:1883
http://influxdb:8086       not   http://172.21.0.2:8086
```

Container IP addresses change on every restart. Service names do not. Always
use the name.

## See it yourself

Start the stack, then look up the other containers from inside Node-RED:

```powershell
docker compose up -d
docker compose exec nodered getent hosts mosquitto influxdb grafana
```

Output, with the addresses your machine assigns:

```
172.21.0.3   mosquitto
172.21.0.2   influxdb
172.21.0.4   grafana
```

Three names, three addresses, resolved from inside a container. No hosts file,
no configuration.

Now prove the addresses are not stable:

```powershell
docker compose down
docker compose up -d
docker compose exec nodered getent hosts influxdb
```

Different address, same name. This is the whole argument for service names in
one experiment.

## The `localhost` trap

**This is the single most common mistake in this course.** Read it twice.

Inside a container, `localhost` means *that container itself* — not your
machine, and not any other container. Each container has its own loopback
interface.

So when you configure the InfluxDB node inside Node-RED:

| You type | Result |
|---|---|
| `http://localhost:8086` | ❌ Node-RED looks inside itself. Nothing is listening. Connection refused. |
| `http://influxdb:8086` | ✅ Docker DNS resolves `influxdb` to the right container. |

Same rule in Grafana when you add the datasource: the URL is
`http://influxdb:8086`, even though you reached the Grafana UI at
`localhost:3000` in your browser.

### The rule

| Who is connecting | Address to use |
|---|---|
| Your browser → a container | `localhost:<host port>` |
| A container → another container | `<service name>:<container port>` |

Your browser is on the host, outside the Docker network, so it uses the
published host port. Containers are inside, so they use service names.

Prove the trap is real:

```powershell
docker compose exec nodered wget -qO- http://influxdb:8086/health
docker compose exec nodered wget -qO- http://localhost:8086/health
```

The first returns JSON. The second fails to connect.

## Reaching the host machine

There is a third case the table above does not cover: a container that needs to
reach a service running on **your machine**, outside Docker entirely. A Modbus
TCP server, a local historian, a database you have not containerised yet.

`localhost` fails here for the same reason as before, and the service name does
not exist because there is no such container. The address is:

```
host.docker.internal
```

Check it resolves:

```powershell
docker compose exec nodered getent hosts host.docker.internal
```

```
192.168.65.254   host.docker.internal
```

You may see an IPv6 address instead, or both:

```powershell
docker compose exec nodered grep host.docker.internal /etc/hosts
```

```
192.168.65.254        host.docker.internal
fdc4:f303:9324::254   host.docker.internal
```

Either is fine. Both point at the same gateway back to your host, and the
client tries them in turn — a server listening only on IPv4 is still reached.

### Example: a Modbus TCP server on your PC

| Field in the Modbus node | Value |
|---|---|
| Type | `TCP` |
| Host | `host.docker.internal` |
| Port | `502` |

### The complete address rule

| Who is connecting | Address to use |
|---|---|
| Your browser → a container | `localhost:<host port>` |
| A container → another container | `<service name>:<container port>` |
| A container → your host machine | `host.docker.internal:<port>` |
| A container → itself | `localhost:<container port>` |

### Why `extra_hosts` is in the compose file

```yaml
nodered:
  extra_hosts:
    - "host.docker.internal:host-gateway"
```

Docker Desktop on Windows and macOS provides `host.docker.internal` for free.
Docker Engine on Linux does **not** — the name simply does not resolve.

That line adds it explicitly, so the same compose file works on every platform.
On Docker Desktop it changes nothing; on Linux it is the difference between
working and not.

### When it still fails

Two causes, in order of likelihood:

**The host service listens on `127.0.0.1` only.** The container arrives from
outside, so it is refused. The service must bind `0.0.0.0`. Check on the host:

```powershell
Get-NetTCPConnection -LocalPort 502 -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess
```

`LocalAddress` of `0.0.0.0` is reachable from a container. `127.0.0.1` is not.

**A host firewall is blocking the port.** On Windows, add an inbound rule for
it. Worth knowing: the host sees the connection arriving from `127.0.0.1`
rather than from the container IP, because Docker Desktop proxies it — so
firewall rules written against a container address will not behave as expected.

## depends_on does not mean "wait until ready"

```yaml
depends_on:
  - mosquitto
  - influxdb
```

This controls **start order only**. Compose starts InfluxDB before Node-RED,
but it does not wait for InfluxDB to finish initialising — only for the
container process to exist.

For this stack it does not matter: Node-RED reconnects on its own. In a system
where startup order genuinely matters, you need a healthcheck plus
`depends_on: { condition: service_healthy }`, or retry logic in the app.

Assuming `depends_on` guarantees readiness causes intermittent failures that
only show up on slower machines. It is a classic.

## Inspecting the network

```powershell
docker network ls
docker network inspect iiot-training_iiot-net
```

The `Containers` section of the inspect output lists every attached container
with its current IP.

Note the network name: Compose prefixes it with the project name, which comes
from the `name:` field at the top of `docker-compose.yml`.

## Checkpoint

1. Your browser opens Grafana at `localhost:3000`. Inside Grafana you add an
   InfluxDB datasource. What URL do you enter, and why is it not `localhost`?
2. What would `"9999:8086"` change, and what would stay the same?
3. Why is `depends_on` alone not enough to guarantee a database is ready?

---

**Previous:** [03 — First Run](03-first-run.md) · **Next:** [05 — Volumes and Persistence](05-volumes.md)
