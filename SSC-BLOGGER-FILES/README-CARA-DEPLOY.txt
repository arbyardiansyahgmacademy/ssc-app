╔══════════════════════════════════════════════════════════════════════════════════╗
║         STUDENT SERVICE CENTER (SSC) — PANDUAN DEPLOYMENT BLOGSPOT              ║
║         Versi: 2.0 | Backend: Google Apps Script | Frontend: Blogger            ║
╚══════════════════════════════════════════════════════════════════════════════════╝

STRUKTUR FILE:
═══════════════════════════════════════════════════════════════
  SSC-BLOGGER-FILES/
  ├── 01_BLOGGER-THEME-EDIT-HTML.xml      ← Edit HTML Theme Blogger
  ├── README-CARA-DEPLOY.txt              ← File panduan ini
  └── pages/
      ├── beranda.html          → Blogger Page: /p/beranda
      ├── layanan.html          → Blogger Page: /p/layanan
      ├── cara-kerja.html       → Blogger Page: /p/cara-kerja
      ├── cek-pengajuan.html    → Blogger Page: /p/cek-pengajuan
      ├── blog.html             → Blogger Page: /p/blog
      ├── kontak.html           → Blogger Page: /p/kontak
      ├── login.html            → Blogger Page: /p/login
      ├── dashboard.html        → Blogger Page: /p/dashboard
      └── admin.html            → Blogger Page: /p/admin

═══════════════════════════════════════════════════════════════
CARA KERJA SISTEM:
═══════════════════════════════════════════════════════════════

Blogger THEME → Menyediakan:
  - Semua CSS (variabel, komponen, layout)
  - Navbar publik (shared)
  - Footer (shared)
  - Floating WA + Back to Top
  - Login Modal
  - JS Engine (SSC Object, auth, GAS, DUMMY_DB)

Setiap Blogger PAGE → Hanya berisi:
  - window.SSC_PAGE = 'nama-halaman'  (identifikasi halaman)
  - Konten HTML unik halaman itu saja
  - window.SSC_pageInit = function(){}  (hook inisialisasi)

Urutan eksekusi:
  1. Theme load → SSC Engine siap
  2. Blogger render Page HTML ke dalam #main
  3. DOMContentLoaded → SSC.init() baca window.SSC_PAGE
  4. Panggil window.SSC_pageInit() jika ada

═══════════════════════════════════════════════════════════════
LANGKAH DEPLOYMENT:
═══════════════════════════════════════════════════════════════

STEP 1: SETUP BLOGGER THEME
─────────────────────────────
1. Buka: Blogger Dashboard (blogger.com)
2. Pilih blog Anda
3. Klik menu "Theme" di sidebar kiri
4. Klik tombol "Customize" → lalu "Edit HTML"
5. HAPUS SEMUA konten yang ada
6. Copy-paste isi file "01_BLOGGER-THEME-EDIT-HTML.xml"
7. Klik "Save theme"
   ⚠ PENTING: Ubah GAS_URL di dalam file itu dengan URL
              Google Apps Script Anda sendiri!

STEP 2: BUAT 9 BLOGGER PAGES
─────────────────────────────
Untuk setiap file di folder pages/:

1. Buka: Blogger Dashboard → Pages → New Page
2. Klik "HTML" (bukan "Compose")  ← WAJIB!
3. HAPUS semua konten yang ada di editor
4. Copy-paste isi file HTML yang sesuai
5. Di "Page title" tulis: Beranda / Layanan / dst.
6. Di "Page URL/permalink", set manual:
   ┌─────────────────────────────────────────┐
   │  beranda.html      → /p/beranda         │
   │  layanan.html      → /p/layanan         │
   │  cara-kerja.html   → /p/cara-kerja      │
   │  cek-pengajuan.html→ /p/cek-pengajuan   │
   │  blog.html         → /p/blog            │
   │  kontak.html       → /p/kontak          │
   │  login.html        → /p/login           │
   │  dashboard.html    → /p/dashboard       │
   │  admin.html        → /p/admin           │
   └─────────────────────────────────────────┘
7. Klik "Publish"
8. Ulangi untuk semua 9 halaman

STEP 3: KONFIGURASI GOOGLE APPS SCRIPT
─────────────────────────────────────────
1. Buka script.google.com
2. Buat project baru
3. Paste kode GAS backend Anda
4. Deploy sebagai "Web App":
   - Execute as: Me
   - Who has access: Anyone
5. Salin "Web app URL" yang didapat
6. Kembali ke Theme → Edit HTML
7. Cari: GAS_URL: "https://script.google.com/..."
8. Ganti dengan URL Anda
9. Save theme

═══════════════════════════════════════════════════════════════
KONFIGURASI BLOGGER YANG DISARANKAN:
═══════════════════════════════════════════════════════════════

Settings → Basic:
  - Privacy: Public

Settings → Posts:
  - Show at most: 1 post

Settings → Comments: OFF (tidak perlu komentar)

Pages: AKTIFKAN semua 9 halaman

═══════════════════════════════════════════════════════════════
AKUN DEMO (untuk testing):
═══════════════════════════════════════════════════════════════

Admin TU:  username: admin     | password: admin2026
Siswa:     username: peserta   | password: edudigital

Masuk ke: /p/login

═══════════════════════════════════════════════════════════════
TROUBLESHOOTING:
═══════════════════════════════════════════════════════════════

❌ CSS tidak tampil:
   → Pastikan tidak ada konflik tema lama. Hapus semua isi Edit HTML sebelum paste.

❌ Halaman tidak ditemukan:
   → Cek permalink/URL halaman sudah sesuai (misal /p/beranda, bukan /p/beranda-html)

❌ Login tidak bisa redirect:
   → Pastikan semua 9 halaman sudah dipublish dan URL-nya benar di SSC_CONFIG.PAGES

❌ GAS tidak merespons:
   → Sistem otomatis pakai DUMMY_DB (mode simulasi) jika GAS bermasalah.
      Semua fitur tetap berfungsi untuk demo.

❌ Navbar masih tampil di halaman dashboard:
   → Pastikan window.SSC_PAGE sudah didefinisikan di baris pertama setiap halaman.

═══════════════════════════════════════════════════════════════
