# PROJECT.md — PageSpeed Monitor

Aplikasi web berbasis Laravel untuk menguji performa banyak website sekaligus menggunakan Google PageSpeed Insights API. Website disimpan sekali, lalu bisa dites ulang kapan saja hanya dengan satu klik. Setiap hasil test tersimpan sebagai riwayat dan bisa diekspor ke PDF atau Excel.

> Dokumen terkait:
> - [ERD.md](ERD.md): rancangan database
> - [API.md](API.md): detail Google PageSpeed Insights API

---

## 1. Tujuan

- Mengelola daftar website yang ingin dipantau performanya.
- Menjalankan test PageSpeed untuk satu atau banyak website sekaligus tanpa mengetik URL ulang.
- Menyimpan setiap hasil test sebagai riwayat yang terhubung ke website-nya.
- Menampilkan tren skor dari waktu ke waktu per website.
- Mengekspor hasil ke PDF dan Excel.
- Mencatat semua aktivitas pengguna di log audit.

---

## 2. Fitur

| # | Fitur | Keterangan |
|---|---|---|
| F1 | **Autentikasi** | Login, logout, lupa password, profil. Pendaftaran publik dimatikan; user dibuat oleh admin. |
| F2 | **Manajemen user** | Admin dapat menambah, mengubah, menonaktifkan user dan mengatur role. |
| F3 | **Daftar website** | CRUD website (nama, URL, catatan). URL divalidasi dan unik per user. |
| F4 | **Test satu klik** | Tombol "Test" di setiap baris website, memakai URL yang sudah tersimpan. |
| F5 | **Multi test** | Pilih beberapa website (checkbox), pilih strategi (mobile/desktop/keduanya), lalu jalankan sekaligus. |
| F6 | **Progres real-time** | Status batch (antre, berjalan, selesai, gagal) diperbarui otomatis dengan polling Livewire. |
| F7 | **Riwayat test** | Semua hasil tersimpan, bisa difilter per website, tanggal, strategi, dan status. |
| F8 | **Detail hasil** | Skor 4 kategori, Core Web Vitals (lab dan field), screenshot, serta daftar rekomendasi perbaikan. |
| F9 | **Tren per website** | Grafik skor dan metrik dari riwayat test website tersebut. |
| F10 | **Export PDF** | Laporan detail satu hasil test, atau ringkasan satu batch. |
| F11 | **Export Excel** | Riwayat test (sesuai filter) dan hasil satu batch, termasuk sheet rekomendasi. |
| F12 | **Log audit** | Mencatat login, logout, CRUD website, menjalankan test, export, dan perubahan user. |

---

## 3. Tech Stack

### 3.1 Inti

| Lapisan | Teknologi | Versi |
|---|---|---|
| Bahasa | PHP | 8.3+ |
| Framework | Laravel | 13.x |
| UI reaktif | Livewire | 4.x |
| Interaksi ringan | Alpine.js | bawaan Livewire 4 |
| Styling | Tailwind CSS | 4.x |
| Build tool | Vite | bawaan Laravel |
| Database | MySQL | 8.0+ |
| Queue | Laravel Queue (driver `database`) | bawaan |
| Cache & session | `database` (opsional `redis`) | bawaan |

### 3.2 Package

| Kebutuhan | Package | Catatan |
|---|---|---|
| Autentikasi | Laravel Livewire Starter Kit | Login, reset password, profil sudah tersedia |
| Export Excel | `maatwebsite/excel` ^3.1.68 | Versi 3.1.68 ke atas mendukung Laravel 13 |
| Export PDF | `barryvdh/laravel-dompdf` ^3.1.2 | PDF dari Blade view |
| Grafik | Chart.js (via npm) | Tren skor di halaman website |
| HTTP client | `Illuminate\Support\Facades\Http` | Bawaan, untuk memanggil PSI API |
| Log audit | Tabel `activity_logs` buatan sendiri | Sederhana, tanpa dependensi tambahan |

> Alternatif log audit: `spatie/laravel-activitylog` v5, tetapi versi ini mensyaratkan PHP 8.4+. Tabel sendiri dipilih agar tetap ringan dan cocok dengan PHP 8.3.

### 3.3 Tooling pengembangan

| Tool | Fungsi |
|---|---|
| Laravel Pint | Format kode |
| Pest | Testing |
| Laravel Debugbar (dev saja) | Debug query |

---

## 4. Role & Hak Akses

| Aksi | Admin | User |
|---|:---:|:---:|
| Login / kelola profil sendiri | ✅ | ✅ |
| CRUD website milik sendiri | ✅ | ✅ |
| Lihat website semua user | ✅ | ❌ |
| Jalankan test | ✅ | ✅ (website sendiri) |
| Lihat riwayat | ✅ semua | ✅ milik sendiri |
| Export PDF / Excel | ✅ | ✅ (data sendiri) |
| Kelola user | ✅ | ❌ |
| Lihat log audit | ✅ semua | ✅ aktivitas sendiri |

