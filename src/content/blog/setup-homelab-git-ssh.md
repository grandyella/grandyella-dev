---
title: "Hari pertama: Gitea mati, SSH berantakan, tapi somehow kelar"
description: "Setup environment dari nol — lebih ribet dari yang gue kira."
pubDate: "Apr 18 2026 10:00:00"
---

Gue pikir hari pertama bakal langsung coding.

Ternyata setengah hari pertama habis buat benerin
hal-hal yang harusnya udah jalan.

---

Mulai dari Gitea yang mati.

Gitea itu self-hosted Git server — semacam GitHub
tapi jalan di server gue sendiri. Waktu gue buka
di browser, yang muncul cuma 502 Bad Gateway.

Bukan error yang informatif. Tapi setidaknya
gue tau Nginx-nya masih hidup — yang mati itu
Gitea di belakangnya.

Cek container, ternyata Gitea baru "Up 7 seconds."
Artinya dia terus crash dan restart sendiri.
Baca log-nya, ketemu ini:

    pq: password authentication failed for user "gitea"

Gitea gak bisa konek ke database-nya sendiri.
Password yang ada di konfigurasi tidak cocok
dengan yang tersimpan di PostgreSQL.

Entah kapan ini berubah. Yang jelas harus difix.

Masuk ke dalam container database, ganti password,
update konfigurasi di Portainer. Gitea hidup lagi.

Lesson learned: 502 itu bukan error Nginx.
Itu Nginx bilang "gue hidup, tapi yang lo cari di belakang gue mati."
Selalu cek log container dulu sebelum panik.

---

Setelah Gitea jalan, giliran setup SSH.

Tujuannya biar gue bisa konek dari laptop ke server
dan ke Gitea tanpa ketik password setiap saat.
Pakai SSH key — semacam kunci digital yang
lebih aman dari password biasa.

Generate key di laptop, copy ke server, daftarin ke Gitea.
Straightforward di atas kertas.

Di prakteknya, gue stuck cukup lama di satu masalah:

    git@gitea.server.home's password:

Gitea minta password terus padahal key sudah didaftarin.

Ternyata masalahnya di port. SSH default itu port 22.
Tapi Gitea SSH jalan di port 2222 — karena port 22
sudah dipakai Ubuntu sendiri.

Jadi selama ini SSH gue konek ke Ubuntu,
bukan ke Gitea. Wajar ditolak.

Fix: tambah satu entry di SSH config dengan port 2222.
Setelah itu langsung dapat:

    Hi there, grandyella! You've successfully authenticated.

Rasanya puas banget buat hal yang sebenernya kecil.

---

Lanjut setup repository.

Buat struktur 3 environment: dev, staging, prod.
Ini yang biasa dipakai perusahaan — dev buat
eksperimen, staging buat test, prod buat yang
beneran dipakai orang.

Branch main di-protect supaya tidak bisa di-push
langsung. Semua perubahan harus lewat Pull Request.

Gue test sendiri — coba push ke main dari terminal.
Langsung ditolak:

    error: Not allowed to push to protected branch main

Berarti jalan. Justru bagus kalau ditolak.

---

Terakhir, setup VS Code Remote SSH.

Ini favorit gue dari semua yang dikerjain hari ini.

VS Code di laptop, tapi semua file ada di server.
Nulis kode di laptop, dijalanin di server.
Terminal di VS Code juga langsung masuk ke server —
gak perlu buka SSH session terpisah.

Workflow-nya bersih. Gue suka.

---

Hari pertama gak ada satu baris kode yang ditulis
buat project utama. Tapi setup yang bener di awal
itu penting — daripada ribet di tengah jalan
waktu lagi asyik-asyiknya coding.

Besok baru mulai yang beneran.
