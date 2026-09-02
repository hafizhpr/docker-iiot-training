# 06 — Lab: MQTT with Mosquitto

**Goal:** publish and subscribe to messages, learn the topic convention this
course uses, and understand why `mosquitto.conf` exists at all.

All commands in this course are **PowerShell**. Open PowerShell in the project
folder before you start.

## Start the stack

```powershell
docker compose up -d
docker compose ps
```

All four containers should show `Up`.

## Why the config file is mandatory

Mosquitto 2.x changed a default that surprises almost everyone: **without a
listener declaration it binds only to loopback inside the container**. The
container starts, the logs look clean, port 1883 is published — and every
client is refused.

Our `mosquitto/config/mosquitto.conf` fixes that:

```conf
listener 1883
allow_anonymous true
```

| Line | Effect |
|---|---|
| `listener 1883` | Accept connections on 1883 from any interface |
| `allow_anonymous true` | No username or password required |

⚠️ `allow_anonymous true` means anyone who can reach port 1883 can publish
anything. Acceptable in a classroom on your own machine, never acceptable on a
plant network.

The file is a **bind mount**, so you edit it on your host and restart the
broker to apply changes:

```powershell
docker compose restart mosquitto
docker compose logs mosquitto
```

## The topic convention

Before publishing anything, agree on the shape of your topics. This course uses
one convention everywhere:

```
factory / <line> / <device> / telemetry / <measurement>     device  →  system
factory / <line> / <device> / command   / <action>          system  →  device
```

| Segment | Meaning | Example |
|---|---|---|
| `factory` | Site root | `factory` |
| `<line>` | Production line or area | `line1` |
| `<device>` | The equipment itself | `press01` |
| `telemetry` \| `command` | **Direction of travel** | see below |
| `<measurement>` / `<action>` | What is being reported or requested | `temperature`, `setpoint` |

### Why the direction segment matters

This is the part worth slowing down for.

| Branch | Who publishes | Who subscribes | Meaning |
|---|---|---|---|
| `telemetry` | The device | Historian, dashboard, alarms | "here is what I measured" |
| `command` | The control system | The device | "please do this" |

Anyone reading a topic knows immediately which way the data flows. More
practically, it makes access control possible later: a sensor gets permission
to publish only under `telemetry`, and never to `command`. Without a direction
segment you cannot express that rule at all.

If you have worked with a SCADA tag database, this is the same discipline as
separating read tags from write tags — enforced by the topic name instead of by
convention in someone's head.

### Examples

```
factory/line1/press01/telemetry/temperature
factory/line1/press01/telemetry/pressure
factory/line2/press02/telemetry/temperature
factory/line1/press01/command/setpoint
factory/line1/press01/command/reset
```

### Subscribing to slices

| Subscription | Gets |
|---|---|
| `factory/#` | Everything |
| `factory/+/+/telemetry/#` | All telemetry from every device |
| `factory/line1/+/telemetry/#` | All telemetry from line 1 |
| `factory/+/+/telemetry/temperature` | Every temperature at the site |
| `factory/line1/press01/command/#` | Every command aimed at one press |

Getting this wrong is expensive to fix later, because every device has to be
reconfigured. Spend time on it up front.

## Subscribe

The Mosquitto image ships the `mosquitto_sub` and `mosquitto_pub` client tools,
so no extra install is needed.

Subscribe to all telemetry:

```powershell
docker compose exec mosquitto mosquitto_sub -h localhost -t 'factory/+/+/telemetry/#' -v
```

This blocks and waits. Leave it running.

> **Running this lab a second time?** Messages may appear the instant you
> subscribe, before you have published anything. Those are *retained* messages
> left over from your previous run — the broker keeps them in the
> `mosquitto_data` volume, so neither `docker compose restart` nor
> `docker compose down` clears them. Nothing is broken. The retained-message
> section below explains what they are; Lesson 11 shows how to wipe them.

> `-h localhost` is correct **here** because the command runs inside the
> mosquitto container. From any other container it would be `-h mosquitto`.
> Lesson 04 covers why.

> Topics are wrapped in **single quotes** throughout this course. PowerShell
> does not mangle a `#` in the middle of a word, so an unquoted
> `-t factory/#` happens to work — but quote them anyway. It costs nothing, and
> it saves you the moment a topic contains a space or a `$`.

## Publish

Open a **second PowerShell window** in the same folder:

```powershell
cd C:\Users\<you>\projects\docker-iiot-training
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/temperature' -m '42.5'
```

The first window prints:

```
factory/line1/press01/telemetry/temperature 42.5
```

Send a few more:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/pressure' -m '3.2'
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line2/press02/telemetry/temperature' -m '38.1'
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/command/setpoint' -m '45'
```

The first two arrive. The **command** does not — your subscription covers only
the `telemetry` branch. That is the direction segment doing its job.

Watch the command branch instead, in the second window:

```powershell
docker compose exec mosquitto mosquitto_sub -h localhost -t 'factory/+/+/command/#' -v -C 1 -W 20
```

Then publish the setpoint again from the first window. It arrives.

Stop a subscriber with Ctrl-C.

## Topic wildcards

| Wildcard | Matches | Example |
|---|---|---|
| `+` | Exactly one level | `factory/+/+/telemetry/temperature` |
| `#` | All remaining levels | `factory/line1/#` |

`#` must be the last character of the topic. `factory/#/telemetry` is invalid.

`+` is what lets you write `factory/+/+/telemetry/#` and pick up every device
on every line without listing them.

## Retained messages

A retained message is stored by the broker and delivered immediately to any
future subscriber. Publish one with `-r`:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/status' -m 'running' -r
```

Now subscribe *after* the fact — a single window is enough, no second terminal:

```powershell
docker compose exec mosquitto mosquitto_sub -h localhost -t 'factory/line1/press01/telemetry/status' -v -C 1 -W 5
```

You receive `running` instantly, even though it was published earlier. Without
`-r` you would wait for the next publish and time out.

This is how a dashboard shows current machine state the moment it connects —
the same idea as a SCADA tag holding its last known value.

Retained messages are written to `mosquitto_data`, so they survive a restart:

```powershell
docker compose restart mosquitto
docker compose exec mosquitto mosquitto_sub -h localhost -t 'factory/line1/press01/telemetry/status' -v -C 1 -W 5
```

Still there. Volumes and MQTT semantics meeting in one place.

## Useful flags

| Flag | Meaning |
|---|---|
| `-v` | Print the topic alongside the payload |
| `-C 1` | Exit after receiving 1 message |
| `-W 10` | Give up after 10 seconds |
| `-r` | Publish as retained |
| `-q 1` | QoS 1 — at least once delivery |

`-C 1 -W 5` together are the reliable way to test a subscription from a single
PowerShell window: take one message, then give up after five seconds.

## Checkpoint

1. Why does Mosquitto 2.x need `listener 1883` in a config file?
2. A pressure sensor and the operator HMI both talk to `press01`. Write the
   topic each of them publishes to.
3. You have one PowerShell window and want to prove a retained message is
   there. Which two flags do you need?
4. Which volume keeps retained messages alive across a restart?

---

**Previous:** [05 — Volumes and Persistence](05-volumes.md) · **Next:** [07 — Lab: Node-RED](07-lab-nodered.md)
