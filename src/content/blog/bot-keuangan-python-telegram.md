---
title: "Dari Keresahan Jadi Kode: Bikin Bot Keuangan Pribadi dengan Python & Telegram"
pubDate: 2026-04-21
description: "Cerita di balik Grandyella Money Tracker — dari frustrasi lihat saldo habis tanpa jejak, sampai bot Telegram yang jalan 24 jam di homelab sendiri."
tags: ["python", "telegram", "postgresql", "homelab", "personal-finance"]
---

## Kenapa Artikel Ini Ada

Maret 2026. Gue duduk sambil ngecek saldo rekening dan ngerasa ada yang aneh. Uang kayak habis gitu aja. Nggak ada yang bisa gue tunjuk — nggak ada pengeluaran besar yang obvious, tapi saldo tetap menyusut tiap minggu.

Masalahnya bukan di pengeluaran yang besar. Masalahnya di pengeluaran kecil yang numpuk tanpa gue sadari. Makan siang 25 ribu, kopi 15 ribu, grab 20 ribu — kecil-kecil tapi kalau dijumlah dalam sebulan, lumayan bikin kaget.

Gue nyoba beberapa aplikasi keuangan. Ada yang bagus tapi ribet input-nya. Ada yang simple tapi datanya nggak bisa gue kontrol. Gue nggak mau data keuangan gue ada di server orang lain yang nggak gue kenal.

Terus gue mikir — gue kan lagi belajar Python. Kenapa nggak bikin sendiri?

---

## Apa yang Mau Dibangun

Kriterianya simpel:

- Input cepat — nggak mau buka app khusus, cukup dari Telegram yang udah always open
- Data gue sendiri — disimpan di server sendiri di rumah
- Bisa lihat summary — kemana aja uang gue bulan ini

Dari situ lahirlah **Grandyella Money Tracker** — bot Telegram yang jalan 24 jam di homelab gue sendiri.

---

## Stack yang Dipilih

Sebelum nulis satu baris kode pun, gue mutusin stack-nya dulu:

```
Telegram Bot    — interface input yang familiar
Python          — bahasa yang lagi gue pelajari
PostgreSQL      — database yang proper, bukan SQLite
Docker          — semua service jalan di container
systemd         — bot jalan 24 jam otomatis
```

Semua jalan di Dell OptiPlex 7050 yang gue jadiin homelab di rumah — Ubuntu 24.04, RAM 16GB, storage 256GB SSD.

---

## Struktur Project

```
grandyella_tracker/
├── src/
│   ├── bot.py                 # main bot
│   ├── db.py                  # database layer
│   ├── belajar_dictionary.py  # parse perintah teks
│   ├── format_rupiah.py       # format angka
│   └── .env                   # credentials
├── logs/
│   └── bot.log
└── requirements.txt
```

Gue sengaja pisah `db.py` dari `bot.py` — semua query SQL ada di satu tempat, bot tidak perlu tahu detail database. Ini yang disebut separation of concerns — konsep yang gue pelajari sambil ngerjain project ini.

---

## Database Schema

```sql
CREATE TABLE users (
    id          BIGINT PRIMARY KEY,
    nama        VARCHAR(100),
    username    VARCHAR(100),
    created_at  TIMESTAMP DEFAULT NOW(),
    deleted_at  TIMESTAMP
);

CREATE TABLE transaksi (
    id          SERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),
    jumlah      BIGINT NOT NULL,
    tipe        VARCHAR(20) CHECK (tipe IN ('masuk', 'keluar', 'transfer')),
    kategori    VARCHAR(50),
    keterangan  TEXT,
    tanggal     DATE DEFAULT CURRENT_DATE,
    created_at  TIMESTAMP DEFAULT NOW(),
    bank_id     INTEGER REFERENCES bank(id),
    bank_tujuan_id INTEGER REFERENCES bank(id)
);

CREATE TABLE bank (
    id          SERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),
    nama        VARCHAR(100),
    tipe        VARCHAR(20) CHECK (tipe IN ('bank', 'ewallet', 'cash', 'kartu_kredit')),
    no_rekening VARCHAR(50),
    saldo_awal  BIGINT DEFAULT 0,
    limit_kredit BIGINT,
    created_at  TIMESTAMP DEFAULT NOW()
);
```

Kenapa `deleted_at` di tabel users dan bukan `DELETE`? Ini namanya **soft delete** — data tidak benar-benar dihapus dari database, hanya ditandai kapan dihapus. Kalau user mau balik, datanya masih ada.

---

## Koneksi ke Database

Ini function pertama yang gue tulis di `db.py`:

