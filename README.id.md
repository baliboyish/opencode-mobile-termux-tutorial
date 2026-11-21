# OpenCode di HP — Tutorial Termux + OpenCode Mobile dari Nol (For Dummy)

**Kontrol sesi coding OpenCode kamu dari HP, pakai UI yang elegan — server-nya jalan di HP kamu sendiri (Termux), atau di laptop/desktop kamu.**

Tutorial ini ditulis dari setup nyata yang dibangun dan diuji sendiri (termasuk uji restart HP beneran), bukan teori. Kalau kamu ikuti urutannya, di akhir tutorial app **OpenCode Mobile** di HP-mu akan terhubung ke server OpenCode dan kamu bisa memantau, melanjutkan, dan mengontrol sesi coding dari mana saja — minimal dari kasur.

> **Males baca?** Bagikan link GitHub ini ke AI agent favoritmu dan suruh dia yang memandumu 😬

> **Versi yang dipakai tutorial ini**
> - App: **OpenCode Mobile v0.17.0** (unofficial client oleh beppe2880) — spec API **2.0.27**
> - Server di HP: **OpenCode 2.0.27** di dalam proot Ubuntu (Termux)
> - Server bonus di laptop: OpenCode Desktop **2.3.2** (Bab 8)
>
> Angka versi ini penting. Separuh tutorial ini isinya soal *mencocokkan versi* — baca Bab 2 sebelum lompat ke mana-mana.

---

## Daftar Isi

