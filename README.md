# IoT Monitoring & Kontrol: REST API Node-RED

Sistem monitoring dan kontrol IoT berbasis MQTT. Simulator ESP32 mengirim data suhu dan kelembaban, Node-RED bertindak sebagai backend REST API, dan dashboard web kustom menampilkan data secara real-time.

## Gambaran Sistem

Simulator ESP32 (Python) mengirim data sensor lewat MQTT ke Node-RED. Node-RED menyimpannya ke database SQLite, menyediakannya lewat REST API dan WebSocket, lalu dashboard web menampilkannya.

![Flow lengkap Node-RED](screenshots/flow-lengkap.png)
*Seluruh alur Node-RED: subscribe MQTT, endpoint REST, pengecekan API key, WebSocket, threshold, alert, dan SQLite.*

---

## Praktikum Dasar (Bagian A-E)

![Simulator ESP32](screenshots/dasar-simulator.png)
*Simulator ESP32 (Python) mengirim data suhu dan kelembaban ke topic `iot/sensor` setiap 3 detik.*

![Flow dasar](screenshots/dasar-flow.png)
*Flow dasar Node-RED. Node `Subscribe Sensor` berstatus connected dan panel Debug menampilkan data sensor yang masuk dari MQTT.*

![Tes GET /api/suhu](screenshots/dasar-curl-suhu.png)
*Tes endpoint `GET /api/suhu` lewat curl. Server mengembalikan JSON berisi suhu, kelembaban, dan waktu.*

![Dashboard dasar](screenshots/dasar-dashboard.png)
*Dashboard dasar: kartu suhu, kelembaban, status sensor ONLINE, tombol kontrol lampu, dan tabel history.*

![Kontrol lampu](screenshots/dasar-kontrol-lampu.png)
*Tombol di dashboard mengirim `POST /api/kontrol`. Perintah dipublish ke MQTT dan diterima simulator (`Perintah diterima: lamp on/off`).*

---

## Tugas Pengembangan

### 1. Endpoint `/api/threshold` (PUT) dan Notifikasi

**Pengertian:** Threshold (ambang batas) adalah nilai suhu maksimum yang dianggap aman. Endpoint ini mengatur batas tersebut lewat request `PUT`. Setiap data sensor masuk, sistem membandingkan suhu dengan batas. Jika melewatinya, sistem mengirim notifikasi ke topic MQTT `iot/alert` dan data ditandai `alert: true`.

![Dashboard alert](screenshots/dashboard-alert.png)
*Batas suhu diatur lewat tombol "Set Batas Suhu". Saat suhu 30.8°C melewati batas 27°C, dashboard menampilkan banner peringatan. Terminal simulator di sebelah kanan menerima `ALERT` dari topic `iot/alert`.*

![Flow threshold](screenshots/flow-threshold.png)
*Alur `PUT /api/threshold` → `Cek API Key` → `Set Threshold`, serta node `Publish Alert` yang mengirim notifikasi ke MQTT.*

### 2. Grafik Real-Time dengan Chart.js

**Pengertian:** Chart.js adalah library JavaScript untuk membuat grafik di web. Grafik pada dashboard menampilkan perubahan suhu dan kelembaban terhadap waktu, dan bertambah otomatis setiap ada data baru, sehingga tren data lebih mudah dilihat dibanding angka saja.

![Grafik Chart.js](screenshots/grafik.png)
*Grafik real-time: garis biru untuk kelembaban dan garis merah untuk suhu, dengan sumbu waktu yang terus bertambah.*

### 3. Autentikasi API Key (`X-API-Key`)

**Pengertian:** API key adalah kunci rahasia sederhana yang harus dikirim klien lewat header `X-API-Key` pada setiap request. Node-RED memeriksa kunci tersebut. Jika benar, request diproses. Jika salah atau tidak ada, server menolak dengan status `401 Unauthorized`. Ini mencegah pihak tanpa izin mengakses data dan mengontrol perangkat.

![Kode API key di dashboard](screenshots/apikey-kode.png)
*Di dashboard, fungsi `apiFetch` otomatis menambahkan header `X-API-Key` pada setiap request ke API.*

![Node Cek API Key](screenshots/flow-apikey.png)
*Node `Cek API Key` dipasang di depan setiap endpoint. Output 1 melanjutkan request yang valid, output 2 mengirim respons penolakan.*

### 4. WebSocket untuk Update Tanpa Polling

**Pengertian:** Pada polling, dashboard bertanya ke server berulang kali (misalnya tiap 3 detik). WebSocket membuat koneksi dua arah yang tetap terbuka, sehingga server langsung mendorong data ke dashboard begitu data sensor tiba. Hasilnya lebih cepat, lebih ringan, dan tanpa request berulang. Endpoint WebSocket: `/ws/sensor`.

![Flow WebSocket](screenshots/flow-websocket.png)
*Node `[ws] /ws/sensor` di Node-RED berstatus connected, artinya dashboard sudah terhubung.*

![WebSocket di browser](screenshots/ws-connected.png)
*Tab Network browser (filter Socket) menunjukkan koneksi `sensor` dengan status 101 (Switching Protocols) bertipe websocket.*

![Tanpa polling suhu](screenshots/ws-tanpa-polling.png)
*Dengan filter `suhu`, hanya ada 1 request saat halaman dibuka. Setelah itu data suhu datang lewat WebSocket, bukan polling berulang.*

### 5. Penyimpanan History ke Database SQLite

**Pengertian:** Sebelumnya history disimpan di flow context (memori), sehingga hilang saat Node-RED di-restart. Dengan SQLite, setiap data sensor ditulis ke file database `iot.db` (tabel `sensor`: `id`, `suhu`, `kelembaban`, `waktu`). Endpoint `/api/history` membaca 50 data terakhir dari database, sehingga data tetap ada walau sistem dimatikan lalu dinyalakan lagi.

![Flow SQLite](screenshots/flow-sqlite.png)
*Node `timestamp` → `Create Table` membuat tabel sekali saja, dan node `Insert Sensor` menyimpan setiap data sensor ke `C:/node-red-data/iot.db`.*

![Request history](screenshots/history-request.png)
*Dashboard membaca `/api/history` dan mendapat respons 200 berisi data dari database.*

Pengujian: setelah Node-RED di-restart, endpoint `/api/history` tetap mengembalikan data sebelum restart karena tersimpan di file `iot.db`.

---

## Daftar Endpoint

| Endpoint | Method | Fungsi |
|---|---|---|
| `/api/suhu` | GET | Data sensor terakhir |
| `/api/history` | GET | 50 data sensor terakhir (dari SQLite) |
| `/api/kontrol` | POST | Kirim perintah ke perangkat |
| `/api/threshold` | PUT | Set batas suhu untuk alert |
| `/ws/sensor` | WebSocket | Stream data sensor real-time |

Semua endpoint `/api/*` memerlukan header `X-API-Key`.

## Isi Repository

- `flows.json`: flow Node-RED
- `dashboard.html`: dashboard web
- `simulasi_esp32.py`: simulator ESP32
- `screenshots/`: bukti hasil pengerjaan
