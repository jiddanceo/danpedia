# CHANGELOG — Dan Pedia Web Platform
> Platform Top Up Game & PPOB — dan-pedia.com  
> Dokumentasi seluruh perubahan, perbaikan, dan penambahan fitur

---

## Informasi Stack

| Komponen | Detail |
|---|---|
| **VPS** | 151.240.0.46 (Ubuntu 24.04) |
| **Backend** | Laravel 12.45.0 — `/var/www/Laravel_Backend` |
| **Frontend** | Next.js 14 — `/var/www/NextJS_FrontEnd` |
| **Database** | MySQL 8.0.45 — DB: `danpedia` |
| **PHP** | 8.3.30 |
| **Node.js** | v20.20.0 |
| **PM2** | v6.0.14 (process name: `danpedia`) |
| **Supervisor** | 4.2.5 |
| **Domain Backend** | https://api.dan-pedia.com |
| **Domain Frontend** | https://dan-pedia.com |

---

## Cara Deploy Setelah Perubahan Kode

### Backend (Laravel)
```bash
cd /var/www/Laravel_Backend
php artisan config:clear
php artisan cache:clear
php artisan route:clear
php artisan view:clear
supervisorctl restart laravel-worker:*   # wajib setelah deploy kode baru
```

### Frontend (Next.js)
```bash
cd /var/www/NextJS_FrontEnd
npm run build
pm2 restart danpedia
```

### Upload File dari Windows ke VPS
```powershell
# Upload file
pscp -pw "PASSWORD" "C:\path\to\file" root@151.240.0.46:/var/www/path/

# Jalankan perintah remote
echo yes | plink -pw "PASSWORD" root@151.240.0.46 "perintah di sini"
```

---

## Monitoring & Perintah Berguna

```bash
# Cek status PM2 (Next.js)
pm2 status

# Cek log PM2
pm2 logs danpedia --lines 50

# Cek status Supervisor (Queue Worker)
supervisorctl status

# Cek failed jobs
php /var/www/Laravel_Backend/artisan queue:failed

# Retry semua failed jobs
php /var/www/Laravel_Backend/artisan queue:retry all

# Cek log worker
tail -f /var/www/Laravel_Backend/storage/logs/worker.log

# Cek log Laravel
tail -f /var/www/Laravel_Backend/storage/logs/laravel.log
```

---

---

# CHANGELOG ENTRIES

---

## [Session 1–15] — Setup Awal & Fitur Dasar
> Periode awal sebelum sesi dokumentasi ini

### Summary Pekerjaan Awal
- Setup VPS (Ubuntu), Nginx, PHP 8.3, MySQL 8.0
- Deploy Laravel Backend + Next.js Frontend
- Setup PM2 untuk menjalankan Next.js
- Konfigurasi domain dan Nginx reverse proxy
- Integrasi Digiflazz untuk sync produk (Pulsa, Data, Games, E-Money, PLN, Voucher)
- Sistem Order dengan payment gateway (QRIS via SmpQris & Qrispy)
- Filament Admin Panel
- Sistem user (Basic, Gold, Platinum tier)
- Sistem profit berbasis persentase per game

---

## [v1.16.1] — 2026 (Session 16a–16h)

### 16a — Halaman Kontak: Info Kontak
- **File**: `NextJS_FrontEnd/src/app/hubungi-kami/page.tsx`
- **Perubahan**: Update informasi kontak (nomor WhatsApp, email, alamat)

### 16b — Revisi Alamat
- **File**: `NextJS_FrontEnd/src/app/hubungi-kami/page.tsx`
- **Perubahan**: Perbaikan teks alamat kantor

### 16c — Fix Kategori Video/Streaming
- **File**: DB table `categories`
- **Perubahan**: Perbaikan nama/sort kategori Streaming/Vidio

### 16d — Update Harga E-Money
- **DB**: Update `selling_price` produk E-Money sesuai harga terbaru

### 16e — Missing game_configurations
- **DB**: Tambah konfigurasi input field untuk game-game yang belum punya konfigurasi

### 16f — Validasi Nickname
- **File**: `Laravel_Backend/app/Http/Controllers/Api/OrderController.php`
- **Perubahan**: Tambah validasi nickname field pada order form

### 16g — PUBG Mobile: Static Message + Disable Validation
- **File**: `Laravel_Backend/app/Http/Controllers/Api/OrderController.php`
- **Perubahan**:
  - Disable validasi field untuk PUBG Mobile
  - Tampilkan pesan statis: "Masukkan User ID PUBG Mobile Anda"

### 16h — Fix Kategori Pulsa vs Data Terpisah (OrderController)
- **Root Cause**: `OrderController` mengambil SEMUA produk berdasarkan brand saja (contoh: Indosat = 65 produk campur Pulsa + Data)
- **File**: `Laravel_Backend/app/Http/Controllers/Api/OrderController.php`
- **Perubahan**:
  - Tambah filter by `category_id` dari tabel `categories`
  - Import: `use App\Models\Category;`
- **DB**: Buat 6 game baru untuk Data (IDs 58–63):
  - Axis Data, Indosat Data, Telkomsel Data, Xl Data, Tri Data, Smartfren Data
  - Semua dengan `category_id=4` (Data)
- **DB**: Buat 6 `game_configurations` baru (IDs 44–49) untuk game Data
- **File**: `Laravel_Backend/app/Http/Controllers/Api/GameController.php`
  - Sederhanakan: `$game->product_category_ids = $game->category_id ? [$game->category_id] : []`

---

## [v1.16.2] — 2026 (Session 16i–16k)

