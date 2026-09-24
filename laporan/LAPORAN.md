# Laporan Proyek Action Learning — A04: Pengolahan Berkas (Metadata Hasil Pengolahan Tiap Job)

**Pola integrasi:** W (Work Queue) — satu worker.
**Panduan Pengerjaan Minggu Kedua (21–25 September 2026).**
Diisi mengikuti struktur template resmi pelatihan (`lab/action-learning/TEMPLATE.md`
pada repo `simpel-lab`). Kode & konfigurasi: [`../README.md`](../README.md).
Identitas & pembagian kontribusi tim: [README §0](../README.md#0-identitas--kontribusi-tim).
Bukti mentah lengkap (lampiran terpisah, tidak diulang di laporan ini):
[`../bukti/`](../bukti/). Diagram satu halaman: [`../diagram/topologi.svg`](../diagram/topologi.svg).

---

## 1. Identifikasi Masalah, Ruang Lingkup, & Kriteria Keberhasilan

**Latar belakang masalah.** Pada alur perizinan SIMPEL, pemohon mengunggah
berkas pendukung yang perlu **diproses** (mis. dianalisis isinya) sebelum
bisa dipakai proses berikutnya (mis. verifikasi kelengkapan). Bila pemrosesan
dilakukan sinkron di dalam request pengunggahan, lonjakan jumlah job atau
gangguan pada layanan pemroses akan langsung menahan pengguna dan berisiko
kehilangan permintaan saat proses pemroses down. Dibutuhkan decoupling:
permintaan pemrosesan berkas (satu **job**) dipublikasikan sebagai event,
diproses oleh worker terpisah, dan hasilnya (metadata hasil pemrosesan)
tersimpan permanen terlepas dari kapan worker sempat memprosesnya.

**Pengguna & peran sistem.**
- **Pemicu event:** proses permintaan pemrosesan berkas (disimulasikan oleh CLI producer `src/produsen.js`, mewakili API/CLI yang menerima permintaan job).
- **Message broker:** RabbitMQ (exchange `files`, routing key `file.process`, queue `filejobs`, lihat README §3).
- **Consumer:** satu worker (`src/pekerja.js`) yang membaca berkas fixture lokal (lihat README §2), menjalankan `operation: word_count`, dan menulis metadata hasil ke PostgreSQL.

**Batasan prototipe.** Berkas yang diproses adalah **tiga berkas teks
sintetis tetap** di `fixtures/` (bukan hasil unggahan sungguhan); `file_id`
pada payload hanya kunci lookup ke berkas-berkas ini, tidak pernah path/URL
bebas. Hanya `operation: "word_count"` yang didukung. Detail lengkap batasan
ada di README §8.

**Kriteria keberhasilan yang terukur** (dipetakan langsung ke skenario uji bersama U1–U4, run `run01`):
1. 20 event valid (`N01`–`N20`) menghasilkan **20 baris metadata unik** di `file_results`, himpunan `event_id` input = output.
2. Saat worker dihentikan, 5 event baru (`G01`–`G05`) **tertahan di queue** (`messages_ready = 5`, `consumers = 0`); setelah worker dipulihkan, kelima event selesai **tanpa pengiriman ulang manual**.
3. Mengirim ulang `N01`–`N05` dengan `event_id` dan payload semula **tidak menambah baris baru** — jumlah total efek bisnis tetap sama.
4. Payload tidak valid (`X01`, `file_id` tidak terdaftar) **tidak menghasilkan efek bisnis** dan masuk jalur penolakan yang terdokumentasi (tabel `file_rejections` + queue `files.penolakan.q`); event valid berikutnya (`V01`) **tetap selesai**, tidak tertahan oleh `X01`.

---

## 2. Arsitektur Sistem & Spesifikasi Kontrak Event

Diagram topologi, tabel exchange/queue/binding, dan penjelasan lengkap ada di
[README §3](../README.md#3-topologi-rabbitmq) — tidak diulang di sini agar
satu sumber kebenaran.

**Pola integrasi: Work Queue (W).** Dipilih karena kasus ini adalah "satu
jenis pekerjaan (pemrosesan berkas) yang harus dikerjakan tepat oleh satu
pemroses per event", bukan "banyak service independen butuh salinan event
yang sama" (itu pola Pub-Sub). Satu queue kerja dengan competing consumers
memungkinkan penambahan worker kedua di kemudian hari tanpa mengubah topologi
(lihat README §8).

**Janji layanan (acceptance vs completion).** Producer CLI ini bersifat
"fire-and-forget setelah publisher confirm": `src/produsen.js` mengembalikan
kendali ke pemanggil segera setelah broker mengonfirmasi penerimaan pesan
(padanan HTTP 202 pada producer berbentuk API sungguhan), **bukan** setelah
efek bisnis tersimpan. Proses bisnis dinyatakan selesai tuntas ketika baris
muncul di `file_results` — dibuktikan lewat `npm run uji -- verifikasi`.
Celah antara "diterima" dan "ter-publish" (tanpa outbox) didokumentasikan di
README §8.

**Kontrak event** — lihat README §2 untuk skema lengkap dan aturan validasi
(`src/kontrak.js`). Ringkasnya: `event_id`, `event_type`
(`file.processing.requested`), `occurred_at` (ISO 8601), `payload`
(`job_id`, `file_id`, `operation`). `file_id` divalidasi terhadap peta fixture
terdaftar ([`src/berkas.js`](../src/berkas.js)) — bukan sekadar format string.

**Strategi idempotensi.** `event_id` adalah primary key tabel efek bisnis;
pemeriksaan-dan-tulis dilakukan atomik lewat satu statement
`INSERT ... ON CONFLICT (event_id) DO NOTHING RETURNING event_id`. Consumer
ack setelah statement ini selesai, sehingga redelivery (baik dari restart
consumer maupun replay manual) tidak pernah menghasilkan baris efek bisnis
kedua. `job_id` (label bisnis job) sengaja **bukan** kunci dedup — lihat
README §9 untuk pembuktian `job_id` berbeda-beda sementara `file_id` dipakai
berulang. Detail di README §4.

---

## 3. Petunjuk Menjalankan dan Menghentikan Sistem

Lihat [README §5](../README.md#5-menjalankan-sistem) untuk instruksi CLI
lengkap (prasyarat, `.env`, `docker compose up`, `npm run db:siapkan`,
menjalankan producer & consumer sebagai proses terpisah, dan cara berhenti).
Proyek ini berdiri sendiri — infrastrukturnya sendiri (RabbitMQ port
5682/15682, PostgreSQL port 5442, kredensial `a04`/`a04pengolahan`), tidak
memakai atau mengganggu container lab lain. Tidak ada kredensial produksi di
repositori.

---

## 4. Konfigurasi Routing dan Kontrol Akses

Deklarasi exchange/queue/binding ada di [`src/topologi.js`](../src/topologi.js)
dan direkap di [README §3](../README.md#3-topologi-rabbitmq). Proyek ini
memakai satu user RabbitMQ (`a04`, tag administrator, dibuat otomatis oleh
`docker-compose.yml`) tanpa user non-admin tambahan — cakupan kontrol akses
per-user (permission regex) berada di luar cakupan capstone ini, sesuai
arahan penugasan yang memfokuskan pola W pada pembagian pekerjaan dan
pemulihan consumer.

---

## 5. Bukti Hasil Pengujian (Evidence Matrix)

Run: **`run01`**. Dijalankan 23 September 2026 pada infrastruktur mandiri
proyek ini, worker tunggal (`A04_WORKER_ID` otomatis dari PID). Cara
menghitung hasil (kenapa 20→25→25→26, dan bedanya dengan pola P) dijelaskan
di [README §9](../README.md#9-cara-menghitung-hasil). Kebijakan bukti
(batas tunggu 60 detik, apa yang direkam) ada di
[README §10](../README.md#10-bukti-yang-dicatat). File bukti mentah (JSON,
keluaran `uji/skenario.js verifikasi`/`ledger`, log worker) ada di
[`../bukti/`](../bukti/).

| Uji | Langkah | Hasil yang Diharapkan | Hasil Aktual | Waktu tunggu (batas 60s) | Bukti | Status |
|---|---|---|---|---|---|---|
| **U1** | Kirim 20 event valid `run01-N01`..`run01-N20` (`file_id` dirotasi `sample-a`/`sample-b`/`sample-c`) | 20 hasil unik, himpunan ID input = output | 20 baris baru di `file_results`; `idHasilBisnis` cocok persis 20 ID yang dikirim; `ukuran_bytes`/`jumlah_kata` sesuai fixture (61/10, 212/29, 43/5) | 29 ms | `bukti/run01-u1.json` | **LULUS** |
| **U2** | Setelah U1, worker dihentikan dengan `Ctrl+C` di terminal worker; kirim 5 event baru `G01`–`G05` | 5 pesan menunggu di queue; setelah worker pulih, ke-5 ID selesai tanpa kirim ulang manual | Saat worker mati, Management UI RabbitMQ menunjukkan `ready=5, unacked=0, consumers=0` (diamati langsung, terekam di video uji skenario). Setelah worker dinyalakan kembali: kelima `G01`–`G05` otomatis diproses (log `"hasil":"baru"`), total hasil bisnis run naik dari 20 → **25** | 36225 ms (±36 s) | `bukti/run01-u2.json` | **LULUS** |
| **U3** | Kirim ulang `run01-N01`..`run01-N05` dengan `event_id` + payload **persis semula** (`occurred_at` sama dengan pengiriman U1) | Efek bisnis tidak bertambah; jumlah hasil tetap 25 | Log worker: kelima event bertanda `"hasil":"duplikat-diabaikan"`. Jumlah baris tetap **25** (sebelum 25, sesudah 25) | 29 ms | `bukti/run01-u3.json` | **LULUS** |
| **U4** | Kirim `run01-X01` (payload `file_id: "sample-tidak-terdaftar"`, tidak valid), lalu `run01-V01` (valid) | `X01` masuk jalur penolakan terdokumentasi tanpa efek bisnis; `V01` selesai, tidak tertahan | `X01` tercatat di `file_rejections` (alasan: *"payload.file_id ... tidak terdaftar di fixture lokal"*) dan `files.penolakan.q` (`ready=1` via dead-letter); **tidak ada** baris `run01-X01` di `file_results`. `V01` selesai (`"hasil":"baru"`) segera setelah `X01` ditolak — total hasil bisnis run naik 25 → **26** | 27 ms | `bukti/run01-u4.json` | **LULUS** |

Kolom "Waktu tunggu" adalah `waktuTungguMs` aktual dari polling
`verifikasi` (lihat README §10). Keempat uji tercapai di bawah batas 60 detik,
sehingga tidak ada kasus `tercapaiDalamWaktu: false` yang perlu dicatat. Waktu
U2 jauh lebih lama karena `verifikasi` dijalankan saat worker masih mati,
sehingga angkanya ikut mencakup jeda sampai worker dinyalakan kembali.

**Ringkasan akhir run01:** 26 baris efek bisnis (20 + 5 + 0 dari replay + 1
dari `V01`), 1 baris penolakan (`X01`), 0 pesan tak ter-route sepanjang
pengujian (`files.tanpa_rute.q` selalu 0) — membuktikan seluruh publish
selama U1–U4 ter-route ke queue tujuan yang benar. Ekspor mentah seluruh 26
baris ledger + 1 baris penolakan (bukan ringkasan), lengkap dengan `job_id`,
`file_id`, `ukuran_bytes`, dan `jumlah_kata` per baris, ada di
`bukti/run01-ledger.json` — sehingga hitungan di atas bisa dibuktikan ulang
oleh pengajar, bukan sekadar diklaim. Verifikasi silang: `job_id` pada
seluruh 26 baris **berbeda satu sama lain** (`JOB-N01`..`JOB-V01`), sementara
`file_id` `sample-a`/`sample-b`/`sample-c` masing-masing dipakai berulang
oleh banyak `job_id` — membuktikan kriteria "berkas yang sama boleh dipakai
beberapa job, dedup berbasis `event_id`" (README §9).

---

## 6. Laporan Investigasi Troubleshooting

### Analisis Perilaku U2: Pesan Tertahan Saat Worker Mati, Terproses Saat Worker Pulih

**Gejala yang diamati.** Setelah worker dihentikan dengan `Ctrl+C`,
producer tetap berhasil mengirim 5 event baru `G01`–`G05`: tidak ada error di
sisi producer, dan broker mengembalikan publisher confirm untuk setiap pesan.
Management UI RabbitMQ menunjukkan queue `filejobs` dengan `ready=5,
unacked=0, consumers=0` (terekam di video uji skenario), dan belum ada
satu pun baris `G01`–`G05` di `file_results`. Begitu worker dinyalakan
kembali, kelima pesan langsung diproses tanpa ada pengiriman ulang dari
producer, dan total hasil bisnis run naik dari 20 → 25.

**Pertanyaan investigasi.** (1) Mengapa producer tidak gagal walaupun tidak
ada consumer yang hidup? (2) Di mana kelima pesan disimpan selama worker
mati? (3) Bagaimana worker yang baru dinyalakan bisa mengambilnya tanpa
campur tangan manual?

**Penjelasan per tahap.**
1. **Producer tidak bergantung pada consumer.** Producer hanya mem-publish
   ke exchange `files` dengan routing key `file.process`. Binding
   `files` → `filejobs` dideklarasikan di broker (`src/topologi.js`), bukan
   dimiliki oleh proses worker, sehingga routing tetap berjalan walaupun
   worker mati. Publisher confirm dikirim broker begitu pesan diterima dan
   ter-route ke queue, tidak menunggu pesan dikonsumsi. Opsi
   `mandatory: true` hanya membuat pesan dikembalikan bila pesan **tidak
   ter-route** ke queue mana pun, bukan bila queue-nya tidak punya consumer. Karena itu
   `files.tanpa_rute.q` tetap 0 sepanjang U2.
2. **Pesan menunggu dengan status *Ready*.** Tanpa consumer terdaftar
   (`consumers=0`), broker tidak punya tujuan pengiriman, jadi pesan tetap
   di queue sebagai *ready*. `unacked=0` karena tidak ada consumer yang
   sedang memegang pesan. Queue `filejobs` bersifat `durable: true` dan
   pesan dikirim dengan `persistent: true` (`src/broker.js`), sehingga
   pesan juga dirancang bertahan saat broker di-restart. Skenario restart
   broker ini sendiri **tidak** diuji di U2.
3. **Worker pulih langsung mengambil antrean.** Saat start, `src/pekerja.js`
   mendaftar sebagai consumer (`basic.consume`) pada `filejobs` dengan
   `prefetch(1)`. Broker kemudian mengirim pesan satu per satu: sebuah pesan
   berpindah dari *ready* ke *unacked* selama diproses, lalu hilang dari queue
   setelah worker `ack`. Worker hanya `ack` sesudah
   `INSERT ... ON CONFLICT` di PostgreSQL berhasil. Kelima pesan ini belum
   pernah dikirim ke consumer mana pun, sehingga tercatat sebagai pengiriman
   pertama (`redelivered: false`), bukan redelivery.
4. **Mengapa tidak ada pesan yang hilang saat worker dihentikan dengan
   `Ctrl+C`.** `Ctrl+C` mengirim `SIGINT` ke worker, dan `src/pekerja.js`
   menanganinya sebagai *graceful shutdown*: consumer di-*cancel* lebih dulu
   supaya tidak menerima pesan baru, worker menunggu pekerjaan aktif selesai
   dan di-`ack`, lalu koneksi ditutup. Pada U2 worker dihentikan dalam
   keadaan idle, karena seluruh pesan U1 sudah di-`ack` dan `unacked=0`.
   Seandainya proses worker mati mendadak di tengah memproses pesan
   (skenario crash, tidak diuji di U2), pesan itu belum di-`ack`. Broker akan
   mengembalikannya ke *ready* saat koneksi AMQP putus dan mengirimkannya
   ulang ke worker berikutnya (`redelivered: true`). Kalaupun baris hasilnya
   sudah terlanjur tersimpan sebelum crash, `ON CONFLICT (event_id) DO
   NOTHING` mencegah baris ganda, seperti yang dibuktikan U3.

**Bukti.** `bukti/run01-u2.json` mencatat `jumlahHasilBisnis: 25` dengan
`idHasilBisnis` memuat `run01-G01`..`run01-G05`, serta `waktuTungguMs:
36225`. Artinya perintah `verifikasi` dijalankan saat worker masih mati dan
menunggu sekitar 36 detik sampai worker dinyalakan kembali dan kelima pesan
selesai, masih di bawah batas 60 detik.

**Catatan pembacaan snapshot queue.** Blok `antrean.kerja` pada file bukti
yang sama masih menunjukkan `ready: 5, consumers: 0`, padahal pada saat itu
25 hasil sudah tercatat di database. Dugaan penyebabnya: Management API
RabbitMQ tidak real-time. Statistik queue diperbarui secara berkala
(bawaan `collect_statistics_interval` 5 detik), sehingga snapshot yang
diambil tepat setelah hasil tercapai masih menampilkan keadaan sebelum
worker pulih. Dugaan ini dapat dikonfirmasi dengan menjalankan
`npm run uji -- antrean` beberapa detik kemudian: angkanya seharusnya sudah
`ready: 0, consumers: 1`. Pelajarannya, sumber kebenaran untuk menyatakan
pekerjaan **selesai** adalah tabel `file_results`. Angka queue dari
Management API hanya indikator pendukung.

**Kesimpulan.** Perilaku ini sesuai desain, bukan bug. Pola Work Queue
memisahkan waktu kerja producer dan consumer: producer cukup memastikan
pesan diterima broker, broker menyimpan pesan secara durable selama
consumer tidak tersedia, dan consumer menyelesaikannya saat pulih dengan
ack manual dan idempotensi berbasis `event_id`. Dengan begitu tidak ada
pesan yang hilang maupun terproses dua kali.

---

## 7. Batasan Sistem dan Pengembangan Berikutnya

Lihat [README §8](../README.md#8-batasan-prototipe) untuk daftar lengkap.
Ringkasnya: satu worker (worker kedua = pengembangan opsional belum diuji),
fixture berkas sintetis tetap dengan satu operasi (`word_count`) yang
didukung, tidak ada transactional outbox pada producer (celah
acceptance-vs-publish), broker & database single-node tanpa TLS/mTLS, dan
**tidak ada klaim "zero message loss" atau "exactly-once" menyeluruh** —
hasil U1–U4 hanya membuktikan idempotensi consumer-side dan pemulihan dari
gangguan proses consumer, bukan dari kegagalan infrastruktur (broker/DB)
itu sendiri.

Pengembangan berikutnya yang disarankan: worker kedua untuk kasus W
(memverifikasi tidak ada duplikasi efek bisnis lintas-consumer berkat
idempotensi berbasis `event_id`), operasi tambahan selain `word_count` (mis.
`checksum`, `line_count`), retry/backoff otomatis dengan TTL queue untuk
kegagalan sementara, dan transactional outbox pada sisi producer bila kasus
ini dikembangkan menjadi API HTTP sungguhan.

---

## 8. Kontribusi dan Rujukan

Rincian lengkap ada di [README §7](../README.md#7-bagian-yang-dipakai-ulang-dari-lab-simpel).
Ringkasnya: `openPublisher()` divendorkan dan diadaptasi dari
`layanan/messaging.js` (repo `simpel-lab`) menjadi [`src/broker.js`](../src/broker.js);
pola alternate-exchange, dead-letter-exchange, graceful shutdown,
`INSERT ... ON CONFLICT`, dan Management API client diadaptasi dari
`layanan/messaging.js`, `tools/kasus.js`, `tools/operasi.js`, dan
`layanan/alur.js` pada repo yang sama. Kontrak event, skema tabel `file_*`,
fixture berkas lokal + pemrosesan `word_count` sungguhan, seluruh CLI
producer/consumer/orkestrasi uji, serta infrastruktur `docker-compose.yml`
mandiri adalah pengembangan baru untuk kasus A04.

---

## Status Checkpoint

| Hari / Tanggal | Target Milestone | Status |
|---|---|---|
| Senin, 21 September | Rumusan masalah, diagram topologi, kontrak event | Selesai (bagian 1–2 di atas) |
| Selasa, 22 September | Implementasi dasar producer, exchange, queue, consumer aktif | Selesai — proyek dipisah menjadi folder mandiri |
| **Rabu, 23 September** | **Routing lengkap + bukti pengujian** | **Selesai — kontrak & topologi disesuaikan ke spesifikasi job/file_id/operation, U1–U4 dijalankan ulang penuh dan lulus (bagian 5), beserta analisis perilaku U2 (bagian 6)** |
| Kamis, 24 September | Pengujian failure/recovery + laporan | Draft laporan ini disusun; perlu direview ulang & dilengkapi refleksi tim sebelum dikumpulkan |
| Jumat, 25 September | Presentasi & sesi feedback | Menunggu jadwal sesi sinkron; lihat `../presentasi/` |
