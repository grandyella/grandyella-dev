---
title: "Kenapa Grandyella Tracker?"
description: "Awal mula ide dan brainstorming project ini."
pubDate: "Apr 17 2026 09:00:00"
---

Jadi gini ceritanya.

Gue kerja di finance dan IT. Sehari-hari ngurusin angka,
rekonsiliasi, monitoring sistem — hal-hal yang sebenernya
banyak datanya tapi gak pernah gue olah dengan bener.

Suatu hari gue sadar: gue kerja di dunia data tapi
gak punya kontrol atas data keuangan gue sendiri.
Gue gak tau berapa yang keluar bulan ini, sektor mana
yang gue belanjain paling banyak, dan apakah kondisi
kas gue lagi siap buat invest atau nggak.

Dari situ mulai iseng — kalau gue bikin sendiri gimana?

---

Awalnya konsepnya sederhana. Bot Telegram buat catat
pengeluaran. Simpan ke database. Lihat summary.
Standar banget.

Tapi makin dipikirin, makin berkembang.

Gue juga penasaran sama saham IDX. Selama ini analisa
saham dan kondisi keuangan pribadi itu dua hal yang
terpisah — Stockbit di sana, catatan keuangan di sini.
Gak ada yang nyambungin keduanya.

Nah, di situlah idenya mulai keliatan menarik.

Bagaimana kalau ada satu sistem yang tahu kondisi
cash flow gue sekarang, sekaligus kasih tahu apakah
ini waktu yang tepat buat masuk ke saham tertentu?

Bukan rekomendasi generik. Tapi rekomendasi yang
beneran tahu kondisi finansial gue hari ini.

---

Dari situ gue mulai brainstorming serius.

Namanya Grandyella Tracker. Stack-nya: Python,
PostgreSQL, ClickHouse, Airflow, dbt, Telegram Bot.
Semuanya running di homelab gue sendiri — server
Dell OptiPlex bekas yang nyala 24 jam di rumah.

Fitur yang mau dibangun banyak. Terlalu banyak
untuk dikerjain sekaligus. Jadi gue mulai dari yang
paling basic: bisa catat transaksi via Telegram,
simpan ke database, dan ambil lagi.

Itu dulu. Sisanya nanti.

---

Masih jauh dari selesai. Tapi progresnya ada,
dan gue dokumentasiin di sini.

Kalau lo penasaran sama prosesnya, stay tuned.
