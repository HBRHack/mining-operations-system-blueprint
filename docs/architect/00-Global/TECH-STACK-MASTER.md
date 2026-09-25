# Tech Stack Master: Simulasi Operasional Tambang (Loading–Hauling)

> Sistem-wide — berlaku untuk semua BAB. Lihat `ADR-MASTER.md` untuk alasan tiap pilihan.
>
> **Untuk siapa dokumen ini:** developer yang membaca repo ini — reviewer portofolio, kontributor, atau siapa pun yang mengevaluasi rancangan proyek. Isinya: opsi teknologi yang dipertimbangkan, cara menilainya **dalam konteks operasional tambang**, dan keputusan akhir beserta alasannya.

Stack inti terpilih: **PHP 8.3 + Laravel 12 (Blade server-rendered) + MariaDB 10.11** — dipilih karena (a) **offline-penuh** setelah satu kali instalasi, (b) **ringan** untuk hardware seadanya di kantor site, (c) **availability runtime & talent lokal tertinggi** di Indonesia, dan (d) paling cocok dengan bentuk masalah: aplikasi form-based dengan relasi many-to-many kompleks + state machine.

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 1.0 |
| **Last Updated** | 25 September 2026 |
| **Status** | Accepted (ADR-001) |

## ⭐ Keputusan (TL;DR)

| Layer | Pilihan | Alasan satu baris |
|-------|---------|-------------------|
| Bahasa | PHP 8.3.6 | Sudah terpasang; security support PHP 8.3 s/d **31 Des 2027** |
| Framework | Laravel 12 + Blade | Server-rendered murni, CRUD relasi kompleks tanpa SPA; sesuai CON-001/003/004 |
| Database | MariaDB 10.11.14 (GPLv2) | Sudah terpasang; Community LTS s/d **16 Feb 2028** — komponen dengan umur dukung terpanjang di stack ini |
| Styling | Tailwind via Vite build **lokal** | Tanpa CDN — aset ikut terbawa saat deploy offline |
| Session | File-based (tanpa Redis, tanpa queue) | <10 user bersamaan; menambah layanan = menambah yang rusak di lapangan |
| Testing | PHPUnit (bawaan Laravel) | NFR-TEST-001: ≥10 test (T-01…T-14 di `spec.md`) |
| CI/CD | Tidak ada pipeline | Uji jalan lokal (`php artisan test`); dependency internet hanya saat awal pengembangan |

## Konteks Operasional Tambang (dasar penilaian)

| # | Batasan lapangan | Implikasi ke pilihan stack |
|---|---|---|
| 1 | Area tambang terpencil, **tidak selalu ada internet** (VSAT/koneksi komunal) | Runtime wajib **offline penuh**: tanpa CDN, tanpa API cloud, tanpa registry saat berjalan |
| 2 | Hardware seadanya (laptop/mini-PC kantor camp), **listrik & uptime terbatas** | Stack harus ringan-RAM, mudah start ulang, berjalan di 1 mesin |
| 3 | Pemeliharaan bertahun-tahun oleh **satu orang IT site**, bukan tim DevOps | Instalasi & patch harus bisa dilakukan tanpa pipeline CI/CD |
| 4 | **Koneksi hanya tersedia pada waktu tertentu** (perjalanan ke kota/kantor pusat) | Sistem jarang di-patch → jendela dukungan framework yang panjang itu penting |
| 5 | Konteks pasar tenaga kerja Indonesia | Availability talent lokal = faktor seleksi, bukan selera |

> **Catatan penilaian berat–ringan:** bersifat **kualitatif** (kelas footprint berdasarkan aritektur runtime: proses, baseline memori, model kerja), bukan benchmark terukur di mesin proyek. Pengukuran performa nyata dilakukan saat implementasi lewat NFR-PERF-001 (dashboard < 2 detik).

## Stack Overview