### 16i — E-Money: Flat Profit Protection
- **Root Cause**: Saat sync Digiflazz, profit E-Money ikut ter-recalculate (harusnya flat)
- **File**: `Laravel_Backend/app/Http/Controllers/ProductDigiflazzController.php`
- **Perubahan**:
  - Blok UPDATE (line ~683): cek `$categoryGame === 'E-Money'` → gunakan flat profit 900/700/500
  - Blok INSERT (line ~711): same protection
- **DB**: Update 163 produk E-Money → `profit=900, profit_gold=700, profit_platinum=500`
- **Flat profit E-Money**: Basic=900, Gold=700, Platinum=500

### 16j — Fix Footer Logo Gepeng
- **Root Cause**: Class `h-10 w-32` memaksa lebar fixed sehingga logo terdistorsi
- **File**: `NextJS_FrontEnd/src/components/panel/footer.tsx`
- **Perubahan**: `className="h-10 w-32"` → `className="h-10 w-auto object-contain"`
- **Deploy**: `npm run build` + `pm2 restart danpedia`

---

## [v1.16.3] — 2026-03-04 (Session 16l)

### Fix "Three" → "hree" pada Produk Tri (TRI brand)
- **Root Cause**: Fungsi `stripLeadingGameName()` membuat abbreviasi `$abbr("TRI") = "T"` (1 karakter). Regex pattern menjadi `^(?:TRI|Tri|tri|T)(?:separator)?` yang cocok dengan awal kata "**T**hree" → strip → "hree"
- **File**: `Laravel_Backend/app/Http/Controllers/ProductDigiflazzController.php`
- **Fix** (line ~337):
  ```php
  $candidates = array_values(array_unique(array_filter($candidates)));
  // Filter out single/double char abbreviations to avoid false positives
  $candidates = array_values(array_filter($candidates, fn($c) => mb_strlen($c) >= 3));
  if (empty($candidates)) {
      return $original;
  }
  ```
- **DB**: `UPDATE products SET title = CONCAT('T', title) WHERE brand='TRI' AND title LIKE 'hree%'` → 13 baris diperbaiki

### Pulsa: Flat Profit + Protection
- **File**: `Laravel_Backend/app/Http/Controllers/ProductDigiflazzController.php`
- **Perubahan**:
  - Blok UPDATE: extend cek `|| $categoryGame === 'Pulsa'`
  - `$flatProfit = $categoryGame === 'E-Money' ? 900 : 1000` (Pulsa = 1000/700/500)
  - Blok INSERT: same extension
- **DB**: Update 119 produk Pulsa → `profit=1000, profit_gold=700, profit_platinum=500`
- **Flat profit Pulsa**: Basic=1000, Gold=700, Platinum=500

---

## [v1.17.0] — 2026-03-05

### Admin Panel: Flat Profit Support
- **Root Cause**: Form "Buat Profit" hanya support persentase (%), tidak ada opsi Flat Rupiah
- **Migration**: `2026_03_05_000001_add_flat_profit_to_profits_table.php`
  - Kolom baru: `profit_type` (enum: percentage/flat, default: percentage)
  - Kolom baru: `profit_flat`, `profit_gold_flat`, `profit_platinum_flat` (decimal, nullable)
- **File**: `Laravel_Backend/app/Models/Profit.php`
  - Tambah kolom baru ke `$fillable`
- **File**: `Laravel_Backend/app/Filament/Resources/ProfitResource.php`
  - Dropdown "Tipe Profit": Persentase (%) / Flat (Rupiah)
  - Field input berubah sesuai tipe yang dipilih (reactive)
  - Tabel: tampil badge tipe + kolom Rp/% yang relevan, yang tidak dipakai tampil `-`
- **File**: `Laravel_Backend/app/Http/Controllers/ProductDigiflazzController.php`
  - Kalkulasi profit sekarang cek `profit_type`:
    - Jika `flat` → pakai `profit_flat/profit_gold_flat/profit_platinum_flat` dari DB
    - Jika `percentage` → hitung dari persentase seperti sebelumnya
  - Guard E-Money/Pulsa hardcode hanya berlaku jika **tidak ada** profit record flat dari admin

#### Cara Penggunaan Flat Profit dari Admin:
1. Buka Admin → Manajemen Game → **Profit**
2. Klik **Buat profit**
3. Pilih Game
4. Pilih **Tipe Profit**: `Flat (Rupiah)`
5. Isi nominal: Profit Basic (Rp), Profit Gold (Rp), Profit Platinum (Rp)
6. Klik Buat

---

### Daftar Harga: Filter Produk per Kategori
- **Root Cause**: `PriceListController` mengembalikan semua produk berdasarkan brand saja → "Telkomsel Data" berisi 154 produk (Pulsa + Data + SMS campur). Frontend filter `category_id` tidak berfungsi karena mapping PHP Collection inkonsisten tipe data
- **File**: `Laravel_Backend/app/Http/Controllers/Api/PriceListController.php`
  - **Rewrite total**: Filter produk berdasarkan `brand` + `categoryName` dari `product_categories`
  - Setiap game hanya mengembalikan produk sesuai kategorinya
  - `category_id` per produk = `game->category_id` (tidak perlu mapping kompleks)
- **Hasil**:
  | Game | Jumlah Produk Setelah Fix |
  |---|---|
  | Telkomsel Data | 108 produk Data ✓ |
  | Telkomsel (Pulsa) | 29 produk Pulsa ✓ |
  | Xl Data | 75 produk Data ✓ |
  | Indosat Data | 63 produk Data ✓ |
  | Tri Data | 46 produk Data ✓ |
  | Axis Data | 28 produk Data ✓ |

