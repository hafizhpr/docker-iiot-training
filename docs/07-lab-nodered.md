# 07 — Lab: Node-RED

**Goal:** subscribe to MQTT telemetry, reshape the payload, and write it into
InfluxDB.

Open <http://localhost:1880>. No login — that is `allow_anonymous` thinking
applied to the editor, and another thing you would lock down for real.

## Step 1 — Install the InfluxDB palette node

The base `nodered/node-red` image does not include database nodes. Install one:

1. Menu **☰** (top right) → **Manage palette**
2. **Install** tab
3. Search `node-red-contrib-influxdb`
4. Click **Install**, confirm

Wait for the success notification.

### Where did that go?

The node was installed into `/data/node_modules` inside the container — which
is the `nodered_data` **named volume** from Lesson 05. Prove it survives:

```powershell
docker compose restart nodered
```

Reload the editor. The InfluxDB nodes are still in the palette.

It survives `restart`, and it survives `down` + `up`. It does **not** survive
`down -v`, because that deletes the volume — every student would have to
install it again by hand. That is the problem Lesson 12 solves.

## Step 2 — Receive MQTT telemetry

Drag an **mqtt in** node onto the canvas and double-click it.

Click the pencil next to **Server** to add a broker:

| Field | Value |
|---|---|
| Name | `mosquitto` |
| Server | `mosquitto` |
| Port | `1883` |

**Server is `mosquitto`, not `localhost`.** Node-RED is a container; its
`localhost` is itself. Docker DNS resolves the service name. Lesson 04.

Click **Add**, then finish the node:

| Field | Value |
|---|---|
| Topic | `factory/+/+/telemetry/#` |
| QoS | `0` |
| Output | `auto-detect (string or buffer)` |
| Name | `telemetry in` |

Note the subscription: **telemetry only**. This flow stores measurements, so it
has no business receiving commands. The topic convention from Lesson 06 makes
that a one-line decision instead of a filtering problem.

Drag a **debug** node, wire `mqtt in` to `debug`, and click **Deploy**.

Open the debug sidebar and publish something from PowerShell:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/temperature' -m '42.5'
```

The message appears in the sidebar. The `mqtt in` node should show a green
**connected** dot. If it says *connecting* forever, you typed `localhost`.

## Step 3 — Reshape the payload

MQTT gives us a topic string and a text payload. InfluxDB wants fields and
tags. A **function** node bridges the two.

Add a function node between `mqtt in` and `debug`, and paste:

```javascript
// Topic:   factory/line1/press01/telemetry/temperature
//            [0]     [1]     [2]       [3]        [4]
// Payload: "42.5"
const parts = msg.topic.split("/");

if (parts[3] !== "telemetry") {
    return null;                      // ignore anything that is not telemetry
}

const value = parseFloat(msg.payload);
if (isNaN(value)) {
    node.warn("Skipping non-numeric payload: " + msg.payload);
    return null;                      // one bad publish must not poison the DB
}

msg.measurement = parts[4];           // temperature
msg.payload = [
    { value: value },                      // fields - the measured values
    { line: parts[1], device: parts[2] }   // tags   - indexed labels
];
return msg;
```

### Fields vs tags — the decision that matters most

This is the single most consequential modelling choice in the whole pipeline,
so it is worth more than a passing mention.

| | Tags | Fields |
|---|---|---|
| Holds | Labels that identify the source | The measured value |
| Indexed | **Yes** | No |
| Filter and group by | Fast | Slow |
| Typical content | `line`, `device`, `area`, `unit_id` | `value`, `raw`, `quality` |
| Cardinality | Must stay bounded | Unbounded is fine |

If you have worked with a historian, the mapping is exact: **tags are the asset
metadata, fields are the process value**. `line1` and `press01` describe *which*
machine; `42.5` is *what it read*.

Get it backwards — a timestamp or a raw reading used as a tag — and you create a
new index entry for every sample. That is the classic way to make a time-series
database fall over, and it usually shows up weeks later.

Two more things in the code above:

- **Returning `null` drops the message.** A sensor that emits `ERR` should not
  crash the flow or write garbage.
- The `influxdb out` node reads an array of two objects as `[fields, tags]`.

Deploy and publish again. The debug output is now a two-element array.

## Step 4 — Write to InfluxDB

Drag an **influxdb out** node and wire `function` to `influxdb out`.

Double-click it, then the pencil next to **Server**:

| Field | Value |
|---|---|
| Version | `2.0` |
| URL | `http://influxdb:8086` |
| Token | `training-token-do-not-use-in-production` |
| Name | `influxdb` |

Again: `influxdb`, not `localhost`.

The token is the one hardcoded in `.env`. Because it was fixed in advance, you
can configure this node without ever opening the InfluxDB UI to generate one —
that shortcut is why the whole lab fits in one session.

Click **Add**, then fill in the node itself:

| Field | Value |
|---|---|
| Organization | `training` |
| Bucket | `sensors` |
| Measurement | *(leave empty — `msg.measurement` supplies it)* |

**Deploy.**

## Step 5 — Send data and confirm

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/temperature' -m '42.5'
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/temperature' -m '43.1'
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line2/press02/telemetry/temperature' -m '38.7'
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/pressure' -m '3.2'
```

Verify it landed:

```powershell
docker compose exec influxdb influx query --org training --token training-token-do-not-use-in-production 'from(bucket:\"sensors\") |> range(start:-1h)'
```

You should see rows with `_measurement` of `temperature` and `pressure`, tagged
by `line` **and** `device`.

## Step 6 — Simulate a sensor

Waiting for real hardware is a poor use of a training session. Build a fake
sensor instead — Lesson 08 replaces it with a real Modbus source.

Drag an **inject** node and set:

| Field | Value |
|---|---|
| Repeat | `interval` |
| every | `5` seconds |

Add a **function** node after it:

```javascript
// Random temperature drifting around 40 degrees C
const value = (38 + Math.random() * 4).toFixed(2);
msg.topic = "factory/line1/press01/telemetry/temperature";
msg.payload = value;
return msg;
```

Wire it into the **same function node** from Step 3, so simulated readings take
the identical path as real ones. Deploy.

Data now arrives every five seconds — exactly what Lesson 09 needs to draw a
moving chart.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `mqtt in` stuck on *connecting* | Server set to `localhost` instead of `mosquitto` |
| Nothing written, no error | Check the debug node — the function may be returning `null` |
| `unauthorized` from InfluxDB | Token does not match `INFLUXDB_TOKEN` in `.env` |
| `influxdb out` node missing | Palette install did not finish, or the volume was wiped with `down -v` |

Node-RED writes its own errors to the container log:

```powershell
docker compose logs -f nodered
```

## Checkpoint

1. Why is the MQTT broker address `mosquitto` and not `localhost`?
2. Your flow subscribes to `factory/+/+/telemetry/#`. An operator publishes a
   setpoint to `factory/line1/press01/command/setpoint`. Does your flow see it?
3. A colleague suggests storing the reading timestamp as a tag. What happens?
4. Which volume holds your flows and installed palette nodes?

---

**Previous:** [06 — Lab: MQTT](06-lab-mqtt.md) · **Next:** [08 — Lab: Modbus to MQTT](08-lab-modbus.md)