| Layer | Technology | Version | License | Status |
|-------|-----------|---------|---------|--------|
| Language | PHP | 8.3.6 (terpasang) | PHP License v3.04 | Security support s/d 31 Des 2027 (php.net) |
| Web framework | Laravel | 12.x | MIT | Bug-fix berakhir 13 Agu 2026; security fix s/d 24 Feb 2027 |
| UI rendering | Blade + Tailwind (build lokal) | bawaan Laravel 12 / MIT | MIT | Aktif |
| Database | MariaDB Server | 10.11.14 (terpasang) | GPLv2 | Community LTS s/d 16 Feb 2028 (mariadb.org) |
| Asset build | Vite + Node.js + npm | Node 24.18.1 / npm 12.0.2 (terpasang) | MIT / MIT | Aktif (hanya saat pengembangan) |
| Testing | PHPUnit | bawaan Laravel 12 | MIT | Aktif |
| CI/CD | — (pengujian lokal) | — | — | Tidak dipakai (konteks offline) |
| Session store | File-based bawaan Laravel | — | — | — |

---

## Layer-by-Layer Breakdown

### 1. Programming Language

**Choice:** PHP 8.3 (terpasang di mesin pengembang: PHP 8.3.6, Composer 2.7.1)

| Research Area | Finding |
|---------------|---------|
| Supported Branches | 8.2, 8.3, 8.4, 8.5 (php.net/supported-versions, per 25 Sep 2026) |
| Active Support 8.3 | Berakhir 31 Des 2025 (masa kritis-bug sudah lewat) |
| Security Support 8.3 | **31 Des 2027** |
| Maintenance Status | Aktif — branch 8.4/8.5 menerima update penuh |
| Compatibility | Laravel 12 menerima PHP 8.2–8.5 → jalur upgrade terbuka |
| License | PHP License v3.04 — bebas komersial |

**Why this choice:** Konfigurasi terkunci (CON-001) sekaligus satu-satunya bahasa yang **sudah terpasang dan terverifikasi** di mesin pengembang. Jendela dukungan security 15 bulan ke depan lebih dari cukup untuk masa hidup proyek, dengan jalur upgrade ke 8.4/8.5 tanpa perubahan kode.

**Alternatives Rejected:**
- Python / Ruby / Java / Node.js sebagai bahasa utama: dinilai di ADR-001 — kalah pada kesesuaian bentuk masalah (form CRUD relasi), availability bundle offline, dan talent lokal.
- PHP 8.4/8.5 sejak awal: versi belum terpasang; menambah variabel instalasi di lingkungan yang harus bisa direproduksi.

**Reference:** https://www.php.net/supported-versions.php

---

### 2. UI Framework

**Choice:** Blade (server-rendered) + Tailwind via Vite build lokal, JavaScript minimal vanilla

| Research Area | Finding |
|---------------|---------|
| Rendering model | Server-side HTML — tanpa SPA, tanpa hydrasi klien (CON-003) |
| Offline Support | ✅ Penuh — hasil build (`public/build/`) ikut terbawa ke server site; **tanpa CDN** |
| Dilarang oleh kontrak | Livewire, Filament, starter kit Inertia/React/Vue (CON-003, CON-004) |
| Learning Curve | Rendah — template engine datar, mudah dibaca reviewer |
| License | MIT |

**Why this choice:** Bentuk aplikasi = form-based (24+ halaman form, tabel, dashboard). Blade menghasilkan HTML tunggal tanpa siklus build di runtime; Tailwind dipilih agar gaya konsisten tanpa menulis ratusan baris CSS manual, tetapi **di-build lokal sekali** — begitu `npm run build` selesai, Node tidak diperlukan lagi di server site.

**Alternatives Rejected:**
- **CDN Tailwind/Google Fonts:** gagal total saat runtime tanpa internet — melanggar batasan #1.
- **CSS tulis tangan 1 file:** bebas dependensi, tapi gaya ~20 halaman cepat tidak konsisten.
- **Livewire/Spark/Filament:** dilarang eksplisit (CON-004) — seluruh CRUD memang ditulis manual.
- **React/Vue SPA:** dilarang (CON-003); dobel kerjaan untuk data yang berubah tiap render.

**Reference:** https://laravel.com/docs/12.x/blade · https://laravel.com/docs/12.x/vite

---

### 3. Database

**Choice:** MariaDB Server 10.11.14 (terpasang), engine InnoDB, transaksi untuk atomik stok

