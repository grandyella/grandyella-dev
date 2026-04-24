---
title: "Python pertama: Dictionary, Function, dan kenapa gue salah paham soal []"
description: "Belajar hal yang kedengarannya simpel tapi ternyata ada nuansanya."
pubDate: "Apr 18 2026 11:00:00"
---

Gue udah tau Variable, List, sama Loop.
Jadi gue pikir Dictionary bakal gampang.

Lumayan gampang sih. Tapi ada satu hal yang
bikin gue salah paham cukup lama.

---

Dictionary itu basically pasangan key dan value.

    transaksi = {
        "jumlah": 50000,
        "kategori": "makan",
        "keterangan": "makan siang"
    }

Mau ambil jumlahnya? tulis transaksi["jumlah"].
Dapat 50000. Simpel.

Masalahnya waktu gue coba akses key yang gak ada:

    transaksi["merchant"]

Error. KeyError. Program berhenti.

Gue pikir masalahnya di [] — "oh berarti [] gak bisa
nerima data apa-apa yang gak ada."

Ternyata bukan gitu.

[] itu fine selama key-nya ada. Yang bikin crash
itu waktu key-nya gak ada dan lo tetep maksa pakai [].

Bedanya sama .get():

    transaksi["merchant"]        # crash kalau gak ada
    transaksi.get("merchant")    # return None, gak crash
    transaksi.get("merchant", "-")  # return "-", gak crash

Setelah paham ini, langsung makes sense kenapa
di dunia nyata orang lebih sering pakai .get() —
data dari API atau input user itu gak bisa dikontrol.
Lebih aman pakai .get() daripada gambling sama [].

---

Lanjut ke nested Dictionary.

Ini yang bikin gue ngerti kenapa harus belajar ini
sebelum bikin bot Telegram.

Data dari Telegram API itu bentuknya kayak gini:

    pesan = {
        "message": {
            "from": {
                "first_name": "Zulfikar"
            },
            "text": "/catat 50000 makan siang"
        }
    }

Mau ambil nama pengirimnya?

    pesan["message"]["from"]["first_name"]

Dibaca dari kiri ke kanan — masuk satu lapisan,
masuk lagi, masuk lagi. Kayak buka kotak di dalam kotak.

Begitu paham ini, struktur data API yang tadinya
keliatan ribet jadi gak se-scary itu.

---

Terus bikin Function pertama.

parse_perintah() — fungsinya mecah teks perintah
dari user jadi Dictionary yang terstruktur.

Input: "/catat 50000 makan siang"
Output: {"perintah": "/catat", "jumlah": 50000, "keterangan": "makan siang"}

Waktu pertama jalan, seneng banget.
Terus langsung kepikiran — apa yang terjadi
kalau user salah kirim? Misalnya "/catat makan siang"
tanpa angka.

Coba jalanin. Langsung:

    ValueError: invalid literal for int() with base 10: 'makan'

Crash. Program berhenti total.

Di bot yang beneran, ini artinya semua user
kena dampaknya — bukan cuma yang salah input.

Jadi tambah validasi:

- Cek dulu apakah ada minimal 3 bagian
- Cek apakah bagian kedua itu angka
- Kalau tidak, return pesan error — jangan crash

Setelah difix, input yang salah dapat response yang
informatif. Program tetap jalan. User lain gak kena.

Ini yang disebut fail fast — deteksi masalah sedini
mungkin, kasih pesan yang jelas, jangan biarkan
data salah masuk lebih dalam ke sistem.

---

Satu lagi yang gue pelajari hari ini: if **name** == "**main**".

Ini standar Python yang gue sering lihat tapi
gak pernah beneran ngerti gunanya sampai sekarang.

Waktu gue import Function dari file lain,
semua print() di file itu ikut jalan.
Output-nya berantakan — campur aduk antara
output file yang diimport sama file utama.

Solusinya: bungkus semua kode test di dalam blok
if **name** == "**main**". Blok itu cuma jalan
kalau file dieksekusi langsung — gak jalan kalau diimport.

Sekarang import bersih, output bersih.

---

Hari ini gak ada yang spektakuler.
Tapi ada beberapa hal kecil yang akhirnya masuk —
dan itu rasanya lebih memuaskan dari sekedar
kode yang jalan tanpa ngerti kenapa.