### Daftar Harga: Tab Kategori Di Tengah
- **Root Cause**: CSS `flex` tanpa `justify-center` → tab menempel ke kiri
- **File**: `NextJS_FrontEnd/src/app/price-list/page.tsx`
- **Perubahan**: `"flex gap-2 flex-wrap"` → `"flex gap-2 flex-wrap justify-center"`
- **Deploy**: `npm run build` + `pm2 restart danpedia`

---

## [v1.17.1] — 2026-03-05

### Admin Produk: Brand Searchable
- **Root Cause**: Kolom `brand` tidak punya `->searchable()`. Search "TELKOMSEL" kosong karena judul produk berisi "Pulsa 5.000", bukan "TELKOMSEL"
- **File**: `Laravel_Backend/app/Filament/Resources/ProductResource.php`
- **Perubahan**: Tambah `->searchable()` dan `->sortable()` pada kolom `brand`

### Admin Produk: Tab Kategori
- **File**: `Laravel_Backend/app/Filament/Resources/ProductResource/Pages/ListProducts.php`
- **Perubahan**: Tambah method `getTabs()` yang membuat tab dinamis berdasarkan tabel `categories`
- **Tabs yang tersedia**: Semua | Pulsa | Data | Games | E-Money | PLN | Voucher | Streaming
- Setiap tab menampilkan **badge** jumlah produk
- Klik tab → filter otomatis berdasarkan `product_categories.game`

### Admin Produk: Filter Panel
- **File**: `Laravel_Backend/app/Filament/Resources/ProductResource.php`
- **Filter baru** (klik ikon filter di tabel):
  - **Kategori**: dropdown pilih kategori (Pulsa/Data/Games/dst)
  - **Brand**: dropdown searchable pilih brand (TELKOMSEL/XL/INDOSAT/dst)
  - **Status Aktif**: toggle Aktif / Nonaktif / Semua
- Kolom label diubah: "Kategori" → "Sub-Kategori" (lebih akurat)

### Frontend Daftar Harga: Perbaikan Filter Kategori per Produk
- **File**: `NextJS_FrontEnd/src/app/price-list/page.tsx`
- Saat tab dipilih → filter produk berdasarkan `category_id` yang dikirim dari API
- **Fungsi baru** `categoryFilteredProducts`: filter produk berdasarkan `category_id` dari tab aktif

---

## [v1.18.0] — 2026-03-05

### Setup Queue + Supervisor
- **Kondisi Sebelum**: `QUEUE_CONNECTION=sync` → semua job (email, WhatsApp) dijalankan langsung, memblokir response
- **Kondisi Sesudah**: `QUEUE_CONNECTION=database` → job dijalankan di background

#### Langkah Setup:
1. **Install Supervisor**:
   ```bash
   apt-get install -y supervisor
   ```

2. **Ubah Queue Driver** di `.env`:
   ```
   QUEUE_CONNECTION=database
   ```

3. **Tabel Queue** di DB (sudah ada dari migration lama):
   - `jobs` — antrian job
   - `failed_jobs` — job yang gagal
   - `job_batches` — batch job

4. **Buat config Supervisor**:
   ```ini
   # /etc/supervisor/conf.d/laravel-worker.conf
   [program:laravel-worker]
   process_name=%(program_name)s_%(process_num)02d
   command=php /var/www/Laravel_Backend/artisan queue:work database --sleep=3 --tries=3 --max-time=3600
   autostart=true
   autorestart=true
   stopasgroup=true
   killasgroup=true
   user=www-data
   numprocs=2
   redirect_stderr=true
   stdout_logfile=/var/www/Laravel_Backend/storage/logs/worker.log
   stopwaitsecs=3600
   ```

5. **Start Supervisor**:
   ```bash
   systemctl enable supervisor
   systemctl start supervisor
   supervisorctl reread
   supervisorctl update
   supervisorctl status
   ```

#### Jobs yang Berjalan Async (Background):
| Job | Deskripsi |
|---|---|
| `SendOrderEmail` | Kirim email konfirmasi order |
| `SendWhatsAppNotificationJob` | Kirim notifikasi WhatsApp |
| `DigiflazzSyncPendingJob` | Sync status order pending ke Digiflazz |

#### Config Worker:
- **2 worker** berjalan paralel (numprocs=2) sebagai user `www-data`
- **3 kali retry** jika job gagal → masuk `failed_jobs`
- **Auto-restart** setiap 1 jam (`--max-time=3600`) untuk fresh memory
- **Auto-start** saat server reboot

---

## Template Entry Changelog Baru

Salin template ini setiap ada perubahan baru:

```markdown
## [v1.X.Y] — YYYY-MM-DD

### Nama Fitur / Fix
- **Root Cause** (jika bugfix): Penjelasan penyebab masalah
- **File**: `path/ke/file.php` atau `path/ke/file.tsx`
- **Perubahan**:
  - Deskripsi perubahan 1
  - Deskripsi perubahan 2
- **DB** (jika ada): Query atau migration yang dijalankan
- **Deploy**:
  - [ ] `php artisan config:clear && php artisan cache:clear`
  - [ ] `npm run build && pm2 restart danpedia`
  - [ ] `supervisorctl restart laravel-worker:*`
```

---

## Referensi File Penting

