# 08 — Lab: Modbus to MQTT

**Goal:** poll a Modbus TCP device, scale the raw registers into engineering
units, publish them as telemetry — and accept commands back the other way.

Lesson 07 assumed the data was already in MQTT. On a real plant it is not. It
sits in a PLC, and something has to go and fetch it. That something is
Node-RED, and this lesson is the missing first link in the chain:

```
PLC  --Modbus TCP-->  Node-RED  --MQTT-->  Mosquitto  -->  InfluxDB  -->  Grafana
                          |
                          +--Modbus write--  commands arriving from MQTT
```

## Modbus in one table

You may know this already. It is here so the node configuration fields make
sense without leaving the page.

| Register type | Size | Access | Read with | Write with |
|---|---|---|---|---|
| Coil | 1 bit | read/write | FC 1 | FC 5 (single), FC 15 (multiple) |
| Discrete input | 1 bit | read only | FC 2 | — |
| Holding register | 16 bit | read/write | FC 3 | FC 6 (single), FC 16 (multiple) |
| Input register | 16 bit | read only | FC 4 | — |

Two things that cause most first-day Modbus grief:

- **A holding register is 16 bits and unsigned: 0 to 65535.** Anything larger,
  and any float, occupies two or more registers.
- **Addressing is off by one between documentation styles.** Vendor manuals
  often use 1-based numbering (40001 = first holding register) while libraries
  use 0-based offsets. If your value is one register out, this is why. This
  course uses the address the node expects.

## Step 1 — Install the Modbus palette node

**Menu → Manage palette → Install →** search `node-red-contrib-modbus` →
Install.

If the install fails with `EBADENGINE`, your Node-RED image is too old — see
[Lesson 10](10-troubleshooting.md). The compose file in this course already
pins an image with Node 22, so it should succeed.

## Step 2 — Stand up a simulated PLC

You do not need hardware. `node-red-contrib-modbus` includes a
**Modbus-Server** node: a real Modbus TCP slave, running inside Node-RED,
backed by a register buffer.

Drag a **Modbus-Server** node onto the canvas:

| Field | Value |
|---|---|
| Name | `simulated PLC` |
| Host | `0.0.0.0` |
| Server Port | `10502` |

Port 10502 rather than 502 because 502 is privileged and Node-RED does not run
as root.

### Feed it a value

The server node has an **input**. Injecting a specially shaped payload writes
straight into its register buffer — that is how we pretend a transmitter is
updating the PLC.

Drag an **inject** node set to repeat every `1` second, then a **function**
node between it and the server:

```javascript
// Simulate a temperature transmitter wired to the PLC.
// The PLC stores it as raw counts, not degrees.
// 0-100 degrees C  ->  0-27648 counts  (Siemens analog scaling)
const degC = 38 + Math.random() * 4;
const raw = Math.round(degC * 27648 / 100);

msg.payload = {
    value: raw,
    register: "holding",
    address: 0,
    disableMsgOutput: 1
};
return msg;
```

Wire `inject` to `function` to `Modbus-Server` and **Deploy**.

Valid `register` values are `holding`, `coils`, `input`, and `discrete`.

### The seeding address is not the register number

This one will cost you an afternoon if nobody warns you.

The server node stores its registers in a raw byte buffer, and the `address`
in the seeding message is a **buffer slot**, not a Modbus register number. Each
slot is 8 bytes, while a register is 2:

| Seeding `address` | Modbus register a client reads |
|---|---|
| `0` | 0 |
| `1` | 4 |
| `2` | 8 |

Seed `address: 1` and then poll register 1, and you read a clean, believable
zero — no error, no warning, just wrong. We use `address: 0` throughout this
lab because slot 0 and register 0 are the only pair that coincide.

This quirk applies **only** to seeding the simulator through its input. A real
Modbus write from a client — Step 6 — uses normal register numbering.

You now have a Modbus slave on port 10502 with a live value in holding
register 0. Everything from here treats it exactly like a real PLC.

## Step 3 — Poll it

Drag a **Modbus-Read** node. Click the pencil next to **Server** to create the
connection:

| Field | Value |
|---|---|
| Type | `TCP` |
| Host | `localhost` |
| Port | `10502` |
| Unit-Id | `1` |

### Why `localhost` is right here — and only here

