# 09 — Lab: Grafana

**Tujuan:** hubungkan Grafana ke InfluxDB dan bangun dashboard langsung.

Pastikan data masih mengalir — baik dari sensor inject-node di Pelajaran 07
maupun polling Modbus dari Pelajaran 08. Topic-nya sama, jadi pelajaran ini tidak
peduli mana yang memberinya makan.

## Langkah 1 — Login

Buka <http://localhost:3000>.

| Field | Nilai |
|---|---|
| Username | `admin` |
| Password | `admin` |

Grafana mungkin menawarkan untuk mengubah password — **Skip** baik-baik saja di sini.

Kredensial itu berasal dari environment, bukan dari wizard setup manual:

```yaml
environment:
  GF_SECURITY_ADMIN_USER: ${GRAFANA_USER}
  GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
```

Setiap pengaturan Grafana bisa dikendalikan dengan cara ini. Polanya adalah
`GF_<SECTION>_<KEY>` dalam huruf kapital — `GF_SERVER_HTTP_PORT`,
`GF_USERS_ALLOW_SIGN_UP`, dan seterusnya. Inilah cara Grafana dikonfigurasi di
produksi juga; satu-satunya yang salah dengan milik kita adalah nilainya `admin`.

## Langkah 2 — Tambah Datasource InfluxDB

**☰ → Connections → Data sources → Add new data source → InfluxDB**

| Field | Nilai |
|---|---|
| Name | `InfluxDB` |
| Query language | **Flux** |
| URL | `http://influxdb:8086` |
| Auth: Basic auth | off |
| Organization | `training` |
| Token | `training-token-do-not-use-in-production` |
| Default Bucket | `sensors` |

**URL adalah `http://influxdb:8086`.** Kamu mengakses Grafana di `localhost:3000`
di browsermu, tapi Grafana itu sendiri adalah container — `localhost`-nya adalah
dirinya sendiri. Ini adalah kali ketiga aturan yang sama menentukan hasilnya.
Ini tidak akan menjadi yang terakhir.

Klik **Save & test**. Harapkan `datasource is working`.

Jika kamu mendapat connection error, jalankan ini untuk mengkonfirmasi bahwa jalur
yang digunakan Grafana benar-benar terbuka:

```powershell
docker compose exec grafana wget -qO- http://influxdb:8086/health
```

JSON kembali berarti network baik-baik saja dan masalahnya ada di token atau nama
organisasi.

## Langkah 3 — Bangun Panel

**☰ → Dashboards → New → New dashboard → Add visualization →** pilih `InfluxDB`.

Alihkan editor query ke mode teks **Flux** dan masukkan:

```flux
from(bucket: "sensors")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "temperature")
  |> filter(fn: (r) => r._field == "value")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

Membacanya baris demi baris:

| Baris | Arti |
|---|---|
| `from(bucket:)` | Bucket mana yang dibaca |
| `range(...)` | Jendela waktu — `v.timeRangeStart` mengikuti pemilih dashboard |
| `filter(_measurement)` | Hanya baris temperature |
| `filter(_field)` | Hanya field `value` |
| `aggregateWindow(...)` | Rata-rata per interval, agar jendela yang lebar tetap terbaca |

Variabel `v.` disediakan oleh Grafana. Menggunakannya adalah yang membuat pemilih
waktu dan zoom dashboard berfungsi.

Atur rentang waktu ke **Last 15 minutes**, dan auto-refresh (kanan atas) ke
**5s**. Beri panel sebuah judul, kemudian **Save dashboard**.

## Langkah 4 — Pisahkan Berdasarkan Lini Produksi

Ubah query untuk mengelompokkan berdasarkan tag yang kita set di Node-RED:

```flux
from(bucket: "sensors")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "temperature")
  |> filter(fn: (r) => r._field == "value")
  |> group(columns: ["line"])
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

Publish dari lini kedua dan perhatikan seri baru muncul sendiri:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line2/press02/telemetry/temperature' -m '39.4'
```

Inilah mengapa tag penting. `line` adalah tag, jadi diindeks dan murah untuk
dikelompokkan. Jika itu adalah field, query ini akan jauh lebih lambat dalam skala.

## Langkah 5 — Tambah Alert Threshold

Di panel: **Edit → Alert → New alert rule**.

| Pengaturan | Nilai |
|---|---|
| Condition | `WHEN last() OF query IS ABOVE 41` |
| Evaluate every | `1m` for `1m` |

Simpan, kemudian dorong nilai melewati batasnya:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/temperature' -m '55'
```

Dalam interval evaluasi, rule tersebut aktif. Mengirimkan alert ke email, Slack,
atau Telegram adalah konfigurasi contact point — di luar kursus ini, tapi di
sinilah mulainya.

## Langkah 6 — Buktikan Dashboard Bertahan

```powershell
docker compose down
docker compose up -d
```

Login kembali. Dashboard dan datasource-mu masih utuh, karena
`grafana_data:/var/lib/grafana` menyimpannya.

Kemudian pertimbangkan apa yang akan dilakukan `docker compose down -v` pada
dashboard yang sama, dan kamu akan mengerti mengapa flag itu layak mendapat
respek.

## Di Mana Setiap Hal Tersimpan

| Item | Tersimpan di |
|---|---|
| Pengukuran | `influxdb_data` |
| Dashboard, pengguna, datasource | `grafana_data` |
| Flow dan palette node | `nodered_data` |
| Retained MQTT message | `mosquitto_data` |

Empat container, empat lifecycle independen. Salah satunya bisa dibangun ulang
tanpa menyentuh yang lain — argumen inti untuk mengontainerisasi stack seperti ini.

## Checkpoint

1. Grafana berjalan di browsermu di `localhost:3000`, namun URL datasource adalah
   `http://influxdb:8086`. Jelaskan perbedaannya kepada seseorang yang belum
   membaca Pelajaran 04.
2. Apa yang dilakukan variabel `v.timeRangeStart` dan `v.windowPeriod`?
3. Mengapa menjadikan `line` sebagai tag alih-alih field adalah keputusan yang baik?

---

**Sebelumnya:** [08 — Lab: Modbus to MQTT](08-lab-modbus.md) · **Selanjutnya:** [10 — Troubleshooting](10-troubleshooting.md)