- [Bab 0 — Gambaran besar](#bab-0--gambaran-besar)
- [Bab 1 — Bahan-bahan](#bab-1--bahan-bahan)
- [Bab 2 — Kenapa app-nya bilang "Not found on the server"](#bab-2--kenapa-app-nya-bilang-not-found-on-the-server)
- [Bab 3 — Cara gampang: suruh agent yang ngerjain](#bab-3--cara-gampang-suruh-agent-yang-ngerjain)
- [Bab 4 — Cara manual: proot Ubuntu + OpenCode 2.0.27](#bab-4--cara-manual-proot-ubuntu--opencode-2027)
- [Bab 5 — Pairing ke app](#bab-5--pairing-ke-app)
- [Bab 6 — Biar awet: auto-start, baterai, start/stop](#bab-6--biar-awet-auto-start-baterai-startstop)
- [Bab 7 — Troubleshooting](#bab-7--troubleshooting)
- [Bab 8 — Bonus: server kedua dari laptop/desktop via Tailscale](#bab-8--bonus-server-kedua-dari-laptopdesktop-via-tailscale)
- [Bab 9 — Keamanan (jangan skip)](#bab-9--keamanan-jangan-skip)
- [Penutup](#penutup)

---

## Bab 0 — Gambaran besar

App OpenCode Mobile itu cuma **remote control**. Dia butuh "mesin" berupa server OpenCode yang jalan di suatu tempat. Tutorial ini memasang mesinnya di dua tempat:

```
┌──────────────────────────────────────────────────────────────┐
│ HP kamu                                                      │
│                                                              │
│  App OpenCode Mobile ──► 127.0.0.1:4096                      │
│                              │                               │
│                     ┌────────▼─────────────┐                 │
│                     │ proot Ubuntu (Termux)│                 │
│                     │ OpenCode 2.0.27      │                 │
│                     └──────────────────────┘                 │
│                                                              │
│  (Bonus, Bab 8) app yang sama juga bisa menunjuk ke:         │
│  Laptop/Desktop ──► http://<ip-tailscale-laptop>:4096        │
│                     OpenCode Desktop 2.3.2                   │
└──────────────────────────────────────────────────────────────┘
```

Kenapa server-nya harus pakai proot Ubuntu, nggak cukup OpenCode Termux biasa? Itu inti masalahnya — lanjut ke Bab 2. Tapi sebelumnya, siapkan bahan-bahannya dulu.

---

## Bab 1 — Bahan-bahan

1. **HP Android** (tutorial ini diuji di Android 14, tanpa root).
2. **Termux** — install dari [F-Droid](https://f-droid.org/packages/com.termux/) atau dari [GitHub Releases](https://github.com/termux/termux-app/releases).
   ⚠️ **Penting:** semua add-on Termux (Termux:Boot, Termux:API, dll.) harus dari **sumber yang sama** dengan Termux-mu, karena tanda tangan digitalnya harus cocok. Termux dari F-Droid → add-on dari F-Droid. Kalau install add-on ditolak Android dengan error *signature/conflict*, itu penyebabnya.
3. **OpenCode Mobile** — download dari repo resminya:
   - Repo: https://github.com/beppe2880/opencode-mobile
   - Release v0.17.0: https://github.com/beppe2880/opencode-mobile/releases/tag/v0.17.0
4. **OpenCode di Termux** (cara install standarnya, misalnya build komunitas guysoft) — ini yang akan jadi bahan bakar Bab 3. Kalau kamu sudah punya OpenCode jalan di Termux, kamu sudah punya modal utama tutorial ini.
5. **Termux:Boot** — nanti untuk Bab 6, install dari sumber yang sama dengan Termux-mu (mis. [F-Droid](https://f-droid.org/packages/com.termux.boot/)).
6. Opsional tapi sangat disarankan: **Tailscale** di HP dan laptop (untuk Bab 8).

---

## Bab 2 — Kenapa app-nya bilang "Not found on the server"

Ini jebakan pertama yang hampir pasti kamu temui, dan penyebabnya **bukan** password salah.

App OpenCode Mobile v0.17.0 memeriksa identitas server lewat endpoint:

```
GET /api/info
```

Endpoint itu bagian dari spec API **2.0.27**. Masalahnya, OpenCode native Termux yang umum beredar (build guysoft) versinya **2.0.0** — dia cuma punya endpoint lama `/api/server`, dan `/api/info` di dia **tidak ada (404)**. App menerjemahkan 404 itu menjadi pesan:

> **"Not found on the server"**

Cek sendiri server lamamu:

```bash
opencode service status
# → http://127.0.0.1:28570  (versi 2.0.0-android-termux.2)

curl -s http://127.0.0.1:28570/api/server   # → 200, servernya hidup
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:28570/api/info
# → 404  ← ini biang masalahnya
```

Kesimpulan kerasnya:

- **Downgrade app tidak membantu** — semua rilis app memakai `/api/info`.
- **Upgrade native Termux tidak tersedia** — build native terbaru masih 2.0.0.
- **Spec-nya harus cocok.** App v0.17.0 = 118 path API. Server yang benar harus menyediakan 118 path itu (persis, atau superset-nya seperti di Bab 8).

Jadi solusinya: jalankan **OpenCode 2.0.27** — dan karena build native-nya tidak ada, kita jalankan dia di dalam **proot Ubuntu** di Termux. Kedengarannya ribet, dan memang ribet — makanya ada Bab 3.

---

## Bab 3 — Cara gampang: suruh agent yang ngerjain

Kamu sudah punya OpenCode jalan di Termux? Berarti kamu punya *agent* yang bisa disuruh membangun semua ini untukmu. Ini cara yang dipakai penulis tutorial ini, dan berhasil.

1. Buka Termux, jalankan:

   ```bash
   opencode
   ```

2. Paste link repo app-nya, lalu bilang persis seperti ini:

   ```
   hi installed this app on my phone - and I want to login - provide me with correct url and password to login
   ```

3. Udah. Agent-nya yang akan mendiagnosis mismatch versi, memasang proot Ubuntu, memasang OpenCode 2.0.27, memigrasikan database sesimu, membuat password, dan menyerahkan URL + username + password yang benar untuk login di app.

Yang perlu kamu lakukan cuma **mengawasi dan membaca laporannya** — terutama:
- Catat URL, username, dan password yang dia berikan.
- Pastikan server-nya hanya bind ke `127.0.0.1` (lihat Bab 9).
- Minta dia membuatkan dokumen handover supaya kamu (dan AI lain yang membantumu) tahu persis apa yang sudah terpasang di HP-mu.

Kalau kamu tipe yang mau paham semua langkahnya — atau agent-mu menyerah di tengah — jalur manualnya ada di Bab 4.

---

## Bab 4 — Cara manual: proot Ubuntu + OpenCode 2.0.27

> Ringkasan jalur ini: install Ubuntu di Termux pakai proot-distro → install OpenCode **2.0.27** di dalamnya → (opsional) migrasikan database sesi dari server lama → buat script start/stop dengan password tetap → jalankan server di `127.0.0.1:4096`.

### 4.1 Install Ubuntu di Termux

```bash
pkg update && pkg install proot-distro
proot-distro install ubuntu
```

Rootfs Ubuntu ukurannya sekitar 1,7 GB. Sabar, ini unduhan terbesar di tutorial ini.

### 4.2 Install OpenCode 2.0.27 di dalam Ubuntu

Masuk ke Ubuntu, lalu install dengan versi yang **dikunci**:

```bash
proot-distro login ubuntu
curl -fsSL https://opencode.ai/v2/install | bash -s -- --version 2.0.27 --no-modify-path
export PATH="$HOME/.opencode/bin:$PATH"
opencode --version   # harus: 2.0.27
exit
```

⚠️ Jangan asal install versi terbaru. App v0.17.0 pasangannya adalah 2.0.27. (Kalau app-nya kelak naik spec, sesuaikan angka ini — caranya di Bab 7.)

### 4.3 (Opsional) Migrasikan sesi dari server lama

Kalau kamu sudah punya sesi di server native lama dan mau membawanya, backup databasenya **selagi server lama masih jalan** (online-safe):

```bash
# di Termux (bukan di dalam proot)
sqlite3 ~/.local/share/opencode/opencode.db ".backup ~/.local/share/opencode/ocm-migrate.db"
```

Lalu taruh snapshot itu sebagai database server baru di dalam rootfs:

```bash
proot-distro login ubuntu -- env -i HOME=/root PATH=/usr/bin:/bin bash -lc '
  D=/root/.local/share/opencode
  mkdir -p "$D"
  rm -f "$D"/opencode.db "$D"/opencode.db-shm "$D"/opencode.db-wal
  cp /data/data/com.termux/files/home/.local/share/opencode/ocm-migrate.db "$D/opencode.db"'
```

Database akan dimigrasikan otomatis oleh server 2.0.27 saat pertama kali jalan.

### 4.4 Buat script start / stop / shell

Tiga script kecil di `~/bin/` ini yang akan mengoperasikan servermu seterusnya. **Ganti `PASSWORD_KAMU` dengan password acak buatanmu sendiri** (contoh format: 24 karakter hex — buat pakai `openssl rand -hex 12` di Termux).

`~/bin/oc-app-server` — menyalakan server:

```bash
#!/data/data/com.termux/files/usr/bin/bash
export OPENCODE_SERVER_PASSWORD="PASSWORD_KAMU"
export OPENCODE_SERVER_USERNAME="opencode"
proot-distro login ubuntu -- env \
  OPENCODE_SERVER_PASSWORD="$OPENCODE_SERVER_PASSWORD" \
  OPENCODE_SERVER_USERNAME="$OPENCODE_SERVER_USERNAME" \
  HOME=/root PATH=/root/.opencode/bin:/usr/bin:/bin \
  bash -lc 'cd /data/data/com.termux/files/home && opencode serve --port 4096 --hostname 127.0.0.1' \
  >> ~/.local/share/opencode/app-server.log 2>&1
```

`~/bin/oc-app-stop` — mematikan server:

```bash
#!/data/data/com.termux/files/usr/bin/bash
pkill -f "opencode serve --port 4096" 2>/dev/null
echo "server dimatikan"
```

`~/bin/oc-app-shell` — masuk ke Ubuntu dengan environment yang benar (untuk maintenance / `opencode auth login`):

```bash
#!/data/data/com.termux/files/usr/bin/bash
proot-distro login ubuntu -- env HOME=/root PATH=/root/.opencode/bin:/usr/bin:/bin bash "$@"
```

Beri izin eksekusi, dan jaga file start-nya karena password-mu ada di dalamnya:

```bash
chmod 700 ~/bin/oc-app-server ~/bin/oc-app-stop ~/bin/oc-app-shell
mkdir -p ~/.local/share/opencode
```

### 4.5 Jalankan dan verifikasi

```bash
nohup ~/bin/oc-app-server >/dev/null 2>&1 &
sleep 10

# tanpa password → harus 401
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:4096/api/info

# dengan password → harus menampilkan versi 2.0.27
curl -s -u opencode:PASSWORD_KAMU http://127.0.0.1:4096/api/info
# → {"version":"2.0.27", ...}
```

Kalau dua cek itu lolos, servermu siap dipasangkan ke app.

---

## Bab 5 — Pairing ke app

1. Buka **OpenCode Mobile**.
2. Pilih tambah server secara manual, isi:
   - **URL:** `http://127.0.0.1:4096`
   - **Username:** `opencode`
   - **Password:** password yang kamu buat di Bab 4.4
3. Simpan. Statusnya harus berubah menjadi **Connected**, dan sesi-sesimu (termasuk yang termigrasi) muncul di daftar.

Selamat — remote control-nya sudah nyambung ke mesinnya. Sekarang bikin dia awet.

---

## Bab 6 — Biar awet: auto-start, baterai, start/stop

Server ini hidup di dalam Termux. Artinya: **kalau proses Termux mati, server ikut mati** — dan app tidak bisa menyalakannya sendiri. Bab ini yang membedakan setup "pernah berhasil connect" dengan setup yang awet dipakai harian. Semua poin di bab ini sudah lolos **uji restart HP beneran**.

### 6.1 Perintah harian yang perlu kamu hafal

```bash
# menyalakan server (satu baris ini kuncinya)
nohup ~/bin/oc-app-server >/dev/null 2>&1 &

# mematikan server
~/bin/oc-app-stop

# cek server hidup atau mati — tanpa perlu buka app
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:4096/api/info
# 401 = HIDUP (dia minta password, itu bagus) · 000 = MATI
```

### 6.2 Yang membunuh server (hindari)

- **Swipe-close Termux dari recents** → proses Termux mati → server mati.
- **Force stop Termux** (dari Settings atau tombol EXIT di notifikasi Termux) → sama, mati total.
- Menutup tampilan TUI OpenCode (`Ctrl+C`) itu **aman** — yang mati cuma tampilannya, server tetap hidup.

Kalau server mati karena salah satu di atas: buka Termux, jalankan baris `nohup` di 6.1, tunggu ±10 detik, app akan connect sendiri.

### 6.3 Auto-start setelah reboot: Termux:Boot

1. Install **Termux:Boot** dari sumber yang sama dengan Termux-mu (Bab 1), lalu **buka app-nya sekali** — langkah buka-sekali ini yang mengaktifkannya.
2. Buat file hook `~/.termux/boot/oc-app-server`:

   ```bash
   #!/data/data/com.termux/files/usr/bin/sh
   termux-wake-lock
   nohup ~/bin/oc-app-server >/dev/null 2>&1 &
   ```

3. Beri izin eksekusi: `chmod +x ~/.termux/boot/oc-app-server`

### 6.4 Jinakkan penghemat baterai

Di Settings HP-mu:

- **Apps → Termux → Battery → Unrestricted** (di OxygenOS: nyalakan **Allow background activity**; "Allow foreground activity" juga biarkan menyala).
- Kunci Termux di recents (ikon gembok) supaya tidak gampang ke-swipe bersih oleh sistem.

### 6.5 Uji restart (lakukan sekali, biar percaya)

Restart HP-mu. Setelah nyala, **buka Termux sekali** — di banyak ROM (termasuk OxygenOS), proses Termux "dibangunkan" penuh saat app-nya dibuka, dan server hasil boot hook butuh itu. Tunggu ±1–5 menit (start proot pasca-boot memang lambat), lalu buka OpenCode Mobile: dia harus **Connected sendiri, tanpa kamu menjalankan perintah apa pun**.

Rumus akhirnya simpel: **habis restart → buka Termux sekali → app konek.**

---

## Bab 7 — Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| App: *"Not found on the server"* | Kamu menunjuk ke server 2.0.0 (port 28570) atau server versi lain yang tidak punya `/api/info` | Pastikan URL `:4096`; cek `curl .../api/info` harus menjawab versi yang cocok dengan app |
| App: *Cannot reach the server* / Connection refused | Server 4096 mati | Jalankan baris `nohup` di Bab 6.1 |
| 401 Unauthorized | Password salah | Cek `OPENCODE_SERVER_PASSWORD` di `~/bin/oc-app-server`; samakan dengan yang di app |
| Habis reboot tidak connect | Termux:Boot belum ada / hook belum jalan / Termux belum dibuka | Bab 6.3 + buka Termux sekali (Bab 6.5) |
| Server mati sendiri di tengah hari | Dibunuh penghemat baterai / ke-swipe | Bab 6.4; kunci Termux di recents |
| App connect tapi chat error | Belum ada credential provider | `~/bin/oc-app-shell` → `opencode auth login` (pakai API key-mu sendiri), atau pakai model free bawaan `opencode` |
| Sesi lama tidak muncul | Database belum dimigrasikan | Bab 4.3 |
| App habis update, tiba-tiba tidak cocok lagi | App naik spec (mis. ke 2.3.2) | Upgrade server ke versi spec app: backup DB dulu, lalu di dalam proot: `curl -fsSL https://opencode.ai/v2/install \| bash -s -- --version <versi-target> --no-modify-path`, restart server |
| Mau cek spec cocok atau tidak | — | Bandingkan `api/openapi.json` dari repo app (di tag versinya) dengan `/openapi.json` dari servermu: semua path app harus ada di server |

---

## Bab 8 — Bonus: server kedua dari laptop/desktop via Tailscale

App-nya mendukung **banyak server tersimpan**. Kalau kamu punya OpenCode Desktop di laptop, kamu bisa menambahkan dia sebagai server kedua dan mengontrol kerjaan desktop dari HP.

Prinsipnya sama — **spec harus cocok** — tapi kabar baiknya: OpenCode Desktop 2.3.2 menyediakan **142 path**, superset penuh dari 118 path yang dibutuhkan app v0.17.0 (terverifikasi: 0 path hilang). Jadi app lama pun bisa connect ke desktop baru.

Langkah ringkas (Windows):

1. Ambil binary CLI dari instalasi desktop-mu, contoh: `%APPDATA%\ai.opencode.desktop\cli\2.3.2-x64\opencode-cli.exe`. (Jangan pakai CLI npm lama kalau ada — versi 1.x tidak kompatibel databasenya dengan jalur 2.x.)
2. Siapkan folder khusus dengan **data home terpisah** supaya database sesi server ini tidak bertabrakan dengan database app desktop:

   ```bat
   set OPENCODE_SERVER_PASSWORD=PASSWORD_BARU_KAMU
   set OPENCODE_SERVER_USERNAME=opencode
   set XDG_DATA_HOME=C:\path\ke\folder-server\data
   set XDG_CONFIG_HOME=C:\path\ke\folder-server\config
   opencode-cli.exe serve --port 4096 --hostname <IP-TAILSCALE-LAPTOP>
   ```

3. Bikin dia awet: bungkus perintah di atas dalam script `.cmd` dengan loop auto-restart, lalu daftarkan sebagai **Scheduled Task** yang jalan saat logon (matikan opsi "stop if on batteries" kalau laptopmu sering dipakai tanpa colokan).
4. Di app, tambah server baru:
   - **URL:** `http://<IP-TAILSCALE-LAPTOP>:4096`
   - **Username:** `opencode`
   - **Password:** password barumu di langkah 2

Kenapa bind ke IP Tailscale, bukan `0.0.0.0`? Karena servermu jadi hanya bisa dijangkau lewat jaringan Tailscale-mu sendiri (terenkripsi, dari mana saja) — tidak terekspos ke Wi-Fi lokal, apalagi internet. Konsekuensi databasenya terpisah: sesi lama di app desktop tidak tampil di HP, tapi sesi baru yang kamu mulai dari HP **jalan beneran di laptop**.

---

## Bab 9 — Keamanan (jangan skip)

- **Server HP hanya bind `127.0.0.1`.** Jangan pernah mengubahnya ke `0.0.0.0` dan jangan pernah mem-forward port 4096 ke internet. Siapa pun yang bisa mencapai port itu hanya berhadapan dengan satu password.
- Untuk akses lintas perangkat, **pakai Tailscale/WireGuard** (seperti Bab 8), bukan expose port.
- Password server tersimpan plaintext di script start (`chmod 700`). Perlakukan file itu — dan **screenshot apa pun yang memuat password-mu** — sebagai rahasia. Jangan upload dokumen handover yang masih memuat password asli ke tempat publik. (Iya, ini ditulis berdasarkan pengalaman.)
- Ganti password = edit `OPENCODE_SERVER_PASSWORD` di script, restart server, update juga di app.
- proot itu lapisan kompatibilitas, **bukan** sandbox keamanan.

---

## Penutup

Recap perjalananmu kalau kamu sampai di sini dengan app berstatus **Connected**:

1. Kamu tahu kenapa "Not found on the server" muncul (spec mismatch, bukan salah password).
2. Kamu punya server OpenCode 2.0.27 di proot Ubuntu, sesi lamamu ikut termigrasi.
3. Kamu bisa menyalakan/mematikan/mengeceknya dengan tiga perintah pendek.
4. Dia auto-start setelah reboot (cukup buka Termux sekali), dan tidak gampang dibunuh baterai.
5. Bonus: laptop/desktop-mu pun bisa kamu setir dari HP yang sama.

**Tautan penting**
- OpenCode Mobile (app): https://github.com/beppe2880/opencode-mobile
- Download v0.17.0: https://github.com/beppe2880/opencode-mobile/releases/tag/v0.17.0
- Termux: https://f-droid.org/packages/com.termux/
- Termux:Boot: https://f-droid.org/packages/com.termux.boot/

*Tutorial ini didasarkan pada setup nyata di satu HP Android + satu laptop Windows, dibangun dan diverifikasi langkah demi langkah pada 21 November 2026 — termasuk bagian yang gagal di tengah jalan, supaya kamu tidak perlu mengulang kegagalan yang sama.*
