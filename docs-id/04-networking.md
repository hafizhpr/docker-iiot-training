# 04 — Networking

Ini adalah pelajaran yang paling banyak menghemat waktumu di kemudian hari.
Hampir setiap masalah "berfungsi di mesinku tapi tidak di Docker" adalah
kesalahpahaman tentang networking.

## Dua Angka, Dua Dunia yang Berbeda

```yaml
ports:
  - "8086:8086"
     │     │
     │     └── container port — port DI DALAM container
     └──────── host port — port DI MESINmu
```

Keduanya biasanya ditulis sama di sini karena kita mempertahankan default, tapi
sebenarnya tidak ada hubungannya. Ini juga valid:

```yaml
ports:
  - "9999:8086"    # akses InfluxDB di localhost:9999 dari browsermu
```

Container tetap meyakini bahwa ia serving di port 8086. Hanya mapping host yang
berpindah.

## Network Privat

Compose membuat user-defined bridge network untuk proyek. Dalam
`docker-compose.yml` kita:

```yaml
networks:
  iiot-net:
    driver: bridge
```

Setiap service bergabung ke dalamnya, dan network menyediakan **DNS server**
tertanam. Itulah yang membuat ini berfungsi:

```
mqtt://mosquitto:1883      bukan   mqtt://172.21.0.3:1883
http://influxdb:8086       bukan   http://172.21.0.2:8086
```

Alamat IP container bisa berubah setiap restart. Nama service tidak pernah berubah.
Selalu gunakan namanya.

## Lihat Sendiri

Jalankan stack, kemudian cari container lain dari dalam Node-RED:

```powershell
docker compose up -d
docker compose exec nodered getent hosts mosquitto influxdb grafana
```

Output, dengan alamat yang ditetapkan mesinmu:

```
172.21.0.3   mosquitto
172.21.0.2   influxdb
172.21.0.4   grafana
```

Tiga nama, tiga alamat, di-resolve dari dalam container. Tanpa hosts file, tanpa
konfigurasi.

Sekarang buktikan bahwa alamat-alamat itu tidak stabil:

```powershell
docker compose down
docker compose up -d
docker compose exec nodered getent hosts influxdb
```

Biasanya alamat yang berbeda, selalu nama yang sama. Itulah seluruh argumen untuk
menggunakan nama service dalam satu eksperimen.

Jika alamat kebetulan kembali identik, jalankan sekali atau dua kali lagi. Docker
membagikan alamat sesuai urutan container yang benar-benar start, dan urutan itu
adalah race — tidak ada yang memesan `172.21.0.2` untuk InfluxDB. Mendapat nomor
yang sama dua kali adalah keberuntungan, bukan jaminan, dan menulis nomor itu ke
dalam konfigurasi adalah persis bug yang ingin dicegah oleh eksperimen ini.

## Jebakan `localhost`

**Ini adalah kesalahan paling umum dalam kursus ini.** Baca dua kali.

Di dalam container, `localhost` berarti *container itu sendiri* — bukan mesinmu,
dan bukan container lain. Setiap container memiliki loopback interface-nya sendiri.

Jadi ketika kamu mengkonfigurasi node InfluxDB di dalam Node-RED:

| Yang kamu ketik | Hasilnya |
|---|---|
| `http://localhost:8086` | ❌ Node-RED melihat ke dalam dirinya sendiri. Tidak ada yang mendengarkan. Connection refused. |
| `http://influxdb:8086` | ✅ Docker DNS me-resolve `influxdb` ke container yang tepat. |

Aturan yang sama berlaku di Grafana saat menambahkan datasource: URL-nya adalah
`http://influxdb:8086`, meskipun kamu mengakses UI Grafana di `localhost:3000`
di browsermu.

### Aturannya

| Siapa yang terhubung | Alamat yang digunakan |
|---|---|
| Browsermu → sebuah container | `localhost:<host port>` |
| Sebuah container → container lain | `<service name>:<container port>` |

Browsermu ada di host, di luar network Docker, jadi ia menggunakan host port yang
dipublikasikan. Container ada di dalam, jadi mereka menggunakan nama service.

Buktikan bahwa jebakannya nyata:

```powershell
docker compose exec nodered wget -qO- http://influxdb:8086/health
docker compose exec nodered wget -qO- http://localhost:8086/health
```

Yang pertama mengembalikan JSON. Yang kedua gagal terhubung.

## Menjangkau Mesin Host

Ada kasus ketiga yang tidak tercakup dalam tabel di atas: container yang perlu
menjangkau service yang berjalan di **mesinmu**, di luar Docker sepenuhnya.
Server Modbus TCP, historian lokal, database yang belum kamu containerisasi.