### Backend (Laravel) — `/var/www/Laravel_Backend`
| File | Fungsi |
|---|---|
| `app/Http/Controllers/Api/OrderController.php` | Proses order, validasi input, kalkulasi harga |
| `app/Http/Controllers/Api/GameController.php` | Data game + konfigurasi untuk order page |
| `app/Http/Controllers/Api/PriceListController.php` | Data untuk halaman Daftar Harga publik |
| `app/Http/Controllers/ProductDigiflazzController.php` | Sync produk dari Digiflazz, kalkulasi profit |
| `app/Filament/Resources/ProductResource.php` | Admin: halaman kelola produk |
| `app/Filament/Resources/ProductResource/Pages/ListProducts.php` | Admin: tab kategori produk |
| `app/Filament/Resources/ProfitResource.php` | Admin: kelola profit per game |
| `app/Models/Product.php` | Model produk |
| `app/Models/Profit.php` | Model profit |
| `app/Models/Game.php` | Model game |
| `app/Models/Category.php` | Model kategori utama |
| `app/Models/ProductCategory.php` | Model sub-kategori produk |
| `app/Services/DigiflazzService.php` | Integrasi API Digiflazz |
| `app/Jobs/SendOrderEmail.php` | Job: kirim email order |
| `app/Jobs/SendWhatsAppNotificationJob.php` | Job: kirim notif WhatsApp |
| `app/Jobs/DigiflazzSyncPendingJob.php` | Job: sync status pending ke Digiflazz |
| `config/queue.php` | Konfigurasi queue driver |
| `.env` | Environment variables (QUEUE_CONNECTION, DB, dll) |

### Frontend (Next.js) — `/var/www/NextJS_FrontEnd`
| File | Fungsi |
|---|---|
| `src/app/price-list/page.tsx` | Halaman Daftar Harga publik |
| `src/app/order/[slug]/page.tsx` | Halaman Order |
| `src/components/panel/footer.tsx` | Footer (logo, links) |
| `src/app/hubungi-kami/page.tsx` | Halaman Kontak |
| `src/app/api/price-list/route.ts` | API Route: proxy ke Laravel price-list |

### Infrastruktur
| File | Fungsi |
|---|---|
| `/etc/supervisor/conf.d/laravel-worker.conf` | Config Supervisor untuk queue worker |
| `/etc/nginx/sites-available/danpedia` | Nginx config (backend + frontend) |

---

## Struktur Database (Tabel Penting)

| Tabel | Deskripsi |
|---|---|
| `products` | Semua produk (brand, category, profit, selling_price) |
| `product_categories` | Sub-kategori produk (game=Pulsa/Data/Games/dll) |
| `categories` | Kategori utama (Pulsa, Data, Games, E-Money, PLN, Voucher) |
| `games` | Daftar game dengan brand & category_id |
| `game_configurations` | Konfigurasi input field per game (nomor HP, user ID, dll) |
| `profits` | Konfigurasi profit per game (percentage/flat) |
| `orders` | Riwayat order |
| `users` | User (Basic/Gold/Platinum) |
| `jobs` | Queue jobs (background tasks) |
| `failed_jobs` | Queue jobs yang gagal |
| `settings` | Konfigurasi global (default profit, dll) |

---

## [v1.19.0] — 2026-03-05

### Scheduler Fetch Product: 2x Daily (00:00 & 06:00 WIB)
- **File**: `/var/www/Laravel_Backend/routes/console.php` (VPS)
- **Perubahan**: Schedule `product:fetch-product` diubah dari 1x sehari (`23:59`) menjadi 2x sehari
- **Schedule sebelumnya**:
  ```php
  Schedule::command('product:fetch-product')->timezone('Asia/Jakarta')->dailyAt('23:59');
  ```
- **Schedule sesudahnya**:
  ```php
  Schedule::command('product:fetch-product')->timezone('Asia/Jakarta')->dailyAt('00:00');
  Schedule::command('product:fetch-product')->timezone('Asia/Jakarta')->dailyAt('06:00');
  ```
- **Hasil** (verifikasi `php artisan schedule:list`):
  - `0 0 * * *` → 00:00 WIB
  - `0 6 * * *` → 06:00 WIB

---

### Fetch Product: Logging ke Admin Log Viewer
- **Root Cause**: Command `product:fetch-product` hanya output ke terminal (`$this->info/error`), tidak menulis ke Laravel log sehingga tidak bisa dipantau dari admin Log Viewer
- **File**: `/var/www/Laravel_Backend/app/Console/Commands/FetchProductDigiflazz.php` (VPS)
- **Perubahan**:
  - Tambah `use Illuminate\Support\Facades\Log;`
  - Saat berhasil: `Log::info('[FETCH-PRODUCT] Berhasil sync produk dari Digiflazz.')`
  - Saat gagal API: `Log::error('[FETCH-PRODUCT] Gagal: ' . $msg)`
  - Saat exception: `Log::error('[FETCH-PRODUCT] Exception: ' . $e->getMessage())`
- **Cara Pantau**: Buka Admin → Log Viewer → cari keyword `FETCH-PRODUCT`

---

---

## [v1.19.1] — 2026-03-06

### Fix Logo Produk: Prioritas Logo Admin vs Logo Digiflazz
- **Root Cause**: Semua produk yang di-sync Digiflazz memiliki kolom `images` terisi (e.g. `product/logo/pulsa-umum.svg`). Frontend mengecek `product.images || product.logo` sehingga `images` selalu dipakai, logo yang di-upload admin di kolom `logo` tidak pernah tampil
- **File**: `/var/www/NextJS_FrontEnd/src/components/order/ProductSelection.tsx` (2 baris)
- **Perubahan**: `product.images || product.logo` → `product.logo || product.images`
- **Efek**: Logo ber-upload di admin panel (e.g. logo by.U) sekarang tampil di halaman order
- **Deploy**: `npm run build` + `pm2 restart danpedia`