Lesson 04 spent a whole section warning you off `localhost`. This is the one
case in the course where it is correct: the Modbus server and the Modbus client
are **the same container**. Node-RED is talking to itself.

Check it against the rule table:

| Who is connecting | Address |
|---|---|
| A container to another container | service name |
| A container to your host machine | `host.docker.internal` |
| **A container to itself** | **`localhost`** — we are here |

Step 7 shows the change for a real PLC.

Now the read node itself:

| Field | Value |
|---|---|
| FC | `FC 3: Read Holding Registers` |
| Address | `0` |
| Quantity | `1` |
| Poll Rate | `2` seconds |
| Name | `poll press01` |

Wire **output 1** of `Modbus-Read` to a **debug** node and Deploy.

The debug sidebar shows an array:

```
[ 10488 ]
```

That is the raw count, not a temperature. Output 1 carries the data array;
output 2 carries the raw response buffer, which you rarely need.

## Step 4 — Scale to engineering units

Raw counts mean nothing to an operator, and storing them makes every future
dashboard repeat the conversion. Scale once, here, at the edge.

Add a **function** node after `Modbus-Read`:

```javascript
// Modbus-Read output 1: msg.payload is an array of register values
const raw = msg.payload[0];

if (raw === undefined) {
    node.warn("Empty Modbus response");
    return null;
}

// Inverse of the PLC scaling: 0-27648 counts -> 0-100 degrees C
const degC = raw * 100 / 27648;

// Basic plausibility check. A disconnected transmitter often reads
// full scale or zero, and that should not reach the historian.
if (degC < -10 || degC > 120) {
    node.warn("Out-of-range reading: " + degC);
    return null;
}

msg.topic = "factory/line1/press01/telemetry/temperature";
msg.payload = degC.toFixed(2);
return msg;
```

The topic follows the Lesson 06 convention exactly. The direction segment is
`telemetry`, because this is the device reporting.

## Step 5 — Publish to MQTT

Drag an **mqtt out** node, select the `mosquitto` broker you configured in
Lesson 07, and leave its Topic **empty** so `msg.topic` from the function node
is used.

Wire `function` to `mqtt out` and **Deploy**.

### Watch it arrive

```powershell
docker compose exec mosquitto mosquitto_sub -h localhost -t 'factory/+/+/telemetry/#' -v
```

A scaled temperature every two seconds.

And because Lesson 07 already subscribes to that same topic branch and writes
to InfluxDB, **your Modbus data is now in the database with no further work.**
Confirm it:

```powershell
docker compose exec influxdb influx query --org training --token training-token-do-not-use-in-production 'from(bucket:\"sensors\") |> range(start:-5m) |> filter(fn:(r) => r._measurement == \"temperature\")'
```

That is the payoff of a topic convention. The storage flow never needed to know
a Modbus device had appeared.

## Step 6 — Commands, the other direction

Telemetry flows device to system. Commands flow the other way, and the
`command` branch of the topic tree is where they live.

Build a second flow: an operator publishes a setpoint over MQTT, and Node-RED
writes it into the PLC.

**mqtt in** node:

| Field | Value |
|---|---|
| Topic | `factory/+/+/command/setpoint` |
| Name | `setpoint command` |

**function** node — engineering units back to raw counts:

```javascript
// Payload arrives as a human-readable setpoint, e.g. "45"
const degC = parseFloat(msg.payload);

// Validate before writing to a PLC. This is the last line of defence
// between a typo in an MQTT client and a real machine.
if (isNaN(degC) || degC < 0 || degC > 100) {
    node.error("Rejected setpoint: " + msg.payload);
    return null;
}

// FC 6 expects a single integer between 0 and 65535
msg.payload = Math.round(degC * 27648 / 100);
return msg;
```

**Modbus-Write** node:

| Field | Value |
|---|---|
| Server | the same `localhost:10502` connection |
| FC | `FC 6: Preset Single Register` |
| Address | `2` |
| Unit-Id | `1` |

Wire `mqtt in` to `function` to `Modbus-Write` and **Deploy**.