`localhost` gagal di sini karena alasan yang sama seperti sebelumnya, dan nama
service tidak ada karena tidak ada container seperti itu. Alamatnya adalah:

```
host.docker.internal
```

Periksa apakah bisa di-resolve:

```powershell
docker compose exec nodered getent hosts host.docker.internal
```

```
192.168.65.254   host.docker.internal
```

Kamu mungkin melihat alamat IPv6 sebagai gantinya, atau keduanya:

```powershell
docker compose exec nodered grep host.docker.internal /etc/hosts
```

```
192.168.65.254        host.docker.internal
fdc4:f303:9324::254   host.docker.internal
```

Keduanya baik-baik saja. Keduanya mengarah ke gateway yang sama kembali ke hostmu,
dan client mencobanya secara bergantian — server yang hanya mendengarkan IPv4 tetap
bisa dijangkau.

### Contoh: Server Modbus TCP di PC-mu

| Field di node Modbus | Nilai |
|---|---|
| Type | `TCP` |
| Host | `host.docker.internal` |
| Port | `502` |

### Aturan Alamat Lengkap

| Siapa yang terhubung | Alamat yang digunakan |
|---|---|
| Browsermu → sebuah container | `localhost:<host port>` |
| Sebuah container → container lain | `<service name>:<container port>` |
| Sebuah container → mesin hostmu | `host.docker.internal:<port>` |
| Sebuah container → dirinya sendiri | `localhost:<container port>` |

### Mengapa `extra_hosts` Ada di File Compose

```yaml
nodered:
  extra_hosts:
    - "host.docker.internal:host-gateway"
```

Docker Desktop di Windows dan macOS menyediakan `host.docker.internal` secara
gratis. Docker Engine di Linux **tidak** — nama itu tidak bisa di-resolve.

Baris itu menambahkannya secara eksplisit, sehingga file compose yang sama berfungsi
di semua platform. Di Docker Desktop tidak mengubah apapun; di Linux itulah
perbedaan antara berfungsi dan tidak.

### Ketika Masih Gagal

Dua penyebab, berdasarkan urutan kemungkinan:

**Service host mendengarkan hanya di `127.0.0.1`.** Container datang dari luar,
sehingga ditolak. Service harus bind ke `0.0.0.0`. Periksa di host:

```powershell
Get-NetTCPConnection -LocalPort 502 -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess
```

`LocalAddress` bernilai `0.0.0.0` bisa dijangkau dari container. `127.0.0.1` tidak bisa.

**Firewall host memblokir port.** Di Windows, tambahkan inbound rule untuknya.
Perlu diketahui: host melihat koneksi yang datang dari `127.0.0.1` bukan dari
IP container, karena Docker Desktop mem-proxy-nya — jadi firewall rule yang ditulis
berdasarkan alamat container tidak akan berperilaku seperti yang diharapkan.

## `depends_on` Bukan Berarti "Tunggu Sampai Siap"

```yaml
depends_on:
  - mosquitto
  - influxdb
```

Ini hanya mengontrol **urutan start**. Compose menjalankan InfluxDB sebelum
Node-RED, tapi tidak menunggu InfluxDB selesai diinisialisasi — hanya menunggu
proses container untuk ada.

Untuk stack ini tidak masalah: Node-RED reconnect dengan sendirinya. Dalam sistem
di mana urutan startup benar-benar penting, kamu memerlukan healthcheck beserta
`depends_on: { condition: service_healthy }`, atau retry logic di aplikasinya.

Mengasumsikan `depends_on` menjamin kesiapan menyebabkan kegagalan intermiten yang
hanya muncul di mesin yang lebih lambat. Ini adalah kesalahan klasik.

## Memeriksa Network

```powershell
docker network ls
docker network inspect iiot-training_iiot-net
```

Bagian `Containers` dari output inspect mencantumkan setiap container yang terhubung
beserta IP-nya saat ini.

Perhatikan nama network: Compose menambahkan prefix nama proyek padanya, yang berasal
dari field `name:` di bagian atas `docker-compose.yml`.

## Checkpoint

1. Browsermu membuka Grafana di `localhost:3000`. Di dalam Grafana kamu menambahkan
   datasource InfluxDB. URL apa yang kamu masukkan, dan mengapa bukan `localhost`?
2. Apa yang akan diubah oleh `"9999:8086"`, dan apa yang tidak berubah?
3. Mengapa `depends_on` saja tidak cukup untuk menjamin database sudah siap?

---

**Sebelumnya:** [03 — First Run](03-first-run.md) · **Selanjutnya:** [05 — Volumes and Persistence](05-volumes.md)