| Research Area | Finding |
|---------------|---------|
| Latest Stable Line | 12.3 (GA 28 Mei 2026); yang terpasang 10.11.14 LTS |
| GA / EOL 10.11 | GA 16 Feb 2023 — **Community & Extended EOL: 16 Feb 2028** |
| Maintenance Status | Aktif; 10.11 adalah LTS dengan dukungan terpanjang di barisnya (mariadb.org) |
| License | GPLv2 — "will remain Free and Open Source Software" (jaminan MariaDB Foundation) |
| Embedded / Client-Server | Client-server (Apache + app + DB dalam 1 mesin) |
| Driver Support | `pdo_mysql` bawaan Laravel — resmi disupport (Laravel butuh MariaDB 10.3+) |
| Encryption Support | Enkripsi InnoDB at-rest tersedia (data proyek 100% fiktif — CON-005) |
| Max DB Size | Tak terbatas praktis; proyek < 10 ribu baris/tahun |

**Benchmark Data:**

| Operation | Hasil |
|-----------|-------|
| Single INSERT | Belum diukur — wajib diukur saat implementasi (NFR-PERF-001) |
| SELECT with index | Belum diukur — idem |
| Complex JOIN (dashboard) | Belum diukur — target < 2 detik (FR dashboard) |

> Angka benchmark sengaja **tidak diisi** sampai diukur di mesin proyek — mengisi dengan angka karangan akan menyesatkan pembaca repo ini.

**Why this choice:** CON-002 mengunci MySQL/MariaDB; MariaDB 10.11 **sudah terpasang** (menghindari setup service baru di mesin yang harus bisa dipulihkan tanpa internet) dan justru punya **umur dukung terpanjang di seluruh stack** (2028) — komponen paling kritis justru paling awet. Transaksi ACID InnoDB menopang aturan "stok tidak minus" + pengambilan part atomik (BR terkait NFR-REL-001).

**Alternatives Rejected:**
- **MySQL 8:** setara fitur, tetapi harus install service tambahan tanpa keunggulan nyata di konteks ini.
- **SQLite:** mode embedded menarik untuk offline, tetapi konkurensi 2 shift + aturan atomik stok lebih aman di client-server; juga di luar batasan CON-002.
- **PostgreSQL:** di luar CON-002; instalasi & tuning ekstra tanpa tuntutan fitur yang membutuhkannya.
- **MariaDB 12.3:** rilis terbaru, tetapi bukan LTS lama — mengorbankan jendela dukungan demi fitur yang tidak dipakai.

**Reference:** https://mariadb.org/about/#maintenance-policy · https://laravel.com/docs/12.x/database

---

### 4. Build System

**Choice:** Vite (dev/build) + npm 12 — hanya saat pengembangan

| Research Area | Finding |
|---------------|---------|
| Latest Version | Vite bawaan Laravel 12; Node 24.18.1 / npm 12.0.2 terpasang |
| Language Support | CSS/JS proyek |
| Runtime Dependency | ❌ Tidak — hasil build statis disajikan web server |
| Offline Support | ✅ Setelah `npm install` awal, `node_modules` + `public/build` self-contained |

**Why this choice:** Satu perintah `npm run build` menghasilkan aset statis; server site **tidak pernah butuh Node, npm, atau internet lagi.** Alternatif tanpa build (CSS mentah) ditolak demi konsistensi gaya; alternatif CDN ditolak karena runtime offline.

**Reference:** https://laravel.com/docs/12.x/vite

---

### 5. Testing Framework

**Choice:** PHPUnit (bawaan Laravel 12) + feature test untuk alur kritis

| Research Area | Finding |
|---------------|---------|
| Unit Testing | PHPUnit |
| Integration Testing | Laravel TestCase + database testing (mysql test terpisah) |
| UI Testing | Manual — aplikasi form sederhana; bukan prioritas |
| Code Coverage | Opsional (`--coverage`) — bukan gate |
| Target | ≥ 10 test otomatis (NFR-TEST-001; skenario T-01…T-14 di `spec.md`) |

**Why this choice:** Sudah terpasang bersama framework — nol dependensi tambahan. Test wajib menutup jalur kritis: transaksi stok atomik, validasi negatif stok, dan transisi 5 state alat.

