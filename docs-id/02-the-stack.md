# 02 — The Stack

## Pipeline-nya

Setiap sistem IIoT memecahkan empat masalah yang sama dalam urutan yang sama:
ambil data dari mesin, ubah bentuknya, simpan, dan tampilkan ke manusia.

```
  Machines  ──▶  Mosquitto  ──▶  Node-RED  ──▶  InfluxDB  ──▶  Grafana
  sensors         broker           flows          TSDB         dashboards
                  :1883            :1880          :8086        :3000

                TRANSPORT        TRANSFORM        STORE        VISUALISE
```

## Empat Service

### Mosquitto — broker (port 1883)

Sensor tidak berbicara langsung ke database. Mereka **publish** ke sebuah topic
pada broker, dan siapapun yang tertarik **subscribe** ke topic tersebut. Publisher
dan subscriber tidak pernah tahu satu sama lain.

```
factory/line1/press01/telemetry/temperature  ──▶  broker  ──▶  Node-RED
                                        ├─▶  an alarm service
                                        └─▶  a mobile app
```

Decoupling itulah poinnya. Menambahkan consumer kelima tidak memerlukan perubahan
apapun di sisi sensor.

MQTT dirancang untuk perangkat terbatas: sebuah paket publish hanya membawa
beberapa byte overhead protokol, dibandingkan sekitar dua ratus byte untuk request
HTTP yang setara. Melalui koneksi seluler dengan 5.000 sensor, perbedaan itu
adalah seluruh tagihan.

### Node-RED — flow engine (port 1880)

Editor berbasis browser di mana kamu menghubungkan node-node alih-alih menulis
kode penghubung. Dalam stack ini, ia subscribe ke topic MQTT, mengubah bentuk
payload, dan menuliskannya ke InfluxDB.

Tugas utamanya adalah **impedance matching**. Sensor memancarkan apapun yang
diputuskan vendornya; database menginginkan skema yang konsisten. Node-RED adalah
tempat bertemunya yang berantakan dengan yang rapi.

### InfluxDB — database time-series (port 8086)

Data sensor memiliki bentuk yang tidak ditangani dengan baik oleh database
general-purpose: append-only, diurutkan berdasarkan timestamp, volume besar, dan
hampir selalu dikueri sebagai "N menit terakhir" atau "rata-rata per jam".

Database time-series dibangun untuk bentuk itu. Ia juga melakukan sesuatu yang
tidak akan dilakukan PostgreSQL untukmu secara out-of-the-box: **secara otomatis
menghapus data yang lebih lama dari X**. Ketika 500 sensor melapor setiap detik,
retention bukan sekadar fitur tambahan.

### Grafana — dashboard (port 3000)

Terhubung ke InfluxDB, menggambar panel time-series, dan mengirim alert pada
threshold. Ia tidak menyimpan data pengukuran sendiri — hanya dashboard, pengguna,
dan definisi datasource.

## Mengapa Port-Port Ini

| Service | Port | Alasannya |
|---|---|---|
| Mosquitto | 1883 | Port MQTT yang terdaftar di IANA |
| Mosquitto | 9001 | Konvensi untuk MQTT over WebSockets |
| Node-RED | 1880 | Default proyek |
| InfluxDB | 8086 | Default proyek |
| Grafana | 3000 | Default proyek |

Kita mempertahankan default dengan sengaja. Setiap tutorial, jawaban forum, dan
dokumen vendor mengasumsikannya, sehingga pelajar yang stuck bisa mencari dan
menemukan jawaban yang sesuai dengan apa yang ada di layarnya.

**Port 3000 adalah yang paling sering bentrok** — React, Next.js, dan Rails
semuanya menggunakannya. Pelajaran 10 menunjukkan cara memindahkannya.

## Mengapa Container Cocok untuk Stack Ini

Menginstal keempatnya secara native berarti empat package manager, empat service
manager, empat lokasi file konfigurasi, dan empat prosedur upgrade yang berbeda.

Sebagai container, itu adalah satu file, satu perintah, dan satu uninstall yang
tidak meninggalkan apapun. Untuk environment pelatihan yang dibangun dan
dihancurkan berulang kali, itu adalah keputusan yang tepat.

## Checkpoint

1. Mengapa sensor publish ke broker alih-alih menulis langsung ke database?
2. Apa yang diberikan database time-series yang tidak dimiliki database relasional?
3. Service mana dari keempat service yang tidak menyimpan data pengukuran sama sekali?

---

**Sebelumnya:** [01 — Docker Fundamentals](01-docker-fundamentals.md) · **Selanjutnya:** [03 — First Run](03-first-run.md)
