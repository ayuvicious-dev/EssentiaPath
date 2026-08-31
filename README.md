# EssentiaPath

Aplikasi pengingat dengan tampilan kalender. Buat pengingat sekali jalan atau berulang — **tiap jam, harian, hari kerja, akhir pekan, dua mingguan, bulanan, triwulanan, atau tahunan** — dan lihat semuanya dalam satu kalender bulanan, lengkap dengan daftar "Mendatang" untuk 14 hari ke depan.

Dibangun sebagai PWA (Progressive Web App) vanilla HTML/CSS/JS — tanpa build tool, tanpa dependensi eksternal selain font. Bisa dibuka langsung di browser atau dipasang ke layar utama seperti aplikasi native.

## Struktur proyek

```
essentiapath/
├── icons/
│   ├── icon-192.png
│   └── icon-512.png
├── .nojekyll
├── README.md
├── index.html        ← seluruh UI, styling, dan logika aplikasi ada di sini
├── manifest.json
└── service-worker.js
```

**Hampir seluruh aplikasi ada di `index.html`** (HTML + CSS + JavaScript dalam satu file), mengikuti pola yang sudah diminta: kalau ada revisi tampilan, fitur, atau logika pengingat, cukup edit `index.html` saja. File lain (`manifest.json`, `service-worker.js`, ikon) jarang perlu disentuh karena hanya berisi konfigurasi PWA yang wajib terpisah menurut spesifikasi browser.

Kapan file lain perlu diubah:
- `manifest.json` — hanya kalau ingin ganti nama aplikasi, warna tema, atau ikon.
- `service-worker.js` — hanya kalau ingin menambah aset yang di-cache untuk mode offline. **Naikkan angka `CACHE_NAME`** (misal `v1` → `v2`) setiap kali `index.html` berubah, supaya pengguna yang sudah install PWA mendapat versi terbaru, bukan versi lama dari cache.
- `icons/` — hanya kalau ingin mengganti logo.

## Menjalankan secara lokal

Tidak perlu instalasi apa pun. Buka `index.html` langsung di browser, atau jalankan server statis sederhana (disarankan, supaya service worker & PWA berfungsi penuh):

```bash
npx serve .
# atau
python3 -m http.server 8080
```

## Deploy ke GitHub Pages

1. Push folder ini ke repository GitHub (branch `main`, root folder, atau folder `/docs` — sesuaikan pengaturan Pages).
2. File `.nojekyll` sudah disertakan agar GitHub Pages tidak memproses folder ini lewat Jekyll (penting karena nama folder `icons` dan struktur file bisa bentrok dengan aturan Jekyll).
3. Di repo → **Settings → Pages**, pilih branch dan folder tempat file-file ini berada, lalu simpan.
4. Aplikasi akan tersedia di `https://<username>.github.io/<repo>/`.

### Alur revisi

Untuk revisi berikutnya:
1. Edit `index.html` (tambah fitur, ubah desain, perbaiki bug logika pengulangan, dll).
2. Kalau ada aset baru yang perlu tersedia offline, naikkan `CACHE_NAME` di `service-worker.js`.
3. Commit & push — GitHub Pages otomatis mem-publish ulang.

## Fitur

- **Login & registrasi dengan email/kata sandi** (Firebase Authentication). Pengingat hanya terlihat setelah masuk, dan tersimpan per akun.
- **Sinkron ke cloud (Firestore)** — pengingat tersimpan di `users/{uid}/reminders` dan otomatis tersinkron real-time; buka akun yang sama di perangkat lain, pengingatnya ikut muncul.
- **Kalender bulanan** dengan navigasi bulan, penanda "hari ini", dan ringkasan pengingat per hari.
- **Jenis pengulangan**: sekali, tiap jam (interval bisa diatur), harian, hari kerja, akhir pekan, dua mingguan, bulanan, triwulanan, tahunan — dengan tanggal berakhir opsional.
- **Kategori berwarna, bisa dikustomisasi** — kelola daftar kategori (tambah, ubah nama/warna, hapus) dari tab Pengaturan → Kategori. Kategori juga tersinkron ke cloud per akun (`users/{uid}/settings/categories`), dan bisa disembunyikan/ditampilkan sebagai filter dari layar yang sama. Kategori yang masih dipakai pengingat tidak bisa dihapus sampai pengingatnya dipindah dulu.
- **Daftar Mendatang** — semua pengingat 14 hari ke depan, dikelompokkan per tanggal.
- **Notifikasi lokal** — saat aplikasi terbuka di tab/perangkat, EssentiaPath memeriksa pengingat yang jatuh tempo setiap 20 detik dan menampilkan notifikasi browser (perlu izin notifikasi, dan perangkat harus dalam keadaan menjalankan aplikasi — lihat catatan di bawah).
- **Mode offline** dasar via service worker (aset di-cache) + cache offline Firestore (data terakhir tetap terbaca saat koneksi putus, lalu tersinkron ulang begitu online).
- **Bisa dipasang (installable)** ke layar utama/desktop lewat prompt "Pasang aplikasi".