**Reference:** https://laravel.com/docs/12.x/testing

---

### 6. CI/CD Pipeline

**Choice:** Tidak ada pipeline — pengujian dijalankan lokal; Git untuk version control

| Research Area | Finding |
|---------------|---------|
| Platform | Lokal (Git lokal; remote opsional untuk publikasi repo — bukan dependency) |
| Package Build | `composer install` + `npm run build` sekali di luar lokasi tambang |
| Auto-Update | Manual, saat koneksi tersedia — selaras batasan #4 |

**Why this choice:** Repo ini bersifat proyek latihan/portofolio yang dijalankan offline; pipeline CI berarti dependensi internet berkelanjutan dan akun layanan pihak ketiga — dua hal yang justru tidak tersedia di lapangan. Deploy = salin folder + jalankan migration/seeder.

**Reference:** —

---

## Opsi Stack yang Dipertimbangkan (penilaian konteks tambang)

### 1. Laravel 12 + Blade — ⭐ TERPILIH

**Kelebihan (konteks tambang):**
- **Offline penuh:** setelah `composer install` saat masih ada koneksi, seluruh `vendor/` + aset lokal ikut terbawa → berjalan bertahun-tahun tanpa menyentuh internet.
- **Cukup ringan untuk 1 mesin camp:** Apache/PHP-FPM + MariaDB pada satu laptop/mini-PC melayani LAN site <10 user bersamaan; beban nyata proyek (~60 baris log shift/bulan, 10–15 WO) jauh di bawah kapasitasnya.
- **Availability runtime tertinggi di kondisi minim koneksi:** bundel PHP+MariaDB (Laragon/XAMPP/offline installer) paling mudah didapat dan paling dikenal IT site — penting saat harus instal ulang di lapangan.
- **Availability talent lokal terbaik:** PHP/Laravel tulang punggung pengembangan web Indonesia → pemeliharaan bisa diwariskan ke tenaga IT site mana pun tanpa rekrut spesialis langka.
- Eloquent menangani relasi many-to-many + state machine 5 state + transaksi stok atomik tanpa kerangka tambahan.

**Kelemahan (konteks tambang):**
- **Jendela patch paling pendek di antara opsi PHP:** bug-fix Laravel 12 berakhir 13 Agu 2026, security fix s/d 24 Feb 2027 — buruk bila dikombinasikan batasan #4 (sistem jarang di-patch). *Mitigasi:* upgrade ke Laravel 13 terbuka lebar (PHP 8.3 kompatibel) → tercatat sebagai risiko **TR-001**.
- **Butuh internet sekali di awal** (`composer create-project`) — wajib diselesaikan sebelum masuk area tambang. Berlaku untuk semua opsi, tetapi tetap langkah kritis.
- Idle footprint menengah (Apache + PHP-FPM + MariaDB) — bukan yang paling hemat daya; wajar untuk satu mesin dedicated.

### 2. Laravel 13

**Kelebihan (konteks tambang):** jendela security sampai **Q1 2028** — paling cocok untuk sistem yang patch-nya hanya bisa dilakukan saat koneksi tersedia; footprint identik dengan Laravel 12.
**Kelemahan (konteks tambang):** katalog tutorial/paket komunitas paling kaya masih berpusat di Laravel 12; seluruh dokumen SA proyek ini (CON-001) memakai Laravel 12 → menggantinya = perubahan terkontrol lewat change-management, bukan keputusan teknis bebas.

### 3. Django 6.1 + Template Django

**Kelebihan (konteks tambang):** **Python adalah bahasa dominan di data & mine planning** (laporan produksi, analisis) → talent melebar: engineer tambang yang menguasai Python bisa ikut memelihara; footprint ringan; dukungan panjang (6.1 s/d Des 2027; LTS 5.2 s/d Apr 2028); auth + ORM bawaan.
**Kelemahan (konteks tambang):** deployment di server Windows (lazim di kantor site) lebih repot — butuh venv + wrapper layanan Windows untuk WSGI, tidak setingkat Laragon; Django Admin justru menggoda memakai CRUD bawaan, padahal manual CRUD adalah syarat proyek; ekosistem hosting lokal kurang lazim → regenerasi maintainer lebih sulit.