Implementasi memakai kolom `role` di tabel `users` dan Laravel Policy (`WebsitePolicy`, `TestResultPolicy`).

---

## 5. Alur Utama

### 5.1 Menambah website

```
User → Halaman Website → Tambah → isi Nama + URL → Simpan
     → validasi URL & unik → insert websites → catat activity_logs (website.created)
```

### 5.2 Test satu klik / multi test

```
User → centang 1..n website → pilih strategi → klik "Jalankan Test"
  → buat test_batches (status: queued)
  → buat test_results per website × strategi (status: pending)
  → dispatch Job ke queue (Bus::batch)
  → catat activity_logs (test.started)
  → halaman batch polling status setiap 3 detik

Queue worker → RunPageSpeedTest job
  → panggil PSI API (lihat API.md)
  → simpan ringkasan ke test_results, rekomendasi ke test_audit_items
  → simpan JSON mentah + screenshot ke storage
  → update websites.last_tested_at & skor terakhir
  → update progres test_batches
```

### 5.3 Export

```
Halaman detail hasil  → "Export PDF"   → dompdf render Blade → download
Halaman batch         → "Export PDF"   → ringkasan semua hasil batch
Halaman batch/riwayat → "Export Excel" → maatwebsite/excel → download .xlsx
Setiap export         → catat activity_logs (export.pdf / export.excel)
```

---

## 6. Halaman & Route

| Route | Nama | Komponen Livewire | Akses |
|---|---|---|---|
| `/login` | `login` | starter kit | guest |
| `/dashboard` | `dashboard` | `Dashboard` | auth |
| `/websites` | `websites.index` | `Websites\Index` | auth |
| `/websites/create` | `websites.create` | `Websites\Form` | auth |
| `/websites/{website}/edit` | `websites.edit` | `Websites\Form` | auth + policy |
| `/websites/{website}` | `websites.show` | `Websites\Show` (riwayat + tren) | auth + policy |
| `/batches/{batch}` | `batches.show` | `Batches\Show` (progres) | auth + policy |
| `/history` | `history.index` | `History\Index` | auth |
| `/results/{result}` | `results.show` | `Results\Show` | auth + policy |
| `/results/{result}/pdf` | `results.pdf` | controller | auth + policy |
| `/batches/{batch}/pdf` | `batches.pdf` | controller | auth + policy |
| `/batches/{batch}/excel` | `batches.excel` | controller | auth + policy |
| `/history/excel` | `history.excel` | controller (ikut filter) | auth |
| `/activity-logs` | `logs.index` | `ActivityLogs\Index` | auth |
| `/users` | `users.index` | `Users\Index` | admin |

### Isi dashboard

- Jumlah website, test hari ini, dan sisa kuota harian (dihitung dari `test_results` hari ini).
- Batch yang sedang berjalan.
- 5 website dengan skor performa terendah.
- 10 hasil test terakhir.

---

## 7. Struktur Folder

```
app/
├── Enums/
│   ├── Strategy.php             # mobile, desktop
│   ├── TestStatus.php           # pending, running, done, failed
│   └── UserRole.php             # admin, user
├── Exports/
│   ├── BatchExport.php          # multi-sheet: Ringkasan + Rekomendasi
│   └── HistoryExport.php
├── Http/Controllers/
│   └── ExportController.php
├── Jobs/
│   └── RunPageSpeedTest.php
├── Livewire/
│   ├── Dashboard.php
│   ├── Websites/{Index,Form,Show}.php
│   ├── Batches/Show.php
│   ├── History/Index.php
│   ├── Results/Show.php
│   ├── ActivityLogs/Index.php
│   └── Users/Index.php
├── Models/
│   ├── User.php
│   ├── Website.php
│   ├── TestBatch.php
│   ├── TestResult.php
│   ├── TestAuditItem.php
│   └── ActivityLog.php
├── Policies/
│   ├── WebsitePolicy.php
│   └── TestResultPolicy.php
├── Services/
│   ├── PageSpeedService.php     # panggil API
│   ├── PageSpeedParser.php      # ekstrak field dari JSON
│   ├── TestRunner.php           # buat batch + dispatch job
│   └── ActivityLogger.php       # helper tulis log audit
└── Listeners/
    └── LogAuthEvents.php        # Login, Logout, Failed

resources/views/
├── livewire/...
└── pdf/
    ├── result.blade.php
    └── batch.blade.php

storage/app/private/pagespeed/{batch_id}/
├── {result_id}.json.gz          # respons mentah
└── {result_id}.jpg              # screenshot
```

