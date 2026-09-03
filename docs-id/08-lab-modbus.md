# 08 — Lab: Modbus ke MQTT

**Tujuan:** polling perangkat Modbus TCP, skalakan register mentah ke satuan
engineering, publish sebagai telemetry — dan terima command dari arah sebaliknya.

Pelajaran 07 mengasumsikan data sudah ada di MQTT. Di pabrik nyata, tidak seperti
itu. Data ada di PLC, dan sesuatu harus pergi mengambilnya. Sesuatu itu adalah
Node-RED, dan pelajaran ini adalah link pertama yang hilang dalam rantai:

```
PLC  --Modbus TCP-->  Node-RED  --MQTT-->  Mosquitto  -->  InfluxDB  -->  Grafana
                          |
                          +--Modbus write--  command yang datang dari MQTT
```

## Modbus dalam Satu Tabel

Kamu mungkin sudah tahu ini. Ada di sini agar field konfigurasi node masuk akal
tanpa harus keluar dari halaman.

| Tipe register | Ukuran | Akses | Baca dengan | Tulis dengan |
|---|---|---|---|---|
| Coil | 1 bit | read/write | FC 1 | FC 5 (tunggal), FC 15 (banyak) |
| Discrete input | 1 bit | read only | FC 2 | — |
| Holding register | 16 bit | read/write | FC 3 | FC 6 (tunggal), FC 16 (banyak) |
| Input register | 16 bit | read only | FC 4 | — |

Dua hal yang menyebabkan sebagian besar masalah hari pertama Modbus:

- **Holding register adalah 16 bit dan unsigned: 0 sampai 65535.** Apapun yang
  lebih besar, dan float apapun, menempati dua register atau lebih.
- **Pengalamatan berbeda satu antara gaya dokumentasi.** Manual vendor sering
  menggunakan penomoran berbasis 1 (40001 = holding register pertama) sementara
  library menggunakan offset berbasis 0. Jika nilai kamu meleset satu register,
  inilah alasannya. Kursus ini menggunakan alamat yang diharapkan node.

## Langkah 1 — Instal Palette Node Modbus

**Menu → Manage palette → Install →** cari `node-red-contrib-modbus` →
Install.

Jika instalasi gagal dengan `EBADENGINE`, image Node-RED-mu terlalu lama — lihat
[Pelajaran 10](10-troubleshooting.md). File compose dalam kursus ini sudah mematok
image dengan Node 22, sehingga seharusnya berhasil.

## Langkah 2 — Buat PLC yang Disimulasikan

Kamu tidak memerlukan hardware. `node-red-contrib-modbus` menyertakan node
**Modbus-Server**: slave Modbus TCP sungguhan, berjalan di dalam Node-RED,
didukung oleh buffer register.

Seret node **Modbus-Server** ke canvas:

| Field | Nilai |
|---|---|
| Name | `simulated PLC` |
| Host | `0.0.0.0` |
| Server Port | `10502` |

Port 10502 bukan 502 karena 502 adalah privileged dan Node-RED tidak berjalan
sebagai root.

### Beri Nilai Padanya

Node server memiliki sebuah **input**. Menyuntikkan payload yang berbentuk khusus
langsung menulis ke buffer register-nya — begitulah cara kita berpura-pura
transmitter sedang memperbarui PLC.

Seret node **inject** yang diatur untuk mengulang setiap `1` detik, kemudian node
**function** antara inject dan server:

