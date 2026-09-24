# Outline Presentasi — A04-W: Pengolahan Berkas

Target: 3 JP sesi sinkron Jumat 25 September (135 menit, dibagi seluruh
peserta). Alokasikan realistis 8–10 menit presentasi per kelompok + tanya
jawab. Outline ini mengikuti urutan 7 slide pada deck final.

1. **Judul** (slide 1)
   Nama pelatihan, kasus A04-W — Pengolahan Berkas, alur Producer →
   Exchange → Queue → Worker → PostgreSQL, tanggal sesi, anggota kelompok 2.

2. **Masalah & ruang lingkup** (slide 2)
   Kendala pengolahan berkas sinkron pada SIMPEL, batas prototipe (data
   sintetis, simulasi lokal) — dari `laporan/LAPORAN.md` §1.

3. **Arsitektur & topologi** (slide 3)
   Diagram producer → exchange `files` → queue `filejobs` → worker →
   PostgreSQL (README §3), plus `files.tanpa_rute.q` dan `files.penolakan.q`.
   Jelaskan kenapa direct exchange + alternate-exchange, kenapa pola W.

4. **Kontrak event & idempotensi** (slide 4)
   Contoh JSON event_id/event_type (`file.processing.requested`)/occurred_at/
   payload (`job_id`/`file_id`/`operation`). Jelaskan `file_id` hanya kunci
   lookup ke fixture lokal (tidak pernah path/URL bebas), dan bedanya
   `job_id` (label bisnis, boleh berulang pakai `file_id` yang sama) vs
   `event_id` (kunci dedup). Strategi `ON CONFLICT DO NOTHING` +
   ack-setelah-tersimpan (README §4).

5. **Skenario & bukti pengujian U1–U4** (slide 5)
   Tabel evidence matrix dari `laporan/LAPORAN.md` §5 — tunjukkan angka
   nyata: 20 → 25 (setelah recovery) → tetap 25 (replay) → 26 (setelah X01
   ditolak + V01 selesai), plus tautan video uji coba.

6. **Analisis U2: gangguan & pemulihan** (slide 6)
   Ringkas dari `laporan/LAPORAN.md` §6: worker dihentikan dengan `Ctrl+C`,
   producer tetap sukses mengirim `G01`–`G05`, Management UI menunjukkan
   `ready=5, consumers=0` (cuplikan video uji skenario). Setelah worker
   dinyalakan kembali, kelima pesan diproses tanpa kirim ulang (20 → 25).
   Jelaskan kenapa: binding & queue durable ada di broker, pesan persistent,
   ack manual setelah tersimpan di PostgreSQL, idempotensi `event_id`.

7. **Penutup** (slide 7)
   Terima kasih, diskusi & tanya jawab, tautan repo
   https://github.com/dinotyyy/actionLearning_messageBroker.

Batasan & pengembangan lanjut (`laporan/LAPORAN.md` §7) serta reuse &
kontribusi (§8) tidak punya slide sendiri — siapkan sebagai bahan jawaban
saat tanya jawab.

**Catatan:** sudah dibuat sebagai `PJJ_AL_Presentasi_case 04-kelompok2.pptx`
(7 slide, di folder ini).