Send a setpoint:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/command/setpoint' -m '45'
```

The write node reports success, and holding register 2 now holds `12442`.

### Payload shapes by function code

| FC | What `msg.payload` must be |
|---|---|
| FC 5 — Force Single Coil | `1`, `0`, `true`, or `false` |
| FC 6 — Preset Single Register | one number, 0 to 65535 |
| FC 15 — Force Multiple Coils | an array of booleans |
| FC 16 — Preset Multiple Registers | an array of numbers, each 0 to 65535 |

Send a float or a negative number to FC 6 and the write fails. Rounding and
range-checking in the function node is not optional.

### Why validate in the flow

The MQTT broker accepts anonymous publishes. Anyone who can reach port 1883 can
send a setpoint. The `if` block above is the only thing standing between a
mistyped payload and a register write on a real machine.

On a plant you would also enforce that a command reaches only the device it was
addressed to — the topic contains `line1/press01`, and the flow should check
that it matches the PLC it is about to write to.

## Step 7 — Pointing at a real PLC

Everything above works against a real device with one change: the connection
address.

| Where your PLC lives | Host | Port |
|---|---|---|
| Simulated inside Node-RED | `localhost` | `10502` |
| A simulator or gateway on your PC | `host.docker.internal` | `502` |
| A PLC on the plant network | its IP, e.g. `192.168.1.50` | `502` |

For the middle case, `extra_hosts` in `docker-compose.yml` is what makes the
name resolve — Lesson 04 covers it, along with the two usual reasons a host
connection is refused.

For the third case, the container reaches the plant network through your
machine's routing. If your PC can reach the PLC, so can the container.

### 32-bit values and word order

A single holding register cannot hold a float or a value above 65535. Those
occupy **two consecutive registers**, and vendors disagree about which half
comes first.

Read two registers instead of one and, if the number is nonsense, swap them:

```javascript
// msg.payload = [reg0, reg1]
const buf = Buffer.alloc(4);

// Big-endian word order (the more common choice)
buf.writeUInt16BE(msg.payload[0], 0);
buf.writeUInt16BE(msg.payload[1], 2);

// If the value looks wrong, swap the two writes above.
msg.payload = buf.readFloatBE(0);
return msg;
```

There is no way to detect the right order from the protocol itself. Check the
vendor manual, or try both and keep the one that produces a plausible number.

### Polling rate

| Rate | Use for |
|---|---|
| 100–500 ms | Fast process values you intend to trend closely |
| 1–5 s | Most temperatures, pressures, levels |
| 30 s and slower | Counters, totals, status |

Every poll is a request the PLC must answer while it is also running the
machine. Poll only what you need, as slowly as you can accept. Ten devices at
100 ms is a thousand requests per second, and the PLC is not the only thing
that suffers — every reading also becomes a row in InfluxDB.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Read node shows `disconnected` | Wrong host or port. Inside the same container it is `localhost:10502` |
| `ECONNREFUSED` to a host PLC | Use `host.docker.internal`, and check the PLC binds a reachable interface |
| Values are one register off | 1-based manual vs 0-based node addressing |
| Simulator reads 0 but no error | Seeded to a buffer slot other than `0` — see Step 2 |
| Value alternates plausible and absurd | 32-bit value read as 16-bit, or the wrong word order |
| Reading pinned at 0 or full scale | Transmitter fault, or you scaled against the wrong raw range |
| Write rejected | FC 6 was sent a float, a negative, or a value above 65535 |

Node-RED reports Modbus errors in the container log:

```powershell
docker compose logs -f nodered
```

## What you built

```
inject --> function --> Modbus-Server            (the simulated PLC)
                             |
         Modbus-Read <-------+
              |
              v
         function (scale) --> mqtt out --> factory/line1/press01/telemetry/temperature
                                                   |
                                                   v
                                          Lesson 07 flow --> InfluxDB

mqtt in factory/+/+/command/setpoint --> function (validate) --> Modbus-Write
```

Telemetry up, commands down, both named by the same convention. That is a
working edge gateway — the piece that connects the plant floor to everything
else in this course.

## Checkpoint

1. Why is `localhost` the correct Modbus host in this lab, when Lesson 04 spent
   a section warning you against it?
2. Your PLC holds a temperature as 0–27648 counts. Where should the conversion
   to degrees happen, and why not in Grafana?
3. An operator publishes `-5` as a setpoint. What stops it reaching the PLC?
4. A 32-bit flow total alternates between a sensible value and a huge one. What
   is the first thing you check?
5. Which topic branch should a sensor be allowed to publish to, and which should
   it be denied?

---

**Previous:** [07 — Lab: Node-RED](07-lab-nodered.md) · **Next:** [09 — Lab: Grafana](09-lab-grafana.md)