---

### Fix Game by.U Data: Produk Data Tidak Muncul di Menu Data
- **Root Cause**: Game "by.U" hanya ada satu dengan `category_id=3` (Pulsa). Tidak ada game "by.U Data" untuk `category_id=4` (Data). Produk Data by.U (32 produk) ada di DB tapi tidak bisa diakses
- **DB Fix**:
  - Buat game baru: `by.U Data` (`brand='by.U'`, `slug='byu-data'`, `category_id=4`)
  - Buat `game_configuration` baru: Nomor by.U (`type=number`)
- **Hasil**: Halaman Order → menu Data → by.U Data menampilkan 32 produk ✓

---

### Fix WeTV & Viu: Kategori Salah (Pulsa → Streaming)
- **Root Cause**:
  - WeTV (`game.id=65`): `category_id=3` (Pulsa) — seharusnya 11 (Streaming). Tidak ada `game_configuration`
  - Viu (`game.id=64`): `category_id=1` (Voucher) — seharusnya 11 (Streaming). Tidak ada `game_configuration`
- **DB Fix WeTV**:
  - `games.category_id=11` (Streaming)
  - Buat `product_category`: Membership/Streaming (`id=75`)
  - Pindahkan 3 produk WeTV → `category=75`
  - Buat `game_configuration`: Email Akun WeTV (`type=email`)
- **DB Fix Viu**:
  - `games.category_id=11` (Streaming)
  - Buat `product_category`: Voucher/Streaming (`id=76`)
  - Pindahkan 2 produk Viu → `category=76`
  - Buat `game_configuration`: Email Akun Viu (`type=email`)
- **Hasil**: WeTV & Viu sekarang muncul di kategori Streaming dengan input Email ✓

---

### Fix Vidio: Produk Tidak Muncul Saat Streaming Category Aktif
- **Root Cause**: Setelah membuat `product_categories` Streaming (id=75, 76), `OrderController` mulai memfilter produk Streaming. Vidio memiliki produk di `product_category=3` (Umum/Pulsa), sehingga tidak terinclude lagi dalam filter Streaming
- **DB Fix**:
  - Buat `product_category`: Umum/Streaming (`id=77`)
  - Pindahkan 3 produk Vidio → `category=77`
- **Hasil**: Vidio tetap tampil di Streaming ✓

---

### Proteksi Sync: Streaming Brands (WeTV, Viu, Vidio)
- **Root Cause**: Saat sinkronisasi Digiflazz, brand WeTV/Viu/Vidio mendapat `categoryGame='Pulsa'` atau `'Voucher'` dari normalizer. Produk baru hasil sync akan dimasukkan ke `product_categories` Pulsa/Voucher, bukan Streaming
- **File**: `app/Http/Controllers/ProductDigiflazzController.php` (line ~574)
- **Perubahan**: Tambah blok override setelah `$isPB` check:
  ```php
  // Proteksi posisi Streaming: WeTV, Viu, Vidio harus selalu di Streaming
  // agar tidak salah posisi saat sinkronisasi produk dari Digiflazz
  $streamingBrands = ['WeTV', 'Viu', 'Vidio'];
  if (in_array($productBrand, $streamingBrands, true)) {
      $categoryGame = 'Streaming';
  }
  ```
- **Efek**: Produk WeTV/Viu/Vidio (baru maupun existing) selalu masuk `product_categories` dengan `game='Streaming'`
- **Deploy**: `php artisan config:clear && php artisan cache:clear`

---

### State Database Streaming (setelah semua fix):
| ID | product_categories.title | .game | Brand | Jumlah Produk |
|---|---|---|---|---|
| 75 | Membership | Streaming | WeTV | 3 |
| 76 | Voucher | Streaming | Viu | 2 |
| 77 | Umum | Streaming | Vidio | 3 |

### State Games Streaming (setelah semua fix):
| ID | Title | Brand | Slug | category_id |
|---|---|---|---|---|
| 64 | Viu | Viu | viu | 11 (Streaming) |
| 65 | WeTV | WeTV | wetv | 11 (Streaming) |
| 66 | by.U Data | by.U | byu-data | 4 (Data) |

---

---

## [v1.19.2] — 2026-03-11

### Fix Payment Fee VA Tidak Masuk ke Total Bayar Invoice

- **Root Cause**: `OrderController::createDuitkuTransaction()` hanya menambahkan fee untuk metode QRIS (`SP`, `NQ`, `DQ`, `GQ`, `SQ`) via blok hardcode 0.7%. Untuk Virtual Account (BRI, BNI, Mandiri, dll), fee dari kolom `payment_methods.fee` / `fee_percent` **tidak pernah** ditambahkan ke `paymentAmount` yang dikirim ke Duitku. Akibatnya Duitku menerima 10.510, mengembalikan amount 10.510, order tersimpan dengan `fee=0, total_price=10.510`.
- **File**: `Laravel_Backend/app/Http/Controllers/Api/OrderController.php`
- **Perubahan**: Tambah blok `else` setelah kondisi QRIS:
  ```php
  } else {
      $feeFixed = (int) ($paymentMethod->fee ?? 0);
      $feePercent = (float) ($paymentMethod->fee_percent ?? 0);
      $feePercentAmount = (int) ceil($final_price * ($feePercent / 100));
      $totalFee = max(0, $feeFixed + $feePercentAmount);
      $final_price = $final_price + $totalFee;
  }
  ```