```javascript
// Simulasikan transmitter suhu yang terhubung ke PLC.
// PLC menyimpannya sebagai raw count, bukan derajat.
// 0-100 derajat C  ->  0-27648 count  (Siemens analog scaling)
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

Hubungkan `inject` ke `function` ke `Modbus-Server` dan **Deploy**.

Nilai `register` yang valid adalah `holding`, `coils`, `input`, dan `discrete`.

### Alamat Seeding Bukan Nomor Register

Yang ini akan menghabiskan waktu satu siang jika tidak ada yang memperingatkanmu.

Node server menyimpan register-nya dalam buffer byte mentah, dan `address` dalam
pesan seeding adalah **slot buffer**, bukan nomor register Modbus. Setiap slot
adalah 8 byte, sementara register adalah 2:

| Seeding `address` | Register Modbus yang dibaca client |
|---|---|
| `0` | 0 |
| `1` | 4 |
| `2` | 8 |

Seed `address: 1` kemudian polling register 1, dan kamu membaca angka nol yang
bersih dan masuk akal — tidak ada error, tidak ada peringatan, hanya salah. Kita
menggunakan `address: 0` di seluruh lab ini karena slot 0 dan register 0 adalah
satu-satunya pasangan yang bertepatan.

Keanehan ini hanya berlaku untuk seeding simulator melalui inputnya. Write Modbus
yang sebenarnya dari client — Langkah 6 — menggunakan penomoran register normal.

Kamu sekarang memiliki slave Modbus di port 10502 dengan nilai langsung di holding
register 0. Segalanya dari sini memperlakukannya persis seperti PLC nyata.

## Langkah 3 — Polling

Seret node **Modbus-Read**. Klik ikon pensil di sebelah **Server** untuk membuat
koneksi:

| Field | Nilai |
|---|---|
| Type | `TCP` |
| Host | `localhost` |
| Port | `10502` |
| Unit-Id | `1` |

### Mengapa `localhost` Benar Di Sini — dan Hanya Di Sini

Pelajaran 04 menghabiskan seluruh bagian memperingatkan kamu tentang `localhost`.
Ini adalah satu kasus dalam kursus di mana itu benar: server Modbus dan client
Modbus adalah **container yang sama**. Node-RED berbicara ke dirinya sendiri.

Periksa terhadap tabel aturan:

| Siapa yang terhubung | Alamat |
|---|---|
| Container ke container lain | nama service |
| Container ke mesin hostmu | `host.docker.internal` |
| **Container ke dirinya sendiri** | **`localhost`** — kita di sini |

Langkah 7 menunjukkan perubahan untuk PLC nyata.

Sekarang node read itu sendiri:

| Field | Nilai |
|---|---|
| FC | `FC 3: Read Holding Registers` |
| Address | `0` |
| Quantity | `1` |
| Poll Rate | `2` seconds |
| Name | `poll press01` |

Hubungkan **output 1** dari `Modbus-Read` ke node **debug** dan Deploy.

Sidebar debug menampilkan array:

```
[ 10488 ]
```

Itu adalah raw count, bukan suhu. Output 1 membawa array data; output 2 membawa
buffer respons mentah, yang jarang kamu butuhkan.

## Langkah 4 — Skalakan ke Satuan Engineering

Raw count tidak bermakna bagi operator, dan menyimpannya membuat setiap dashboard
masa depan harus mengulangi konversi. Skalakan sekali, di sini, di edge.

Tambahkan node **function** setelah `Modbus-Read`:

```javascript
// Output 1 Modbus-Read: msg.payload adalah array nilai register
const raw = msg.payload[0];

if (raw === undefined) {
    node.warn("Empty Modbus response");
    return null;
}

// Kebalikan dari PLC scaling: 0-27648 count -> 0-100 derajat C
const degC = raw * 100 / 27648;

// Pemeriksaan plausibilitas dasar. Transmitter yang terputus sering membaca
// full scale atau nol, dan itu tidak boleh sampai ke historian.
if (degC < -10 || degC > 120) {
    node.warn("Out-of-range reading: " + degC);
    return null;
}

