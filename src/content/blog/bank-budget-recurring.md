---
title: "Fitur Lengkap Money Tracker Bot: Bank, Budget, dan Recurring Transaction"
pubDate: 2026-04-23
description: "Dari bot sederhana catat pengeluaran, ke sistem keuangan yang track saldo per rekening, kasih warning budget, dan catat transaksi rutin otomatis."
tags: ["python", "telegram", "bot", "personal-finance", "postgresql"]
---

## Dari Simple ke Proper

Versi pertama bot ini cuma bisa satu hal — catat pengeluaran. Ketik angka, pilih kategori, selesai.

Tapi makin lama dipakai, makin banyak pertanyaan yang muncul:

_"Uang ini keluar dari rekening mana?"_
_"Bulan ini udah habis berapa untuk makan?"_
_"Netflix tiap bulan harus input manual terus?"_

Tiga pertanyaan itu yang akhirnya jadi tiga fitur besar di iterasi berikutnya — **Bank Management**, **Budget Limit**, dan **Recurring Transaction**.

---

## Bank Management — Uang Ada Dimana?

Masalah klasik orang yang punya lebih dari satu rekening — susah track uang ada di mana. Gue sendiri punya BCA, BRI, GoPay, dan kartu kredit. Kalau cuma catat "keluar 50 ribu", gue nggak tahu itu dari rekening mana.

Solusinya: setiap transaksi sekarang wajib pilih sumber dana.

**Empat tipe bank yang didukung:**

```
🏦 Bank          — rekening tabungan konvensional
💳 E-Wallet      — GoPay, OVO, Dana, dll
💵 Cash          — uang tunai
💳 Kartu Kredit  — dengan tracking limit dan tagihan
```

Kartu kredit punya logika berbeda dari yang lain. Kalau bank biasa:

```
Saldo = saldo awal + masuk - keluar
```

Kartu kredit:

```
Tagihan    = total belanja bulan ini
Limit sisa = limit kartu - tagihan
```

Ini penting karena saldo kartu kredit itu hutang, bukan aset.

**Transfer antar akun** juga dihandle dengan tipe transaksi khusus — bukan keluar, bukan masuk, tapi **transfer**. Kenapa perlu dibedakan? Karena tarik tunai dari ATM itu bukan pengeluaran — uang tidak hilang, hanya pindah dari rekening ke dompet. Kalau dicatat sebagai keluar, total kekayaan kelihatan berkurang padahal tidak.

```
Tarik tunai Rp 500.000:
BCA  → -500.000
Cash → +500.000
Net  →       0
```

---

## Budget Limit — Berapa Batas Pengeluaran?

Tahu kemana uang pergi itu satu hal. Tahu kapan harus berhenti itu hal lain.

Budget limit memungkinkan set batas pengeluaran per kategori per bulan:

```
🍔 Makan & Minum    → Rp 2.000.000/bulan
🚗 Transport        → Rp 500.000/bulan
🛒 Belanja          → Rp 1.500.000/bulan
```

Bot otomatis cek realisasi vs budget setiap malam jam 20:00 dan kirim warning:

```
⚠️ Budget Warning

🟡 Makan & Minum — 82% terpakai
   Sisa: Rp 360.000

🔴 Transport — OVER BUDGET!
   Terpakai: Rp 520.000 / Rp 500.000
```

Di dashboard Metabase, ini juga keliatan visual lewat query Budget vs Realisasi — langsung keliatan kategori mana yang sudah merah.

**Implementasi di database:**

```sql
CREATE TABLE budget (
    id       SERIAL PRIMARY KEY,
    user_id  BIGINT REFERENCES users(id),
    kategori VARCHAR(50),
    jumlah   BIGINT,
    bulan    INTEGER,
    tahun    INTEGER,
    UNIQUE(user_id, kategori, bulan, tahun)
);
```

`UNIQUE constraint` di sini penting — satu user tidak bisa punya dua budget untuk kategori yang sama di bulan yang sama. Kalau set ulang, pakai `ON CONFLICT DO UPDATE` — otomatis update bukan insert duplikat.

---

## Recurring Transaction — Jangan Input Manual Terus

Beberapa pengeluaran datang rutin setiap bulan — gaji, tagihan Netflix, cicilan, iuran. Membosankan kalau harus input manual setiap bulan.