- **Efek**: Semua metode Duitku (VA, e-wallet, dll) sekarang menambahkan fee ke amount sebelum dikirim ke API Duitku. Invoice menampilkan harga + fee = total yang benar.

---

## [v1.19.3] — 2026-03-11

### Fix #1 — Auto Refund ke Wallet Saat Digiflazz Gagal

- **Root Cause**: `WalletService::refundOrder()` memiliki guard `if ($order->payment_method !== 'SALDO') return null;` yang memblokir refund untuk semua metode selain Saldo (VA, QRIS Manual, dll). Selain itu, alur order Saldo tidak memanggil `refundOrder` setelah Digiflazz submit.
- **File**: `Laravel_Backend/app/Services/WalletService.php`
  - Hapus guard `payment_method !== 'SALDO'`
  - Sekarang semua order dengan `user_id` dapat di-refund ke wallet apapun metode pembayarannya
- **File**: `Laravel_Backend/app/Http/Controllers/Api/OrderController.php`
  - Pada alur pembayaran Saldo: tambah `refundOrder($saldoOrder)` di dalam `dispatch()->afterResponse()` setelah `buy_status === 'Gagal'`
- **File**: `Laravel_Backend/app/Http/Controllers/DigiflazzWebhookController.php` (QrispyWebhook)
  - Tambah blok refund setelah `DigiflazzService::submit()` jika `buy_status === 'Gagal'`

### Fix #2 — Riwayat Transaksi Tampil "Sukses" Padahal Status Gagal

- **Root Cause**: `TransactionController::normalizeBuyStatus()` mengembalikan `'Sukses'` hanya karena `payment_status='PAID'`, tanpa membaca `buy_status`. Order yang dibayar tapi Digiflazz gagal (`buy_status=Gagal`) tetap tampil "Sukses" di riwayat.
- **File**: `Laravel_Backend/app/Http/Controllers/Api/TransactionController.php`
- **Perubahan**: Saat `payment_status=PAID`, cek `buy_status` terlebih dahulu:
  - `PAID` + `buy_status=Gagal` → **Gagal**
  - `PAID` + `buy_status=Proses/Pending` → **Proses**
  - `PAID` + `buy_status=Sukses` (atau kosong) → **Sukses**

---

## [v1.19.4] — 2026-03-11

### Refund Terpusat di DigiflazzService (Semua Metode Pembayaran)

- **Root Cause**: Setiap callback controller (Duitku, Tripay, Paydisini, SmpQris, Qrispy) harus mengingat untuk memanggil `refundOrder()` sendiri setelah Digiflazz submit gagal. Jika ada exception di `afterCommit` (khususnya SmpQrisCallbackController), refund tidak pernah terpanggil dan tidak ada log error.
- **File**: `Laravel_Backend/app/Services/DigiflazzService.php`
  - Tambah `use App\Services\WalletService;`
  - Tambah method `triggerRefundIfNeeded(Order $order)`:
    - Cek `user_id` harus ada
    - Cek `payment_status === 'PAID'` (uang sudah masuk)
    - Panggil `WalletService::refundOrder()` — dilindungi guard double-refund
    - Dibungkus `try-catch` dengan `Log::error` agar tidak mengganggu alur utama
  - Di `processDigiflazzTransaction()`: panggil `triggerRefundIfNeeded()` pada 2 kondisi:
    1. HTTP error dari Digiflazz (sekarang selalu set `buy_status=Gagal`)
    2. Response 200 dengan status `Gagal` (misal "Nomor tujuan salah")
- **File**: `Laravel_Backend/app/Http/Controllers/SmpQrisCallbackController.php`
  - Bungkus isi `DB::afterCommit` dengan `try-catch` + `Log::error('[SMP-QRIS] AFTER_COMMIT_ERROR', ...)`
  - Tambah `$order->refresh()` setelah `processDigiflazzTransaction()` sebelum dispatch event
  - Efek: exception apapun tidak lagi hilang tanpa jejak

#### Diagram Perlindungan Refund (3 Lapis):
```
Digiflazz Gagal
    └─→ DigiflazzService::triggerRefundIfNeeded()  [lapis 1 — terpusat]
            └─→ WalletService::refundOrder()
                    └─→ Guard: cek SaldoHistory (type=refund, reference=order_id)
                                [lapis 2 — cegah double refund]
DigiflazzSyncPendingJob (cron)
    └─→ $wallet->refundOrder($order)               [lapis 3 — untuk order Pending lama]
```

#### Metode Pembayaran yang Terlindungi:
| Metode | Controller | Status |
|---|---|---|
| Saldo Danpedia | OrderController | ✅ |
| Duitku VA | DuitkuCallbackController | ✅ |
| Tripay | TripayCallbackController | ✅ |
| Paydisini | PaydisiniCallbackController | ✅ |
| QRIS Manual (SMP) | SmpQrisCallbackController | ✅ |
| QRIS Py | QrispyWebhookController | ✅ |

---

---

## [v1.20.0] — 2026-03-15

### Integrasi Produk Digital Koalastore

Penambahan sistem produk digital berbasis integrasi dengan Koalastore API. Produk digital (Netflix, Canva, Spotify, dll) dikelola secara terpisah dari produk Digiflazz melalui tabel `products` dengan `provider='koalastore'` dan ditampilkan di halaman `/digital/[slug]`.

---

### Fix #1 — Sinkronisasi Produk Koalastore: Duplikasi Kategori "Digital"