msg.topic = "factory/line1/press01/telemetry/temperature";
msg.payload = degC.toFixed(2);
return msg;
```

Topic mengikuti konvensi Pelajaran 06 dengan tepat. Segmen arah adalah `telemetry`,
karena ini adalah perangkat yang melapor.

## Langkah 5 — Publish ke MQTT

Seret node **mqtt out**, pilih broker `mosquitto` yang kamu konfigurasi di Pelajaran
07, dan biarkan Topic-nya **kosong** sehingga `msg.topic` dari node function yang
digunakan.

Hubungkan `function` ke `mqtt out` dan **Deploy**.

### Perhatikan Datanya Datang

```powershell
docker compose exec mosquitto mosquitto_sub -h localhost -t 'factory/+/+/telemetry/#' -v
```

Suhu yang sudah diskalakan setiap dua detik.

Dan karena Pelajaran 07 sudah subscribe ke cabang topic yang sama dan menulis ke
InfluxDB, **data Modbus-mu sekarang ada di database tanpa pekerjaan lebih lanjut.**
Konfirmasi:

```powershell
docker compose exec influxdb influx query --org training --token training-token-do-not-use-in-production 'from(bucket:\"sensors\") |> range(start:-5m) |> filter(fn:(r) => r._measurement == \"temperature\")'
```

Itulah hasil dari konvensi topic. Flow penyimpanan tidak perlu tahu bahwa perangkat
Modbus telah muncul.

## Langkah 6 — Command, Arah Sebaliknya

Telemetry mengalir dari perangkat ke sistem. Command mengalir ke arah sebaliknya,
dan cabang `command` dari pohon topic adalah tempatnya.

Bangun flow kedua: operator mempublish setpoint melalui MQTT, dan Node-RED
menulisnya ke PLC.

Node **mqtt in**:

| Field | Nilai |
|---|---|
| Topic | `factory/+/+/command/setpoint` |
| Name | `setpoint command` |

Node **function** — satuan engineering kembali ke raw count:

```javascript
// Payload datang sebagai setpoint yang mudah dibaca manusia, misalnya "45"
const degC = parseFloat(msg.payload);

// Validasi sebelum menulis ke PLC. Ini adalah pertahanan terakhir
// antara kesalahan ketik di client MQTT dan write register di mesin nyata.
if (isNaN(degC) || degC < 0 || degC > 100) {
    node.error("Rejected setpoint: " + msg.payload);
    return null;
}

// FC 6 mengharapkan satu bilangan bulat antara 0 dan 65535
msg.payload = Math.round(degC * 27648 / 100);
return msg;
```

Node **Modbus-Write**:

| Field | Nilai |
|---|---|
| Server | koneksi `localhost:10502` yang sama |
| FC | `FC 6: Preset Single Register` |
| Address | `2` |
| Unit-Id | `1` |

Hubungkan `mqtt in` ke `function` ke `Modbus-Write` dan **Deploy**.

Kirim setpoint:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/command/setpoint' -m '45'
```

Node write melaporkan sukses, dan holding register 2 sekarang berisi `12442`.

### Bentuk Payload Berdasarkan Function Code

| FC | Yang harus berupa `msg.payload` |
|---|---|
| FC 5 — Force Single Coil | `1`, `0`, `true`, atau `false` |
| FC 6 — Preset Single Register | satu angka, 0 sampai 65535 |
| FC 15 — Force Multiple Coils | array boolean |
| FC 16 — Preset Multiple Registers | array angka, masing-masing 0 sampai 65535 |

Kirim float atau angka negatif ke FC 6 dan write akan gagal. Pembulatan dan
pengecekan rentang di node function bukan opsional.

### Mengapa Validasi di Flow

Broker MQTT menerima publish anonim. Siapapun yang bisa menjangkau port 1883 bisa
mengirim setpoint. Blok `if` di atas adalah satu-satunya yang berdiri antara
payload yang salah ketik dan write register di mesin nyata.

Di pabrik, kamu juga akan memastikan bahwa command hanya menjangkau perangkat yang
dituju — topic berisi `line1/press01`, dan flow harus memverifikasi bahwa itu cocok
dengan PLC yang akan ditulisnya.

## Langkah 7 — Mengarahkan ke PLC Nyata

Semua di atas bekerja terhadap perangkat nyata dengan satu perubahan: alamat koneksi.

| Di mana PLC-mu berada | Host | Port |
|---|---|---|
| Disimulasikan di dalam Node-RED | `localhost` | `10502` |
| Simulator atau gateway di PC-mu | `host.docker.internal` | `502` |
| PLC di jaringan pabrik | IP-nya, misalnya `192.168.1.50` | `502` |

Untuk kasus tengah, `extra_hosts` di `docker-compose.yml` adalah yang membuat nama
ter-resolve — Pelajaran 04 membahasnya, beserta dua alasan umum mengapa koneksi
host ditolak.

Untuk kasus ketiga, container menjangkau jaringan pabrik melalui routing mesinmu.
Jika PC-mu bisa menjangkau PLC, container juga bisa.

### Nilai 32-bit dan Word Order

