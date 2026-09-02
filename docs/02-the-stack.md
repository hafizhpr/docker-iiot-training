# 02 — The Stack

## The pipeline

Every IIoT system solves the same four problems in the same order: get data off
the machines, reshape it, store it, and show it to a human.

```
  Machines  ──▶  Mosquitto  ──▶  Node-RED  ──▶  InfluxDB  ──▶  Grafana
  sensors         broker           flows          TSDB         dashboards
                  :1883            :1880          :8086        :3000

                TRANSPORT        TRANSFORM        STORE        VISUALISE
```

## The four services

### Mosquitto — the broker (port 1883)

Sensors do not talk to databases directly. They **publish** to a topic on a
broker, and anything interested **subscribes** to that topic. Publisher and
subscriber never know about each other.

```
factory/line1/press01/telemetry/temperature  ──▶  broker  ──▶  Node-RED
                                        ├─▶  an alarm service
                                        └─▶  a mobile app
```

That decoupling is the point. Adding a fifth consumer requires changing
nothing on the sensor side.

MQTT is built for constrained devices: a publish packet carries only a couple
of bytes of protocol overhead, against roughly two hundred for an equivalent
HTTP request. Over a cellular link with 5,000 sensors, that difference is the
entire bill.

### Node-RED — the flow engine (port 1880)

A browser-based editor where you wire nodes together instead of writing glue
code. In this stack it subscribes to MQTT topics, reshapes the payloads, and
writes them into InfluxDB.

Its real job is **impedance matching**. Sensors emit whatever their vendor
decided on; databases want a consistent schema. Node-RED is where messy meets
tidy.

### InfluxDB — the time-series database (port 8086)

Sensor data has a shape general-purpose databases handle badly: append-only,
timestamp-ordered, huge volume, and almost always queried as "the last N
minutes" or "the hourly average".

A time-series database is built around that shape. It also does something
PostgreSQL will not do for you out of the box: **automatically delete data
older than X**. When 500 sensors report every second, retention is not a
nice-to-have.

### Grafana — the dashboards (port 3000)

Connects to InfluxDB, draws time-series panels, and fires alerts on
thresholds. It stores no measurement data of its own — only dashboards, users,
and datasource definitions.

## Why these ports

| Service | Port | Why that number |
|---|---|---|
| Mosquitto | 1883 | The IANA-registered MQTT port |
| Mosquitto | 9001 | Convention for MQTT over WebSockets |
| Node-RED | 1880 | The project default |
| InfluxDB | 8086 | The project default |
| Grafana | 3000 | The project default |

We keep the defaults deliberately. Every tutorial, forum answer, and vendor
document assumes them, so a student who gets stuck can search and find answers
that match what is on their screen.

**Port 3000 is the one that will collide** — React, Next.js, and Rails all
claim it. Lesson 10 shows how to move it.

## Why containers suit this stack

Installing these four natively means four package managers, four service
managers, four config file locations, and four different upgrade procedures.

As containers it is one file, one command, and one uninstall that leaves
nothing behind. For a training environment that gets built and destroyed
repeatedly, that is decisive.

## Checkpoint

1. Why do sensors publish to a broker instead of writing to the database directly?
2. What does a time-series database give you that a relational one does not?
3. Which of the four services stores no measurement data at all?

---

**Previous:** [01 — Docker Fundamentals](01-docker-fundamentals.md) · **Next:** [03 — First Run](03-first-run.md)