---

## 8. Log Audit

Semua pencatatan lewat `ActivityLogger::log($action, $subject, $properties)`.

| Action | Pemicu | Subject |
|---|---|---|
| `auth.login` | Event `Login` | User |
| `auth.logout` | Event `Logout` | User |
| `auth.failed` | Event `Failed` | – (email di properties) |
| `website.created` | Simpan website baru | Website |
| `website.updated` | Ubah website (properties: old/new) | Website |
| `website.deleted` | Hapus website | Website |
| `test.started` | Jalankan test/multi test | TestBatch |
| `test.finished` | Batch selesai | TestBatch |
| `export.pdf` | Download PDF | TestResult / TestBatch |
| `export.excel` | Download Excel | TestBatch / null (riwayat) |
| `user.created` / `user.updated` / `user.deactivated` | Admin kelola user | User |

---

## 9. Queue & Scheduler

| Item | Konfigurasi |
|---|---|
| Driver | `QUEUE_CONNECTION=database` |
| Nama queue | `pagespeed` |
| Rate limit | `RateLimiter::for('pagespeed')` = 200/menit |
| Timeout job | 180 detik |
| Retry | 3× dengan backoff 30, 90, 180 detik |
| Worker | `php artisan queue:work --queue=pagespeed,default --timeout=200` |

Scheduler (`routes/console.php`):

| Jadwal | Tugas |
|---|---|
| Harian 02:00 | Hapus file JSON/screenshot berumur > 90 hari (`pagespeed:prune`) |
| Harian 02:30 | Hapus `activity_logs` berumur > 365 hari |
| Per menit | Tandai batch macet (running > 30 menit) sebagai gagal |

---

## 10. Konfigurasi `.env`

```env
APP_NAME="PageSpeed Monitor"
APP_TIMEZONE=Asia/Jakarta
APP_LOCALE=id

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=pagespeed_monitor
DB_USERNAME=root
DB_PASSWORD=

QUEUE_CONNECTION=database
SESSION_DRIVER=database
CACHE_STORE=database

PSI_API_KEY=
PSI_LOCALE=id
PSI_DAILY_QUOTA=25000
PSI_RAW_RETENTION_DAYS=90
```

---

## 11. Instalasi

```bash
# 1. Buat project dengan Livewire starter kit
laravel new pagespeed-monitor        # pilih: Livewire starter kit, MySQL, Pest
cd pagespeed-monitor

# 2. Package tambahan
composer require maatwebsite/excel:^3.1.68 barryvdh/laravel-dompdf:^3.1.2
npm install chart.js

# 3. Database & queue
php artisan queue:table              # jika belum ada
php artisan queue:batches-table
php artisan migrate --seed           # seeder membuat user admin awal

# 4. Jalankan
composer run dev                     # server + vite + queue (sesuaikan nama queue)
```

---

## 12. Deployment

- PHP 8.3+, ekstensi: `pdo_mysql`, `mbstring`, `gd`, `zip`, `xml`, `zlib` (untuk `gzencode`).
- Worker queue dijalankan oleh **Supervisor** (minimal 2 proses untuk queue `pagespeed`).
- Cron: `* * * * * php /path/artisan schedule:run`.
- `php artisan storage:link` tidak diperlukan karena file disimpan di disk privat dan diakses lewat controller.
- `php artisan optimize` setelah deploy.

---

## 13. Rencana Pengerjaan

| Tahap | Lingkup | Hasil |
|---|---|---|
| 1 | Setup project, auth, role, seeder admin | Bisa login |
| 2 | Migration & model sesuai ERD | Database siap |
| 3 | CRUD website + log audit | Daftar website tersimpan |
| 4 | Service + parser + job PSI, multi test, halaman progres batch | Test berjalan & tersimpan |
| 5 | Riwayat, detail hasil, tren per website | Riwayat lengkap |
| 6 | Export PDF & Excel | File unduhan |
| 7 | Dashboard, manajemen user, halaman log audit | Fitur lengkap |
| 8 | Testing (Pest), scheduler pembersihan, deploy | Siap produksi |

---

## 14. Batasan yang Perlu Diingat

- Kuota gratis PSI: 25.000 request/hari per project Google Cloud. 1 website × 2 strategi = 2 request.
- Satu test memakan 15–40 detik, sehingga multi test 50 website (×2 strategi) bisa berjalan beberapa menit.
- Skor bisa bervariasi ±5–10 poin antar test; tren lebih bermakna daripada satu angka.
- Field data (CrUX) tidak tersedia untuk website bertrafik rendah.
- Website yang membutuhkan login atau berada di jaringan lokal tidak bisa dites, karena PSI mengakses dari server Google.
