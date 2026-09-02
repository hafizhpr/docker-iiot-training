# Docker Training — Building an IIoT Stack

A hands-on Docker course. You will not learn Docker from slides here; you will
learn it by assembling a real Industrial IoT data pipeline out of four
containers and then breaking it on purpose to see how the pieces fit.

## What you will build

```
     PLC          broker            flow engine          database        dashboards
  Modbus TCP -->  Mosquitto  -->     Node-RED     -->    InfluxDB  -->    Grafana
    :502           :1883              :1880              :8086            :3000
                                        |
                                        +-- commands back to the PLC
```

Everything runs on one `docker compose up`.

## What you will actually learn

The stack is the excuse. The Docker concepts are the point:

| Concept | Where you meet it |
|---|---|
| Images vs containers | Lesson 01 |
| Port publishing (`host:container`) | Lesson 04 |
| User-defined bridge networks and DNS | Lesson 04 |
| Named volumes vs bind mounts | Lesson 05 |
| Environment-driven configuration | Lesson 05 |
| Reaching a service on your host machine | Lesson 04 |
| A topic convention that scales (`telemetry` / `command`) | Lesson 06 |
| Polling a PLC and scaling raw registers | Lesson 08 |
| Reading logs and shelling into containers | Lesson 10 |
| Building your own image | Lesson 12 |

## Course outline

| # | Lesson | Time |
|---|---|---|
| 00 | [Prerequisites](docs/00-prerequisites.md) | 15 min |
| 01 | [Docker Fundamentals](docs/01-docker-fundamentals.md) | 30 min |
| 02 | [The Stack](docs/02-the-stack.md) | 20 min |
| 03 | [First Run](docs/03-first-run.md) | 30 min |
| 04 | [Networking](docs/04-networking.md) | 40 min |
| 05 | [Volumes and Persistence](docs/05-volumes.md) | 40 min |
| 06 | [Lab: MQTT](docs/06-lab-mqtt.md) | 35 min |
| 07 | [Lab: Node-RED](docs/07-lab-nodered.md) | 45 min |
| 08 | [Lab: Modbus to MQTT](docs/08-lab-modbus.md) | 50 min |
| 09 | [Lab: Grafana](docs/09-lab-grafana.md) | 45 min |
| 10 | [Troubleshooting](docs/10-troubleshooting.md) | 30 min |
| 11 | [Cleanup](docs/11-cleanup.md) | 15 min |
| 12 | [Bonus: Building Custom Images](docs/12-building-custom-images.md) | 40 min |

Roughly one and a half days, or three comfortable half-days.

## Quick start

Every command in this course is **PowerShell**. Open PowerShell in this
folder first.

```powershell
docker compose up -d
docker compose ps
```

Then open:

| Service | URL | Login |
|---|---|---|
| Node-RED | http://localhost:1880 | none |
| InfluxDB | http://localhost:8086 | `admin` / `admin12345` |
| Grafana | http://localhost:3000 | `admin` / `admin` |

## Repository layout

```
.
├── docker-compose.yml          the whole stack
├── .env                        training credentials
├── mosquitto/config/
│   └── mosquitto.conf          broker configuration
├── nodered/
│   └── Dockerfile              custom image built in Lesson 12
└── docs/                       the lessons
```

## ⚠️ Security notice

This stack is configured for teaching, not for production:

- MQTT accepts **anonymous** connections from anywhere
- The InfluxDB admin token is **hardcoded** and committed to this repository
- Grafana uses **admin / admin**
- Credentials live in a **committed** `.env` file

Every one of those is a deliberate shortcut to keep the labs moving. Lesson 10
explains what you would change to make each one safe.
