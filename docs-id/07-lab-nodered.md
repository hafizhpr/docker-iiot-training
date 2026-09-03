# 07 — Lab: Node-RED

**Tujuan:** subscribe ke telemetry MQTT, ubah bentuk payload, dan tuliskan ke
InfluxDB.

Buka <http://localhost:1880>. Tidak ada login — itu adalah `allow_anonymous` yang
diterapkan ke editor, dan hal lain yang akan kamu kunci untuk penggunaan nyata.

## Langkah 1 — Instal Palette Node InfluxDB

Image `nodered/node-red` dasar tidak menyertakan node database. Instal satu:

1. Menu **☰** (kanan atas) → **Manage palette**
2. Tab **Install**
3. Cari `node-red-contrib-influxdb`
4. Klik **Install**, konfirmasi

Tunggu notifikasi sukses.

### Di Mana Itu Tersimpan?

Node diinstal ke `/data/node_modules` di dalam container — yang merupakan **named
volume** `nodered_data` dari Pelajaran 05. Buktikan ia bertahan:

```powershell
docker compose restart nodered
```

Muat ulang editor. Node InfluxDB masih ada di palette.

Ia bertahan setelah `restart`, dan bertahan setelah `down` + `up`. Ia **tidak**
bertahan setelah `down -v`, karena itu menghapus volume — setiap pelajar harus
menginstalnya lagi secara manual. Itulah masalah yang diselesaikan Pelajaran 12.

## Langkah 2 — Terima Telemetry MQTT

Seret node **mqtt in** ke canvas dan klik dua kali.

Klik ikon pensil di sebelah **Server** untuk menambahkan broker:

| Field | Nilai |
|---|---|
| Name | `mosquitto` |
| Server | `mosquitto` |
| Port | `1883` |

**Server adalah `mosquitto`, bukan `localhost`.** Node-RED adalah container;
`localhost`-nya adalah dirinya sendiri. Docker DNS me-resolve nama service.
Pelajaran 04.

Klik **Add**, kemudian selesaikan node-nya:

| Field | Nilai |
|---|---|
| Topic | `factory/+/+/telemetry/#` |
| QoS | `0` |
| Output | `auto-detect (string or buffer)` |
| Name | `telemetry in` |

Perhatikan subscription: **hanya telemetry**. Flow ini menyimpan pengukuran, jadi
tidak ada urusan menerima command. Konvensi topic dari Pelajaran 06 menjadikan itu
keputusan satu baris alih-alih masalah filtering.

Seret node **debug**, hubungkan `mqtt in` ke `debug`, dan klik **Deploy**.

Buka sidebar debug dan publish sesuatu dari PowerShell:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/temperature' -m '42.5'
```

Pesan muncul di sidebar. Node `mqtt in` seharusnya menampilkan titik hijau
**connected**. Jika bertuliskan *connecting* selamanya, kamu mengetik `localhost`.

## Langkah 3 — Ubah Bentuk Payload

MQTT memberi kita string topic dan payload teks. InfluxDB menginginkan field dan
tag. Node **function** menjembatani keduanya.

Tambahkan node function antara `mqtt in` dan `debug`, dan tempel:

```javascript
// Topic:   factory/line1/press01/telemetry/temperature
//            [0]     [1]     [2]       [3]        [4]
// Payload: "42.5"
const parts = msg.topic.split("/");

if (parts[3] !== "telemetry") {
    return null;                      // abaikan apapun yang bukan telemetry
}

const value = parseFloat(msg.payload);
if (isNaN(value)) {
    node.warn("Skipping non-numeric payload: " + msg.payload);
    return null;                      // satu publish buruk tidak boleh merusak DB
}

