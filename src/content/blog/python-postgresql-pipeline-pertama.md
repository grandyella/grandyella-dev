---
title: 'Pipeline pertama jalan — dari Telegram ke database'
description: 'Pertama kalinya data tersimpan beneran, bukan cuma di memory.'
pubDate: 'Apr 18 2026 12:00:00'
---

Ada satu momen yang bikin beda antara
"belajar coding" sama "bikin sesuatu beneran."

Momen itu waktu data yang lo masukin
tersimpan di database dan bisa dipanggil lagi.

Hari ini gue ngerasain itu untuk pertama kalinya.

---

Mulai dari desain tabel dulu.

Sebelum nulis kode Python, gue harus tau
mau nyimpen data apa dan formatnya gimana.

Tabel transaksi. Kolom-kolomnya:
id, user_id, jumlah, tipe, kategori, keterangan,
tanggal, created_at.

Yang menarik waktu bikin ini: PostgreSQL bisa
validasi data di level database. Kolom tipe
cuma boleh diisi "masuk" atau "keluar" —
kalau diisi yang lain, langsung ditolak.

Bukan validasi di Python. Bukan validasi di bot.
Tapi di database itu sendiri.

Namanya CHECK constraint. Gue gak tau ini ada
sebelumnya. Sekarang tau.

---

Konek Python ke PostgreSQL pakai psycopg2.

Library-nya straightforward. Yang gue pelajari
lebih banyak bukan soal cara pakainya,
tapi soal kenapa hal-hal tertentu ditulis
dengan cara tertentu.

Misalnya waktu nulis query INSERT:

    cursor.execute(sql, (user_id, jumlah, tipe, kategori, keterangan))

Kenapa parameternya dipisah begitu?
Kenapa gak langsung masukin ke string SQL-nya?

Jawabannya: SQL Injection.

Kalau lo masukin nilai langsung ke string SQL,
user jahat bisa kirim input yang mengandung
perintah SQL — dan database akan jalanin itu.

Dengan cara di atas, psycopg2 yang handle.
Nilai user diperlakukan sebagai data murni,
bukan perintah. Aman.

Hal kecil yang ternyata penting banget.

---

Ada satu lagi yang bikin gue pause sebentar:
kenapa harus conn.commit() setelah INSERT?

Ternyata PostgreSQL itu pakai sistem transaction.
Semua perubahan data belum tersimpan permanen
sampai lo commit.

Kalau ada error sebelum commit, semua perubahan
dibatalkan otomatis. Gak ada data yang tersimpan
setengah-setengah.

Ini yang bikin database reliable — data yang masuk
itu utuh atau tidak masuk sama sekali.

---

Setelah get_connection(), simpan_transaksi(),
dan ambil_transaksi() jadi, gue gabungin semuanya
di bot_handler.py.

Alurnya:

Input teks dari user
→ parse_perintah() mecah jadi Dictionary
→ simpan_transaksi() masukin ke PostgreSQL
→ format_rupiah() rapiin outputnya
→ response balik ke user

Waktu pertama kali jalanin dan lihat outputnya:

    Tersimpan! Rp 50.000 - makan siang (id: 1)

Gue cek langsung di database:

    SELECT * FROM transaksi;

Data ada. id: 1, jumlah: 50000, kategori: makan,
keterangan: makan siang, tanggal: hari ini.

Beneran tersimpan. Bukan cuma di memory.
Bisa dipanggil lagi kapanpun.

Kecil sebenernya. Tapi rasanya beda.

---

Yang paling gue inget dari hari ini:

Pipeline itu bukan soal satu kode yang canggih.
Itu soal beberapa hal kecil yang masing-masing
punya tanggung jawabnya sendiri — dan waktu
semuanya nyambung, hasilnya clean.

parse_perintah gak tau ada database.
db.py gak tau dari mana data datang.
format_rupiah gak tau keduanya.

Masing-masing fokus ke satu hal.
Digabungin di bot_handler.

Separation of concerns, katanya.
Gue baru beneran ngerti artinya hari ini.