- **Root Cause**: `SyncKoalastoreProductsCommand` menggunakan `'Digital'` di `$categoryMap` dan `firstOrCreate`. Setiap kali sync berjalan, sistem membuat atau mencari kategori bernama `'Digital'` (bukan `'Produk Digital'`), sehingga setiap run membuat kategori duplikat baru.
- **File**: `Laravel_Backend/app/Console/Commands/SyncKoalastoreProductsCommand.php`
- **Perubahan**:
  - Seluruh `$categoryMap` diubah dari `'Digital'` → `'Produk Digital'`
  - `firstOrCreate` fallback diubah dari `['title' => 'Digital']` → `['title' => 'Produk Digital', 'sort' => 8]`
- **DB**: Hapus kategori duplikat "Digital" (sort 99) via SQL

---

### Fix #2 — Sinkronisasi Produk Koalastore: Profit Margin Tidak Terbaca

- **Root Cause**: `Setting::getSetting('koalastore')` mengembalikan null karena package `outerweb/filament-settings` menyimpan setiap field sebagai row terpisah dengan key dot-notation penuh (contoh: key row = `koalastore.profit_percent`, bukan `koalastore`).
- **File**: `Laravel_Backend/app/Console/Commands/SyncKoalastoreProductsCommand.php`
- **Perubahan**:
  - `Setting::getSetting('koalastore')['profit_percent']` → `Setting::getSetting('koalastore.profit_percent', 10)`
  - Opsi `--force-update` dihapus penggunaan `max()` yang mencegah harga turun; sekarang selalu pakai `$sellingPrice` kalkulasi terbaru
  - Harga gold/platinum juga dikalkulasi dari `$profitPercent` yang sama

---

### Fix #3 — Sinkronisasi Produk Koalastore: Label "Developers" Salah

- **Root Cause**: Field `developers` di tabel `games` di-hardcode sebagai `'Koalastore.digital'` sehingga tampil sebagai sub-label produk di frontend.
- **File**: `Laravel_Backend/app/Console/Commands/SyncKoalastoreProductsCommand.php`
- **Perubahan**: `developers` diubah dari `'Koalastore.digital'` menjadi `$koalaCat` (nama kategori Koalastore, misal: "Streaming", "Music Streaming", "Productivity Tools", "Security", "Creative", "Entertainment")
- **DB**: Jalankan sync `--force-update` untuk update 22 record yang sudah ada

---

### Fix #4 — Gambar Admin Tidak Muncul di Daftar Produk Digital

- **Root Cause**: `DigitalProductController::index()` hanya membaca `$product->images` (URL koalastore). Gambar yang di-upload admin di tabel `games` tidak pernah dipakai di endpoint list.
- **File**: `Laravel_Backend/app/Http/Controllers/Api/DigitalProductController.php`
- **Perubahan** di method `index()`:
  - Pre-load `$gameImages` dan `$gameBanners` dari tabel `games` (keyed by slug)
  - Prioritaskan gambar admin (`games.image`) atas gambar koalastore (`products.images`)
- **Catatan**: Method `show()` sudah benar (sudah mengecek `$game->image` terlebih dahulu)

---

### Fix #5 — Urutan Grup Pembayaran: QRIS di Bawah, Harusnya di Atas

- **Root Cause**: `Object.entries()` pada halaman digital mengikuti urutan insert API. Kelompok QRIS muncul di bawah karena urutan DB.
- **File**: `NextJS_FrontEnd_New/src/app/digital/[slug]/page.tsx`
- **Perubahan**:
  - Tambah sort eksplisit dengan prioritas group: `Saldo → QRIS → Virtual Account → Retail → E-Wallet → Lainnya`
  - Tambah sort `outside_sort` untuk item di dalam setiap group (konsisten dengan halaman order biasa)

---

### Fitur Baru — Warranty Terms Modal: Tab per Varian + Dua Kolom

- **Background**: Sebelumnya modal garansi hanya berupa satu RichEditor (field teks tunggal).
- **File**: `Laravel_Backend/app/Filament/Resources/GameResource.php`
  - Ganti `RichEditor` dengan `Repeater` — setiap item mewakili satu varian produk
  - Field per item: `tab_name` (nama varian), `syarat` (Syarat & Ketentuan), `garansi` (Ketentuan Garansi)
- **File**: `Laravel_Backend/app/Models/Game.php`
  - Tambah `'warranty_terms' => 'array'` pada `$casts`
- **File**: `NextJS_FrontEnd_New/src/app/digital/[slug]/page.tsx`
  - Tambah interface `WarrantyTab { tab_name, syarat, garansi }`
  - Modal warranty didesain ulang: baris tab varian di atas + dua kolom di bawah (kiri: Syarat & Ketentuan biru, kanan: Ketentuan Garansi hijau)
  - State `activeWarrantyTab` untuk navigasi antar tab
- **Artisan Command Baru**: `koalastore:import-warranty`
  - File: `Laravel_Backend/app/Console/Commands/ImportKoalastoreWarrantyTerms.php`
  - Fetch dari `https://koalastore.digital/api/warranty-terms`, konversi markdown → HTML
  - Simpan sebagai JSON array di kolom `warranty_terms`
  - **Hasil run pertama**: 23 produk diupdate (NETFLIX: 4 tab, CANVA: 7 tab, dll)

---

### Fix #6 — Invoice: Logo Produk Digital Tidak Muncul

- **Root Cause**: `InvoiceController::mapGame()` hanya mengecek `$g->image` jika record Game ditemukan, tetapi tidak pernah fallback ke gambar koalastore (`Product->images`).
- **File**: `Laravel_Backend/app/Http/Controllers/Api/InvoiceController.php`
- **Perubahan**: Refactor `mapGame()` — resolve `$imageUrl` terpisah:
  1. Cek `$g->image` (gambar upload admin)
  2. Fallback ke `Product->images` (gambar koalastore) jika ada