## Menyiapkan Firebase (wajib sebelum pakai)

Aplikasi ini sudah diisi dengan konfigurasi proyek Firebase **essentiapath** langsung di `index.html` (bagian `firebaseConfig`) — nilai ini aman untuk ada di kode client-side (bukan rahasia), tapi tetap ada dua hal yang **wajib diaktifkan dari Firebase Console** agar login & penyimpanan data berfungsi:

1. **Aktifkan metode masuk Email/Password**
   Firebase Console → **Authentication** → tab **Sign-in method** → aktifkan provider **Email/Password**.

2. **Buat database Firestore**
   Firebase Console → **Firestore Database** → **Create database** (pilih mode production atau test, lokasi bebas).

3. **Pasang aturan keamanan Firestore**, supaya tiap pengguna hanya bisa membaca/menulis pengingat miliknya sendiri. Di tab **Rules**, ganti dengan:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /users/{userId}/reminders/{reminderId} {
         allow read, write: if request.auth != null && request.auth.uid == userId;
       }
       match /users/{userId}/settings/{docId} {
         allow read, write: if request.auth != null && request.auth.uid == userId;
       }
     }
   }
   ```

   Tanpa aturan ini, database akan memakai aturan default (biasanya menolak semua akses, atau — kalau masih mode test — mengizinkan siapa saja membaca/menulis, yang tidak aman untuk produksi).

   > **Sudah pernah pasang aturan versi lama (hanya `reminders`)?** Tambahkan blok `match /users/{userId}/settings/{docId}` di atas ke aturan yang sudah ada, lalu simpan lagi — kalau tidak, penyimpanan kategori kustom akan gagal dengan error izin ditolak.

4. Kalau nanti membuat proyek Firebase baru atau ganti proyek, tinggal ganti nilai `firebaseConfig` di `index.html` (dekat bagian atas `<script type="module">`) dengan konfigurasi dari **Project settings → Your apps** di Firebase Console.

## Keterbatasan yang perlu diketahui

- **Notifikasi hanya berfungsi selagi aplikasi terbuka** (tab browser aktif atau PWA berjalan di latar depan/latar belakang tab). Karena tidak ada server pengirim push sendiri, EssentiaPath tidak bisa mengirim *push notification* saat aplikasi benar-benar tertutup — itu perlu layanan tambahan (misalnya Firebase Cloud Messaging + service worker khusus) yang bisa ditambahkan di revisi berikutnya jika dibutuhkan.
- Font (`Fraunces`, `Public Sans`, `IBM Plex Mono`) dimuat dari Google Fonts saat online; jika ingin offline 100% sejak kunjungan pertama, font bisa di-*self-host* dan ditambahkan ke daftar cache di `service-worker.js`.
- Firebase SDK dimuat dari CDN (`gstatic.com`) melalui `import` modul di `index.html` — butuh koneksi internet saat pertama kali membuka aplikasi (setelah itu, browser biasanya sudah meng-cache modulnya sendiri).

## Palet & tipografi

- Warna: ink `#1E2B22`, moss `#3F6659`, amber `#C97A3D`, paper `#F6F4EA` — terinspirasi dari tema "jalur/peta" (path) sesuai nama aplikasinya.
- Tipografi: **Fraunces** (judul/display), **Public Sans** (isi/UI), **IBM Plex Mono** (label waktu).