### 4. Ruby on Rails 8.1

**Kelebihan (konteks tambang):** rilis sangat aktif (8.1.4 — 24 Sep 2026), konvensi matang, scaffold cepat.
**Kelemahan (konteks tambang):** **availability tooling terlemah** — Ruby tidak terpasang di mesin pengembang; bootstrap rbenv/RVM = unduh + kompilasi panjang, sangat menyakitkan di koneksi VSAT; talent Ruby di Indonesia langka → risiko sistem terbengkalai setelah pembawanya pindah.

### 5. Spring Boot 4.1 (Java 21)

**Kelebihan (konteks tambang):** Java 21 tersedia; state machine & transaksi diekspresikan paling tegas (enum + service); umur framework enterprise terpanjang.
**Kelemahan (konteks tambang):** **paling berat** — baseline JVM (ratusan MB RAM, cold start berdetik-detik) boros untuk hardware camp dan tidak sepadan untuk <10 user; boilerplate CRUD 3–4× lebih panjang dari Laravel → waktu perawatan satu maintainer membengkak; build Maven/Gradle menarik volume dependency internet terbesar saat awal proyek.

### 6. Express 5 + templat

**Kelebihan (konteks tambang):** paling ringan & start instan (Node 24 tersedia); `node_modules` self-contained → offline setelah install.
**Kelemahan (konteks tambang):** **tanpa bawaan** — session, validasi, migrasi, ORM, state machine semua rakitan tangan → beban pemahaman penuh di pundak satu maintainer yang bekerja tanpa internet dan tanpa rekan diskusi; **availability maintainer dependensi sedang buruk: tim keamanan Express menunda rilis 17 Sep–6 Okt 2026** (expressjs.com) — untuk sistem yang jarang di-patch, jeda patch upstream adalah risiko nyata, bukan catatan kaki.

### (Diskualifikasi) FastAPI 0.141

Kuat untuk JSON API + dokumentasi OpenAPI otomatis (MIT, rilis 29 Jul 2026), tetapi proyek ini berbentuk **form POST + halaman HTML** — memakai FastAPI berarti membangun layer SSR di atas framework API = melawan arah desainnya. Kesesuaian bentuk masalah jatuh ke Laravel/Django.

### Perbandingan ringkas

| Opsi | Kelas footprint | Offline runtime | Bootstrap awal di mesin dev | Availability talent (ID) | Dukungan framework | Putusan |
|---|---|---|---|---|---|---|
| **Laravel 12 + MariaDB** | Sedang-ringan | ✅ Penuh setelah `composer install` sekali | ✅ Termudah (Laragon/XAMPP) | ✅ Sangat tinggi | Security s/d Feb 2027 | ⭐ **Dipilih** |
| Laravel 13 | Sedang-ringan | ✅ Penuh | ✅ Mudah | Tinggi | s/d Q1 2028 | Jalur upgrade (via change-management) |
| Django 6.1 | Ringan | ✅ Penuh | ⚠️ Sedang (venv + wrapper Windows) | Tinggi — Python lazim di data tambang | s/d Des 2027 (LTS 2028) | Alternatif terkuat |
| Rails 8.1 | Sedang | ✅ Penuh | ❌ Sulit (Ruby tidak terpasang) | ❌ Rendah di pasar ID | Aktif (8.1.4) | Tidak dipilih |
| Spring Boot 4.1 | ❌ Paling berat | ✅ Penuh | ⚠️ Java 21 ada; dependency build terbanyak | Sedang | Aktif (4.1.1) | Tidak dipilih |
| Express 5 | ✅ Paling ringan | ✅ Penuh | ✅ Node 24 tersedia | Sedang-tinggi | ⚠️ Tim keamanan pause sementara | Tidak dipilih |
| FastAPI 0.141 | Ringan | ✅ | ✅ Python 3.12 | Tinggi | Aktif | Diskualifikasi (bentuk masalah API vs form) |

---

## Compatibility Matrix