---

### Fix #7 — Trending & Daftar Produk: Produk Digital Mengarah ke URL Salah

- **Root Cause**: Komponen trending, game list, dan search tidak membedakan produk digital (harusnya `/digital/[slug]`) dari produk biasa (`/order/[slug]`). Semua produk digital dari Koalastore diarahkan ke halaman order yang salah.
- **File**: `Laravel_Backend/app/Http/Controllers/Api/GameController.php`
  - Pre-load `$digitalCategoryId` via `Category::where('title', 'Produk Digital')->value('id')`
  - Tambah field `is_digital` (boolean) ke response `populerGames`, `games`, dan `search()`
- **File**: `NextJS_FrontEnd_New/src/components/home/game-populer.tsx`
  - Tambah `is_digital?: boolean` ke interface `Game`
  - `href` diubah: `game.is_digital ? '/digital/${slug}' : '/order/${slug}'`
- **File**: `NextJS_FrontEnd_New/src/components/home/game-list.tsx`
  - Sama seperti `game-populer.tsx`
- **File**: `NextJS_FrontEnd_New/src/components/panel/search.tsx`
  - Tambah `is_digital?: boolean` ke interface `Game`
  - Fix link di hasil pencarian desktop dan mobile

---

### State Produk Digital (setelah semua fix):
| Fitur | Status |
|---|---|
| Sinkronisasi produk dari Koalastore | ✅ |
| Profit margin dari setting admin (`koalastore.profit_percent`) | ✅ |
| Label developers = nama kategori (Streaming, Music Streaming, dll) | ✅ |
| Gambar admin prioritas atas gambar Koalastore | ✅ |
| Urutan grup pembayaran: QRIS di atas | ✅ |
| Warranty modal: tab per varian + dua kolom | ✅ |
| 23 produk terisi warranty dari Koalastore API | ✅ |
| Invoice menampilkan logo produk digital | ✅ |
| Trending/popular/search: link ke `/digital/[slug]` | ✅ |

---

## [v1.20.1] — 2026-03-17

### Fix — Daftar Harga: Tombol "Order" Produk Digital Mengarah ke URL Salah

- **Root Cause**: Halaman `/price-list` selalu membangun href `/order/[slug]?product_id=...` untuk semua produk tanpa membedakan produk digital. Produk seperti Alight Motion, Canva, dll seharusnya diarahkan ke `/digital/[slug]` (halaman pilih varian digital), bukan ke halaman order biasa.
- **File**: `/var/www/NextJS_FrontEnd/src/app/price-list/page.tsx`
- **Perubahan**: Tambah deteksi `isDigital` berdasarkan `selectedGame.category_ids` mengandung `12` (ID kategori "Produk Digital"):
  ```tsx
  const isDigital = selectedGame?.category_ids?.some((id: any) => Number(id) === 12);
  const href = selectedGame?.slug
    ? (isDigital
        ? `/digital/${selectedGame.slug}`
        : `/order/${selectedGame.slug}?product_id=${encodeURIComponent(String(product.id))}`)
    : "#";
  ```
- **Efek**: Produk digital → `/digital/[slug]`, produk lain → `/order/[slug]?product_id=...` (tidak berubah)
- **Deploy**: `npm run build` + `pm2 restart danpedia`

---

## [v1.20.2] — 2026-03-17

### SEO — Google Search Console & Sitemap Setup

#### Fix robots.txt: Domain Lama
- **Root Cause**: `robots.txt` masih menunjuk ke domain lama `topup.imdpoin.com` di baris `Sitemap:`
- **File**: `/var/www/NextJS_FrontEnd/public/robots.txt`
- **Perubahan**: `Sitemap: https://topup.imdpoin.com/sitemap.xml` → `Sitemap: https://www.dan-pedia.com/sitemap.xml`

#### Fix NEXT_PUBLIC_SITE_URL: Inkonsistensi www
- **Root Cause**: `NEXT_PUBLIC_SITE_URL=https://dan-pedia.com` (tanpa `www`), sedangkan Google Search Console didaftarkan sebagai `https://www.dan-pedia.com`. Sitemap menghasilkan URL tanpa `www` sehingga Google melaporkan "Couldn't fetch" saat pertama submit.
- **File**: `/var/www/NextJS_FrontEnd/.env.local`
- **Perubahan**: `https://dan-pedia.com` → `https://www.dan-pedia.com`
- **Efek**: Semua URL di `sitemap.xml` sekarang konsisten menggunakan `https://www.dan-pedia.com`

#### Tambah Google Site Verification Meta Tag
- **Tujuan**: Verifikasi kepemilikan domain di Google Search Console via HTML tag method
- **File**: `/var/www/NextJS_FrontEnd/src/app/layout.tsx`
- **Perubahan**: Tambah field `verification` di kedua blok metadata (success & fallback):
  ```tsx
  verification: { google: "q8cHxdKdTH1NG60cwuhe5aKr2NHdPL9Yi-sKVPOHMbY" },
  ```
- **Deploy**: `npm run build` + `pm2 restart danpedia`

#### Hasil Akhir:
| Item | Status |
|---|---|
| Google Search Console terverifikasi | ✅ |
| Sitemap `https://www.dan-pedia.com/sitemap.xml` submitted | ✅ |
| Google menemukan **50 halaman** dari sitemap | ✅ |
| robots.txt menunjuk ke domain yang benar | ✅ |
| URL sitemap konsisten dengan `www` | ✅ |

---

*File ini terakhir diupdate: 2026-03-17*