msg.measurement = parts[4];           // temperature
msg.payload = [
    { value: value },                      // fields - nilai yang diukur
    { line: parts[1], device: parts[2] }   // tags   - label yang diindeks
];
return msg;
```

### Field vs Tag — Keputusan yang Paling Penting

Ini adalah pilihan pemodelan yang paling berpengaruh dalam seluruh pipeline,
sehingga layak mendapat lebih dari sekadar disebutkan sekilas.

| | Tag | Field |
|---|---|---|
| Menyimpan | Label yang mengidentifikasi sumber | Nilai yang diukur |
| Diindeks | **Ya** | Tidak |
| Filter dan group by | Cepat | Lambat |
| Konten tipikal | `line`, `device`, `area`, `unit_id` | `value`, `raw`, `quality` |
| Kardinalitas | Harus tetap terbatas | Tidak terbatas boleh |

Jika kamu pernah bekerja dengan historian, pemetaannya tepat: **tag adalah metadata
aset, field adalah nilai proses**. `line1` dan `press01` mendeskripsikan *mesin mana*;
`42.5` adalah *apa yang dibacanya*.

Membalik ini — timestamp atau bacaan raw digunakan sebagai tag — dan kamu membuat
entri indeks baru untuk setiap sampel. Itulah cara klasik untuk membuat database
time-series jatuh, dan biasanya muncul berminggu-minggu kemudian.

Dua hal lagi dalam kode di atas:

- **Mengembalikan `null` membuang pesan.** Sensor yang memancarkan `ERR` tidak
  boleh menyebabkan crash flow atau menulis data sampah.
- Node `influxdb out` membaca array dua objek sebagai `[fields, tags]`.

Deploy dan publish lagi. Output debug sekarang berupa array dua elemen.

## Langkah 4 — Tulis ke InfluxDB

Seret node **influxdb out** dan hubungkan `function` ke `influxdb out`.

Klik dua kali, kemudian klik ikon pensil di sebelah **Server**:

| Field | Nilai |
|---|---|
| Version | `2.0` |
| URL | `http://influxdb:8086` |
| Token | `training-token-do-not-use-in-production` |
| Name | `influxdb` |

Sekali lagi: `influxdb`, bukan `localhost`.

Token adalah yang hardcoded di `.env`. Karena sudah ditetapkan di awal, kamu bisa
mengkonfigurasi node ini tanpa pernah membuka UI InfluxDB untuk membuat token —
itulah shortcut yang memungkinkan seluruh lab selesai dalam satu sesi.

Klik **Add**, kemudian isi node itu sendiri:

| Field | Nilai |
|---|---|
| Organization | `training` |
| Bucket | `sensors` |
| Measurement | *(biarkan kosong — `msg.measurement` yang menyediakannya)* |

**Deploy.**

## Langkah 5 — Kirim Data dan Konfirmasi

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/temperature' -m '42.5'
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/temperature' -m '43.1'
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line2/press02/telemetry/temperature' -m '38.7'
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/pressure' -m '3.2'
```

Verifikasi datanya sudah tersimpan:

```powershell
docker compose exec influxdb influx query --org training --token training-token-do-not-use-in-production 'from(bucket:\"sensors\") |> range(start:-1h)'
```

Kamu seharusnya melihat baris dengan `_measurement` berupa `temperature` dan
`pressure`, diberi tag berdasarkan `line` **dan** `device`.

## Langkah 6 — Simulasikan Sensor

Menunggu hardware nyata adalah penggunaan sesi pelatihan yang buruk. Bangun sensor
palsu sebagai gantinya — Pelajaran 08 menggantinya dengan sumber Modbus yang nyata.

Seret node **inject** dan atur:

| Field | Nilai |
|---|---|
| Repeat | `interval` |
| every | `5` seconds |

Tambahkan node **function** setelahnya:

```javascript
// Suhu acak yang bergerak sekitar 40 derajat C
const value = (38 + Math.random() * 4).toFixed(2);
msg.topic = "factory/line1/press01/telemetry/temperature";
msg.payload = value;
return msg;
```

Hubungkan ke **node function yang sama** dari Langkah 3, sehingga bacaan yang
disimulasikan mengambil jalur yang identik dengan yang nyata. Deploy.

Data sekarang tiba setiap lima detik — tepat apa yang dibutuhkan Pelajaran 09
untuk menggambar grafik yang bergerak.

## Troubleshooting

| Gejala | Penyebab |
|---|---|
| `mqtt in` stuck di *connecting* | Server diatur ke `localhost` bukan `mosquitto` |
| Tidak ada yang ditulis, tidak ada error | Periksa node debug — function mungkin mengembalikan `null` |
| `unauthorized` dari InfluxDB | Token tidak cocok dengan `INFLUXDB_TOKEN` di `.env` |
| Node `influxdb out` hilang | Instalasi palette tidak selesai, atau volume dihapus dengan `down -v` |

Node-RED menulis errornya sendiri ke log container:

```powershell
docker compose logs -f nodered
```

## Checkpoint

1. Mengapa alamat broker MQTT adalah `mosquitto` dan bukan `localhost`?
2. Flow-mu subscribe ke `factory/+/+/telemetry/#`. Seorang operator mempublish
   setpoint ke `factory/line1/press01/command/setpoint`. Apakah flow-mu melihatnya?
3. Seorang rekan menyarankan untuk menyimpan timestamp bacaan sebagai tag. Apa yang terjadi?
4. Volume mana yang menyimpan flow dan palette node yang terinstal?

---

**Sebelumnya:** [06 — Lab: MQTT](06-lab-mqtt.md) · **Selanjutnya:** [08 — Lab: Modbus to MQTT](08-lab-modbus.md)