Satu holding register tidak bisa menampung float atau nilai di atas 65535. Nilai
itu menempati **dua register berurutan**, dan vendor tidak sepakat tentang mana yang
lebih dulu.

Baca dua register alih-alih satu dan, jika angkanya tidak masuk akal, tukarkan:

```javascript
// msg.payload = [reg0, reg1]
const buf = Buffer.alloc(4);

// Big-endian word order (pilihan yang lebih umum)
buf.writeUInt16BE(msg.payload[0], 0);
buf.writeUInt16BE(msg.payload[1], 2);

// Jika nilainya terlihat salah, tukar dua write di atas.
msg.payload = buf.readFloatBE(0);
return msg;
```

Tidak ada cara untuk mendeteksi urutan yang benar dari protokolnya sendiri. Periksa
manual vendor, atau coba keduanya dan pertahankan yang menghasilkan angka yang masuk akal.

### Laju Polling

| Laju | Gunakan untuk |
|---|---|
| 100–500 ms | Nilai proses cepat yang ingin kamu tren secara dekat |
| 1–5 s | Kebanyakan suhu, tekanan, level |
| 30 s ke atas | Counter, total, status |

Setiap polling adalah request yang harus dijawab PLC sementara ia juga sedang
menjalankan mesin. Poll hanya apa yang kamu butuhkan, seperlahan yang bisa kamu
terima. Sepuluh perangkat di 100 ms adalah seribu request per detik, dan PLC
bukan satu-satunya yang menderita — setiap bacaan juga menjadi baris di InfluxDB.

## Troubleshooting

| Gejala | Penyebab |
|---|---|
| Node Read menampilkan `disconnected` | Host atau port salah. Di dalam container yang sama itu adalah `localhost:10502` |
| `ECONNREFUSED` ke PLC host | Gunakan `host.docker.internal`, dan periksa apakah PLC bind ke interface yang bisa dijangkau |
| Nilai meleset satu register | Pengalamatan manual berbasis 1 vs node berbasis 0 |
| Simulator membaca 0 tapi tidak ada error | Di-seed ke slot buffer selain `0` — lihat Langkah 2 |
| Nilai bergantian antara masuk akal dan absurd | Nilai 32-bit dibaca sebagai 16-bit, atau word order salah |
| Bacaan terpaku di 0 atau full scale | Kerusakan transmitter, atau kamu skalakan terhadap rentang raw yang salah |
| Write ditolak | FC 6 dikirim float, negatif, atau nilai di atas 65535 |

Node-RED melaporkan error Modbus di log container:

```powershell
docker compose logs -f nodered
```

## Apa yang Kamu Bangun

```
inject --> function --> Modbus-Server            (PLC yang disimulasikan)
                             |
         Modbus-Read <-------+
              |
              v
         function (scale) --> mqtt out --> factory/line1/press01/telemetry/temperature
                                                   |
                                                   v
                                          Flow Pelajaran 07 --> InfluxDB

mqtt in factory/+/+/command/setpoint --> function (validate) --> Modbus-Write
```

Telemetry naik, command turun, keduanya dinamai oleh konvensi yang sama. Itulah
edge gateway yang berfungsi — bagian yang menghubungkan lantai pabrik dengan
segalanya dalam kursus ini.

## Checkpoint

1. Mengapa `localhost` adalah host Modbus yang benar dalam lab ini, padahal Pelajaran
   04 menghabiskan satu bagian memperingatkanmu tentangnya?
2. PLC-mu menyimpan suhu sebagai 0–27648 count. Di mana konversi ke derajat
   seharusnya terjadi, dan mengapa tidak di Grafana?
3. Seorang operator mempublish `-5` sebagai setpoint. Apa yang mencegahnya sampai ke PLC?
4. Total flow 32-bit bergantian antara nilai yang masuk akal dan nilai yang sangat besar.
   Apa yang pertama kali kamu periksa?
5. Ke cabang topic mana sensor seharusnya diizinkan publish, dan mana yang harus ditolak?

---

**Sebelumnya:** [07 — Lab: Node-RED](07-lab-nodered.md) · **Selanjutnya:** [09 — Lab: Grafana](09-lab-grafana.md)