```python
def get_connection():
    conn = psycopg2.connect(
        host=os.getenv("DB_HOST", "192.168.28.28"),
        port=os.getenv("DB_PORT", 5432),
        database=os.getenv("DB_NAME", "grandyella_db"),
        user=os.getenv("DB_USER", "grandyella"),
        password=os.getenv("DB_PASSWORD"),
        options="-c timezone=Asia/Jakarta",
    )
    return conn
```

Dua hal yang penting di sini:

Pertama — credential disimpan di file `.env`, bukan hardcode di kode. Ini standard keamanan dasar — kalau kode di-push ke GitHub, password tidak ikut ketahuan.

Kedua — `options="-c timezone=Asia/Jakarta"` — ini yang bikin semua query otomatis pakai timezone WIB. Tanpa ini, timestamp tersimpan dalam UTC dan semua jam tampil +7 jam lebih awal dari yang seharusnya. Gue ngalamin bug ini dan butuh beberapa sesi debugging buat nemuin root cause-nya.

---

## Flow Catat Transaksi

Bot punya dua cara input yang gue desain untuk kecepatan:

**Flow A — Ketik langsung:**

```
User: 50000 makan siang
Bot:  💰 Rp 50.000 — makan siang
      Pilih kategori: [🍔 Makan & Minum] [🚗 Transport] ...
User: 🍔 Makan & Minum
Bot:  Dari bank mana? [🏦 BCA] [💵 Cash] ...
User: 🏦 BCA
Bot:  ✅ Tercatat!
```

**Flow B — Lewat tombol:**

```
User: klik 💸 Keluar
Bot:  Berapa jumlahnya?
User: 50000
Bot:  Pilih kategori...
```

Flow A untuk input cepat saat lagi di jalan. Flow B untuk yang lebih teliti. Keduanya berakhir di function yang sama — `simpan_transaksi()`.

---

## Fitur Lengkap

Setelah beberapa minggu development, bot punya fitur:

- **Catat keluar, masuk, transfer** antar akun sendiri
- **Kelola bank** — BCA, BRI, GoPay, Cash, Kartu Kredit dengan tracking saldo
- **Riwayat harian** dengan navigasi per hari
- **Dashboard** harian dan bulanan dengan progress bar per kategori
- **Budget limit** per kategori dengan warning 80% dan over budget
- **Recurring transaction** — transaksi bulanan otomatis di tanggal tertentu
- **Export PDF** laporan bulanan
- **Notifikasi** — summary pagi, reminder siang dan malam

---

## Deploy 24 Jam

Bot jalan sebagai systemd service — artinya otomatis start saat server nyala, restart sendiri kalau crash:

```ini
[Unit]
Description=Grandyella Tracker Bot
After=network.target docker.service

[Service]
Type=simple
User=grandyella
WorkingDirectory=/home/grandyella/grandyella_tracker/dev
ExecStart=/usr/bin/python3 src/bot.py
Restart=always
RestartSec=10
StandardOutput=null
StandardError=null

[Install]
WantedBy=multi-user.target
```

Dengan setup ini, bot tetap jalan meski terminal ditutup, server restart, atau bot crash karena error apapun.

---

## Yang Gue Pelajari

Project ini bukan cuma soal bikin bot. Ini soal belajar problem solving yang nyata:

- **Timezone bug** — data tersimpan UTC, tampil salah WIB. Fix: `AT TIME ZONE 'Asia/Jakarta'` di semua query
- **ConversationHandler stuck** — bot hang karena state tidak reset. Fix: `conversation_timeout=300`
- **Circular import** — `db.py` tidak sengaja import dari dirinya sendiri. Fix: pahami dependency antar file
- **Double logging** — log muncul dua kali karena systemd dan Python handler sama-sama nulis. Fix: `StandardOutput=null` di systemd

Setiap error punya pelajaran. Dan pelajaran itu jauh lebih nempel dari tutorial yang cuma jalan mulus.

---

## Apa Selanjutnya

Bot ini belum selesai. Yang masih dalam pipeline:

- Voice note — input transaksi via suara pakai Whisper API
- Foto struk — OCR otomatis baca nominal dari foto nota
- Deteksi lokasi — catat di mana transaksi dilakukan
- Data pipeline — Airflow + dbt + ClickHouse untuk analitik lebih dalam
- Machine learning — prediksi pengeluaran bulan depan

Tapi untuk sekarang, tujuan awal sudah tercapai — gue tahu kemana aja uang gue pergi.

---

_Reach out via LinkedIn atau website kalau mau ngobrol lebih lanjut._

🌐 [grandyella-dev.vercel.app](https://grandyella-dev.vercel.app)
💼 [linkedin.com/in/ahmadzulfikartanjung](https://linkedin.com/in/ahmadzulfikartanjung)