Recurring transaction menyelesaikan ini — set sekali, bot catat otomatis di tanggal yang ditentukan:

```
Tanggal 1  → Gaji masuk ke BCA
Tanggal 5  → Netflix Rp 54.000 dari BCA
Tanggal 25 → Cicilan motor Rp 800.000 dari BRI
```

Setiap hari jam 06:00 WIB, scheduler cek apakah ada recurring yang perlu dieksekusi hari ini. Kalau ada, transaksi otomatis tercatat dan bot kirim notifikasi:

```
🔁 Recurring otomatis tercatat!
💸 Rp 54.000
📝 Netflix
🗓 2026-04-05
#️⃣ ID: 47
```

User juga bisa pause recurring tertentu pakai fitur toggle — misalnya bulan ini skip Netflix karena lagi hemat.

**Implementasi scheduler pakai APScheduler:**

```python
scheduler = AsyncIOScheduler(timezone="Asia/Jakarta")
scheduler.add_job(
    eksekusi_recurring,
    CronTrigger(hour=6, minute=0, timezone="Asia/Jakarta"),
    args=[application.bot]
)
```

Kenapa APScheduler bukan cron Linux? Karena notifikasi butuh akses ke bot instance yang jalan di Python. Dengan APScheduler, scheduler dan bot jalan dalam satu proses — tidak perlu spawn process baru setiap trigger.

---

## Semua Notifikasi yang Berjalan

Sekarang ada 6 scheduler yang berjalan otomatis setiap hari:

```
06:00 — Eksekusi recurring transaction
07:00 — Summary pengeluaran kemarin
07:30 — Reminder kalau belum ada transaksi hari ini
12:30 — Reminder kalau belum ada transaksi hari ini
17:30 — Reminder kalau belum ada transaksi hari ini
20:00 — Reminder harian + warning budget
22:30 — Reminder kalau belum ada transaksi hari ini
```

Reminder hanya kirim kalau memang belum ada transaksi — tidak spam kalau sudah input. Warning budget hanya kirim kalau ada kategori yang sudah ≥80%.

---

## Arsitektur Final

Setelah semua fitur ditambahkan, arsitektur bot jadi lebih complex tapi tetap terorganisir:

```
bot.py               → handler, conversation flow, notifikasi
db.py                → semua query SQL (transaksi, bank, budget, recurring)
format_rupiah.py     → format angka ke Rupiah (support negatif)
belajar_dictionary.py → parse teks input langsung
```

Total ada 23 state di ConversationHandler — dari `TUNGGU_JUMLAH` sampai `TUNGGU_RECURRING_TOGGLE`. Setiap flow punya timeout 5 menit — kalau user tidak reply, state otomatis reset dan keyboard kembali normal.

---

## Pelajaran dari Fitur Ini

Tiga fitur ini ngajarin gue beberapa hal di luar syntax Python:

**Database design matters.** Keputusan awal pakai `bank_id` nullable di tabel transaksi memungkinkan data lama tetap valid saat fitur bank ditambahkan. Kalau dari awal `NOT NULL`, semua data lama jadi invalid.

**Timezone itu subtle bug.** Hampir setiap query harus pakai `AT TIME ZONE 'Asia/Jakarta'` — kalau lupa satu saja, data filter salah dan user lihat transaksi di hari yang tidak tepat.

**User experience itu penting.** Bot yang powerful tapi ribet dipakai tidak akan dipakai. Setiap flow dirancang seminimal mungkin tap — input yang bisa di-skip, default yang masuk akal, dan selalu ada tombol Home untuk escape dari flow yang salah.

---

## Status Project Sekarang

Bot sudah production, jalan 24 jam, dipakai beberapa orang. Dashboard Metabase sudah connect dan auto refresh. Yang masih dalam pipeline:

- Voice note — input via suara pakai Whisper API
- Foto struk — OCR otomatis dari foto nota
- Deteksi lokasi — catat di mana transaksi dilakukan
- Data pipeline — Airflow + dbt + ClickHouse
- Prediksi pengeluaran — machine learning untuk forecast

Project ini belum selesai. Dan itu yang bikin seru.

---

🌐 [grandyella-dev.vercel.app](https://grandyella-dev.vercel.app)
💼 [linkedin.com/in/ahmadzulfikartanjung](https://linkedin.com/in/ahmadzulfikartanjung)
