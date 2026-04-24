---
title: "Personal Finance Dashboard dengan Metabase & PostgreSQL"
date: 2026-04-24
description: "Data sudah terkumpul di PostgreSQL — sekarang gimana cara lihatnya secara visual tanpa keluar uang? Jawaban: Metabase yang sudah ada di Docker stack sendiri."
tags: ["metabase", "postgresql", "dashboard", "data", "homelab"]
---

## Data Ada, Tapi Kemana?

Setelah bot jalan beberapa minggu, database mulai terisi. Transaksi masuk, keluar, transfer — semua tercatat rapi di PostgreSQL. Tapi ada satu masalah: data itu "terkunci" di dalam database. Untuk lihat tren, gue harus query manual di DBeaver.

Nggak praktis.

Gue butuh sesuatu yang bisa gue buka dari browser, langsung keliatan datanya dalam bentuk visual. Dan jawabannya sudah ada di Docker stack gue sejak awal — **Metabase**.

---

## Kenapa Metabase?

Gue sempat dilema antara Metabase, Power BI, dan Looker. Tapi pilihannya jatuh ke Metabase karena tiga alasan:

Pertama — sudah ada di Docker stack, tinggal diaktifkan. Tidak perlu install apapun.

Kedua — akses via browser dari device apapun. Tidak terikat Windows seperti Power BI Desktop.

Ketiga — data tetap di server sendiri. Tidak ada data keuangan yang keluar ke cloud orang lain.

---

## Setup Koneksi

Metabase sudah jalan di port 3003. Tinggal connect ke PostgreSQL:

```
Host:     192.168.28.28
Port:     5432
Database: grandyella_db
Username: grandyella
```

Satu hal yang perlu diperhatikan — tidak perlu SSL atau SSH tunnel karena Metabase dan PostgreSQL jalan di server yang sama. SSL diperlukan hanya kalau koneksi melewati internet.

Setelah connect, Metabase otomatis scan semua tabel: `bank`, `budget`, `recurring`, `transaksi`, `users`.

---

## Query yang Dibangun

Dashboard dibangun dari 11 query SQL. Semua query pakai variabel opsional `[[...]]` supaya bisa difilter dari dashboard tanpa break kalau filter kosong.

**Number Cards — gambaran cepat:**

```sql
SELECT COALESCE(SUM(jumlah), 0) as total_pemasukan
FROM transaksi
WHERE tipe = 'masuk'
[[AND created_at AT TIME ZONE 'Asia/Jakarta' >= {{tanggal_mulai}}]]
[[AND created_at AT TIME ZONE 'Asia/Jakarta' <= {{tanggal_selesai}}]]
[[AND user_id = {{user_id}}]]
AND DATE_TRUNC('month', created_at AT TIME ZONE 'Asia/Jakarta')
    = DATE_TRUNC('month', NOW() AT TIME ZONE 'Asia/Jakarta')
```

**Pie Chart — distribusi per kategori:**

```sql
SELECT kategori, SUM(jumlah) as total
FROM transaksi
WHERE tipe = 'keluar'
[[AND created_at AT TIME ZONE 'Asia/Jakarta' >= {{tanggal_mulai}}]]
[[AND created_at AT TIME ZONE 'Asia/Jakarta' <= {{tanggal_selesai}}]]
[[AND user_id = {{user_id}}]]
AND DATE_TRUNC('month', created_at AT TIME ZONE 'Asia/Jakarta')
    = DATE_TRUNC('month', NOW() AT TIME ZONE 'Asia/Jakarta')
GROUP BY kategori
ORDER BY total DESC
```

**Line Chart — tren 30 hari:**

```sql
SELECT
    DATE(created_at AT TIME ZONE 'Asia/Jakarta') as tanggal,
    SUM(jumlah) as total
FROM transaksi
WHERE tipe = 'keluar'
AND (created_at AT TIME ZONE 'Asia/Jakarta')
    >= (NOW() AT TIME ZONE 'Asia/Jakarta') - INTERVAL '30 days'
GROUP BY DATE(created_at AT TIME ZONE 'Asia/Jakarta')
ORDER BY tanggal ASC
```

Catatan penting — semua query pakai `AT TIME ZONE 'Asia/Jakarta'`. Tanpa ini, tanggal dan waktu tampil dalam UTC dan semua grafik tren jadi off 7 jam.

---

## Layout Dashboard

```
┌─────────────┬─────────────┬─────────────┬─────────────┐
│ Total Masuk │ Total Keluar│ Saldo Bersih│ Jml Transaksi│
├─────────────────────┬───────────────────────────────────┤
│ Keluar per Kategori │      Masuk per Kategori           │
├─────────────────────────────────────────────────────────┤
│              Tren Pengeluaran 30 Hari                   │
├─────────────────────────────────────────────────────────┤
│           Pengeluaran vs Pemasukan Harian               │
├─────────────────────────────────────────────────────────┤
│                   Saldo per Bank                        │
├─────────────────────────┬───────────────────────────────┤
│    Budget vs Realisasi  │   Top 10 Transaksi Terbesar   │
└─────────────────────────┴───────────────────────────────┘
```

Auto refresh setiap 30 menit — dashboard selalu up to date tanpa perlu refresh manual.

---

## Insight yang Langsung Keliatan

Begitu dashboard jalan dengan data nyata, beberapa hal langsung obvious:

Pengeluaran terbesar gue bukan di kategori yang gue kira. Gue pikir transport yang paling besar, ternyata Investasi & Tabungan dan Lainnya yang dominan.

Tren pengeluaran ada polanya — awal bulan dan akhir bulan cenderung lebih tinggi dari pertengahan bulan.

Saldo per bank langsung keliatan mana yang minus — terutama kartu kredit yang memang by design negatif karena itu tagihan, bukan saldo.

---

## Apa Selanjutnya

Dashboard ini masih bisa dikembangkan lebih jauh:

- Filter by user — untuk multi-user tracking
- Filter rentang tanggal — untuk analisa periode tertentu
- Integrasi ClickHouse — untuk query analitik yang lebih cepat di data besar
- Prediksi pengeluaran — machine learning sederhana untuk forecast bulan depan

Tapi untuk sekarang, tujuannya sudah tercapai — data yang tadinya "terkunci" di database sekarang bisa dilihat siapapun dalam hitungan detik.

---

🌐 [grandyella-dev.vercel.app](https://grandyella-dev.vercel.app)
💼 [linkedin.com/in/ahmadzulfikartanjung](https://linkedin.com/in/ahmadzulfikartanjung)
