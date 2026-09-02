# 01 — Docker Fundamentals

## The problem Docker solves

You install an application. It needs Python 3.9, but another application on the
same machine needs Python 3.12. You install a database, and it scatters config
files across four directories you will never find again. You hand the project
to a colleague and it does not run, because their machine is not your machine.

Docker packages an application *together with its entire environment* into a
single image. That image runs identically on your laptop, your colleague's
laptop, and the production server.

## Image vs container

An **image** is a stack of read-only layers:

```
┌────────────────────────────┐
│ Layer 4: app configuration │
├────────────────────────────┤
│ Layer 3: application code  │   ← read-only
├────────────────────────────┤
│ Layer 2: Node.js runtime   │
├────────────────────────────┤
│ Layer 1: Debian base       │
└────────────────────────────┘
```

Starting a **container** adds one thin writable layer on top. Everything the
container writes lands in that layer — and it is destroyed with the container.
That is exactly why Lesson 05 on volumes exists.

Layers are also shared. If three images are all built on `debian:bookworm`,
that base layer is downloaded and stored **once**.

## Anatomy of `docker run`

```powershell
docker run -d --name my-broker -p 1883:1883 eclipse-mosquitto:2
```

| Part | Meaning |
|---|---|
| `-d` | Detached — run in the background |
| `--name my-broker` | A readable name instead of a random one |
| `-p 1883:1883` | Publish host port 1883 to container port 1883 |
| `eclipse-mosquitto:2` | The image and its tag |

Try it:

```powershell
docker run -d --name my-broker -p 1883:1883 eclipse-mosquitto:2
docker ps
docker logs my-broker
docker stop my-broker
docker rm my-broker
```

## Essential commands

| Command | What it does |
|---|---|
| `docker ps` | List running containers |
| `docker ps -a` | List all containers, including stopped ones |
| `docker images` | List downloaded images |
| `docker logs <name>` | Show a container's output |
| `docker logs -f <name>` | Follow the output live (Ctrl-C to stop watching) |
| `docker exec -it <name> sh` | Open a shell **inside** a running container |
| `docker stop` / `start` / `rm` | Stop, start, delete a container |

`docker exec` is the one to remember. When something misbehaves, being able to
step inside the container and look around turns guesswork into diagnosis.

## Why Compose

Our stack needs four containers, one network, six volumes, and about a dozen
environment variables. As `docker run` commands that is four long, fragile
lines nobody will type correctly twice.

Compose puts all of it in one declarative file:

```powershell
docker compose up -d     # create and start everything
docker compose ps        # status of this project's containers
docker compose logs -f   # follow logs from all services
docker compose down      # stop and remove containers + network
```

Compose also gives you two things `docker run` does not:

1. **A private network** created automatically, where containers find each
   other by service name
2. **Idempotency** — running `up` again only changes what actually differs

## Tags matter

```yaml
image: influxdb:2.7      # a specific minor version — predictable
image: influxdb:latest   # whatever is newest today — a moving target
```

`latest` is not a magic "always current" pointer; it is just a tag name, and it
changes under you. This course pins every image — `influxdb:2.7`, `eclipse-mosquitto:2`,
`nodered/node-red:4.1.0-22`, and `grafana/grafana:12.3.3`
because a course that breaks when an upstream release lands is a bad course.

## Checkpoint

You should now be able to answer:

1. Where does a container's written data go, and what happens to it on `docker rm`?
2. In `-p 8080:80`, which number is the host and which is the container?
3. What does `docker exec -it <name> sh` give you?

---

**Previous:** [00 — Prerequisites](00-prerequisites.md) · **Next:** [02 — The Stack](02-the-stack.md)