| Component A | Component B | Compatible | Notes |
|-------------|-------------|------------|-------|
| PHP 8.3.6 | Laravel 12 | ✅ | Laravel 12 menerima PHP 8.2–8.5 |
| Laravel 12 | MariaDB 10.11.14 | ✅ | Laravel resmi mendukung MariaDB 10.3+ |
| MariaDB 10.11 | `pdo_mysql` | ✅ | Ekstensi PHP bawaan |
| Vite/Tailwind | Blade | ✅ | Alur resmi Laravel (`@vite` directive) |
| PHPUnit | Laravel 12 | ✅ | Bawaan framework |
| Node 24 / npm 12 | Proses build | ✅ | Hanya dibutuhkan saat pengembangan |

## License Summary

| Component | License | Commercial Use | Distribution |
|-----------|---------|----------------|--------------|
| PHP 8.3 | PHP License v3.04 | ✅ Allowed | ✅ Bebas |
| Laravel 12 | MIT | ✅ Allowed | ✅ Bebas |
| Blade / Tailwind / Vite | MIT | ✅ Allowed | ✅ Bebas |
| MariaDB 10.11 | GPLv2 | ✅ Allowed | Penggunaan jaringan (ASP clause) tidak memaksa pembukaan source aplikasi; pasal source-berbagi MariaDB aktif hanya saat **distribusi biner** MariaDB itu sendiri |
| Node.js / npm | MIT | ✅ Allowed | ✅ Bebas |
| PHPUnit / Composer | MIT | ✅ Allowed | ✅ Bebas |

**Total License Cost:** Rp 0 — seluruh stack open source, tanpa komponen komersial, tanpa tier berbayar.

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **TR-001** — Fase bug-fix Laravel 12 berakhir 13 Agu 2026 (tinggal security fix s/d 24 Feb 2027) | Tinggi | Jalur upgrade Laravel 13 terbuka (PHP 8.3 kompatibel, estimasi ≤1 hari); `APP_DEBUG=false` saat deploy; aplikasi tidak terekspos internet (LAN site saja) memperkecil permukaan serangan |
| **TR-002** — PHP 8.3 security support berakhir 31 Des 2027 | Sedang | Upgrade minor ke PHP 8.4/8.5 (diterima Laravel 12 & 13) saat koneksi tersedia |
| **TR-003** — MariaDB 10.11 Community EOL 16 Feb 2028 | Rendah | Jendela terpanjang di stack (1.5 tahun lagi); jalur upgrade ke 11.8 LTS (s/d Jun 2028) atau 12.x saat tiba waktunya |
| **TR-004** — Instalasi awal butuh internet (composer/npm) dan gagal di lokasi tanpa koneksi | Sedang | Selesaikan scaffold + `composer install` + `npm run build` **di luar lokasi tambang**; commit `vendor/` atau simpan bundle offline; dokumentasikan langkah di README |
| **TR-005** — Kehilangan satu-satunya maintainer IT site | Sedang | Talent PHP paling melimpah di ID + dokumentasi SA/arsitektur lengkap di repo → onboarding pengganti paling murah di antara semua opsi |

## References

- https://www.php.net/supported-versions.php — PHP Supported Versions (cabang 8.2–8.5, jendela dukungan per branch)
- https://laravel.com/docs/12.x/releases — Laravel 12 Release Notes & Support Policy (bug fix 13 Agu 2026, security 24 Feb 2027)
- https://laravel.com/docs/13.x/releases — Laravel 13 Release Notes (rilis Q1 2026, PHP 8.3–8.5)
- https://laravel.com/docs/12.x/database — Database: Getting Started (dukungan MariaDB 10.3+)
- https://mariadb.org/about/#maintenance-policy — MariaDB Server long-term release maintenance periods (10.11 EOL 16 Feb 2028)
- https://www.djangoproject.com/download/ — Django download & support policy (6.1.1, LTS 5.2)
- https://rubyonrails.org/ — Ruby on Rails (8.1.4, 24 Sep 2026)
- https://spring.io/projects/spring-boot — Spring Boot (4.1.1)
- https://expressjs.com/ — Express (5.2.1; catatan pause tim keamanan 17 Sep–6 Okt 2026)
- https://pypi.org/project/fastapi/ — FastAPI (0.141.1, MIT)
