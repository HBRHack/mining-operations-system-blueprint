# ADR-001: PHP 8.3 + Laravel 12 + Blade (Server-Rendered) sebagai Stack Inti

Memilih PHP 8.3 + Laravel 12 dengan rendering Blade server-rendered dan MariaDB 10.11 sebagai stack inti aplikasi, karena satu-satunya kandidat yang memenuhi sekaligus: runtime offline-penuh, footprint ringan untuk hardware site, availability talent lokal Indonesia tertinggi, dan kesesuaian bentuk masalah (form CRUD relasi kompleks + state machine).

## Metadata

| Field | Value |
|-------|-------|
| **ADR ID** | ADR-001 |
| **Status** | Accepted |
| **Date** | 25 September 2026 |
| **Deciders** | Pemilik proyek (persetujuan) · architect-grill-with-docs (riset & rekomendasi) |
| **Related ADRs** | — (ADR pertama; ADR berikutnya: session-store, deployment, backup) |
| **Related Requirements** | CON-001, CON-002, CON-003, CON-004, NFR-PORT-001, NFR-PERF-001, NFR-SEC-001 |
| **Scope** | Global |
| **Chapter(s) Terdampak** | Global (semua BAB-1…BAB-6) |

## Context

Proyek adalah aplikasi web latihan simulasi operasional tambang (3 modul: Equipment Monitoring, Inventory Spare Part, Maintenance Work Order) berbentuk **aplikasi form-based server-rendered**, data 100% fiktif, dengan kebutuhan: state machine 5 state alat, relasi many-to-many (pemakaian part), transaksi stok atomik (stok tidak minus), dashboard < 2 detik, UI Bahasa Indonesia.

**Technical Context:**
- Mesin pengembang terverifikasi (25 Sep 2026): PHP 8.3.6 · Composer 2.7.1 · MariaDB 10.11.14 · Node 24.18.1 / npm 12.0.2 · Python 3.12.3 · Java 21 · **Ruby tidak terpasang**.
- Belum ada kode aplikasi; seluruh dokumen SA (SRS, matrix, ERD, business rules) telah terbit lebih dulu.
- Kontrak keputusan awal (CON-001…CON-004) sudah mengunci: Laravel 12 + PHP 8.3, MySQL/MariaDB, Blade + JS minimal (tanpa SPA), tanpa Livewire/Filament/CRUD-generator.

**Business Context:**
- Tujuan: latihan (membangun pemahaman relasi & state machine) sekaligus portofolio yang dibaca developer/perekrut di GitHub.
- Deployment di area tambang terpencil ("di hutan"): **tanpa internet yang andal**; pemeliharaan oleh satu orang IT site, bukan tim DevOps.
- Beban nyata kecil: <10 user bersamaan (2 shift), ~5–10 alat, 15 part, ~60 log/bulan, 10–15 WO/bulan.

**Forces at Play:**
- **Offline-vs-modern:** setiap pilihan yang menarik resource dari internet saat runtime (CDN, API cloud, registry) otomatis gugur.
- **Ringan-vs-berat:** hardware seadanya + listrik terbatas menghukum stack berat (baseline JVM) tanpa memberi manfaat nyata di beban <10 user.
- **Jendela dukungan-vs-jarang-patch:** sistem yang patch-nya hanya bisa dilakukan saat koneksi tersedia sangat bergantung pada panjangnya support window framework.
- **Kesesuaian bentuk masalah:** aplikasi = banyak form HTML + relasi kompleks; kandidat API-first (FastAPI) atau scaffolding-generator (Django Admin, Rails) justru melawan tujuan proyek.
- **Kesinambungan pemeliharaan:** maintainer pengganti harus bisa ditemukan di pasar tenaga kerja Indonesia.

## Decision

**Kita akan memakai PHP 8.3 + Laravel 12 + Blade server-rendered + MariaDB 10.11 sebagai stack inti**, karena kandidat yang paling sesuai dengan bentuk masalah dan konteks operasional tambang (offline penuh, ringan, talent lokal melimpah), sekaligus mematuhi kontrak CON-001…CON-004 yang sudah disepakati.

Komponen pendukung: Tailwind via Vite **build lokal** (tanpa CDN), session file-based (tanpa Redis/queue), PHPUnit bawaan, tanpa CI/CD (ujian lokal).

## Alternatives Considered

### Option A: Django 6.1 + Template Django (Python)

| Aspect | Assessment |
|--------|------------|
| Description | Framework Python "batteries included": auth, admin, ORM, template server-side |
| Pros | Footprint ringan; Python lazim di data & mine planning (talent melebar); dukungan panjang (6.1 → Des 2027, LTS 5.2 → Apr 2028); Python 3.12 terpasang |
| Cons | Deployment Windows (lazim di kantor site) butuh venv + wrapper layanan; Django Admin menggoda CRUD bawaan padahal manual CRUD adalah syarat proyek; ekosistem hosting lokal kurang lazim |
| Cost | Lisensi BSD-3 gratis; biaya penulisan ulang seluruh konvensi dokumen SA (berbahasa Laravel) |
| Risk | Dokumen SA (ERD, seeder, state machine) harus diterjemahkan ke idiom Python → geser jadwal |

**Research Findings:**
- djangoproject.com/download: versi terbaru 6.1.1; 6.1 mainstream s/d Apr 2027, extended s/d Des 2027; LTS 5.2 extended s/d Apr 2028.

### Option B: Ruby on Rails 8.1

| Aspect | Assessment |
|--------|------------|
| Description | Framework Ruby konvensi-over-configuration, scaffold cepat, ORM relasi matang |
| Pros | Rilis sangat aktif (8.1.4 — 24 Sep 2026); konvensi mengurangi keputusan manual |
| Cons | **Ruby tidak terpasang** — bootstrap rbenv/RVM butuh unduh + kompilasi panjang (menyakitkan di koneksi VSAT); talent Ruby langka di Indonesia |
| Cost | Lisensi MIT; biaya bootstrap toolchain + kurva bahasa Ruby |
| Risk | Sistem terbengkalai saat maintainer pindah (regenerasi talent paling sulit di pasar ID) |

**Research Findings:**
- rubyonrails.org: Rails 8.1.4 — released September 24, 2026.
- Verifikasi mesin (25 Sep 2026): `ruby` tidak terdaftar di PATH.

### Option C: Spring Boot 4.1 (Java 21)

| Aspect | Assessment |
|--------|------------|
| Description | Framework enterprise Java: state machine tegas via enum, transaksi kuat, umur dukung panjang |
| Pros | Java 21 terpasang; state machine & transaksi diekspresikan paling eksplisit; umur framework enterprise terpanjang |
| Cons | **Paling berat** — baseline JVM ratusan MB + cold start berdetik-detik, boros untuk hardware site dan tidak sepadan untuk <10 user; boilerplate CRUD 3–4× lebih panjang dari Laravel |
| Cost | Lisensi OpenJDK (GPLv2+CE) + Spring (Apache-2.0) gratis; biaya waktu: volume kode terbesar untuk satu maintainer |
| Risk | Jadwal melebar; build Maven/Gradle menarik dependency internet terbanyak saat bootstrap |

**Research Findings:**
- spring.io/projects/spring-boot: versi terbaru 4.1.1.

### Option D: Express 5 + templat (Node.js)

| Aspect | Assessment |
|--------|------------|
| Description | Framework web minimal Node — fleksibel, paling ringan, tanpa opini |
| Pros | Node 24.18.1 / npm 12.0.2 terpasang; footprint paling ringan; `node_modules` self-contained setelah install |
| Cons | **Tanpa bawaan**: session, validasi, migrasi, ORM, state machine semua rakitan tangan — beban penuh di satu maintainer tanpa internet; maintenance upstream sedang bermasalah (lihat riset) |
| Cost | Lisensi MIT; biaya tersembunyi tertinggi: keputusan arsitektur manual yang tidak terdokumentasi framework |
| Risk | Kode inti proyek (transaksi stok atomik, state machine) dibangun dari nol tanpa kerangka — kesalahan logika lebih mudah terjadi dan lebih sulit ditemukan |

**Research Findings:**
- expressjs.com: versi 5.2.1; banner situs: "The Express security team is pausing from September 17 to October 6, 2026" — sinyal jeda proses rilis keamanan, relevan untuk sistem yang patch-nya jarang.

### Option E: FastAPI 0.141 (Python) + layer template

| Aspect | Assessment |
|--------|------------|
| Description | Framework API-first Python; template (Jinja2) hanya opsional sekunder |
| Pros | Performa tinggi; validasi tipe otomatis; dokumentasi OpenAPI gratis; MIT |
| Cons | **Bentuk masalah salah**: proyek butuh form POST + halaman HTML, bukan JSON API — layer SSR jadi melawan desain framework |
| Cost | Lisensi MIT; biaya membangun komponen yang diberikan Laravel/Django gratis |
| Risk | Arsitektur jadi hybrid API+SSR yang tidak pernah dibutuhkan siapa pun |

**Research Findings:**
- pypi.org/project/fastapi: versi 0.141.1 (29 Jul 2026), MIT, Python ≥3.10.

### Option F: PHP 8.3 + Laravel 12 + Blade (PILIHAN)

| Aspect | Assessment |
|--------|------------|
| Description | Monolith MVC server-rendered; Blade menghasilkan HTML penuh; Eloquent ORM; MariaDB InnoDB |
| Pros | Offline-penuh setelah sekali install; ringan-menengah untuk 1 mesin site; availability bundle (Laragon/XAMPP) & talent PHP tertinggi di ID; Eloquent menangani relasi many-to-many + state machine tanpa kerangka tambahan; seluruh dokumen SA sudah berbahasa Laravel |
| Cons | Bug-fix berakhir 13 Agu 2026 (tinggal security fix s/d 24 Feb 2027); idle footprint menengah (Apache + PHP-FPM + MariaDB); butuh internet sekali saat bootstrap |
| Cost | Lisensi MIT + PHP License — Rp 0; kontrak CON-001…CON-004 dipenuhi tanpa perubahan |
| Risk | TR-001 (siklus dukungan) — dimitigasi jalur upgrade Laravel 13 + deployment LAN-only |

**Research Findings:**
- laravel.com/docs/12.x/releases: rilis 24 Feb 2025; bug fix s/d 13 Agu 2026; security fix s/d 24 Feb 2027; dukung PHP 8.2–8.5.
- laravel.com/docs/13.x/releases: Laravel 13 rilis Q1 2026, PHP 8.3–8.5, security s/d Q1 2028 → jalur upgrade terbuka tanpa ganti bahasa.
- laravel.com/docs/12.x/database: resmi mendukung MariaDB 10.3+ (mesin: 10.11.14 ✅).
- php.net/supported-versions.php: PHP 8.3 security support s/d 31 Des 2027.
- Verifikasi mesin (25 Sep 2026): PHP 8.3.6, Composer 2.7.1, MariaDB 10.11.14, Node 24.18.1 terpasang.

## Comparison Matrix

| Criterion | Django | Rails | Spring Boot | Express | FastAPI | **Laravel 12 (dipilih)** |
|-----------|--------|-------|-------------|---------|---------|--------------------------|
| Footprint (klasifikasi kualitatif) | Ringan | Sedang | Berat | Ringan | Ringan | Sedang-ringan |
| Offline support | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Bootstrap awal di mesin dev | ⚠️ Sedang | ❌ Sulit (Ruby absen) | ⚠️ Sedang | ✅ | ✅ | ✅ Termudah |
| Kesesuaian bentuk (form CRUD relasi) | ✅ | ✅ | ✅ | ⚠️ Manual total | ❌ API-first | ✅ Terbaik |
| Manual-CRUD learning value | ⚠️ Admin menggoda | ⚠️ Scaffold | ✅ | ✅ | ✅ | ✅ |
| Availability talent ID | Tinggi | Rendah | Sedang | Sedang-tinggi | Tinggi | **Sangat tinggi** |
| Stack compatibility (dengan dokumen SA & mesin) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Maintenance status | Aktif (6.1.1) | Aktif (8.1.4) | Aktif (4.1.1) | ⚠️ Jeda rilis security | Aktif (0.141.1) | ⚠️ Security-only (TR-001) |
| License cost | Gratis | Gratis | Gratis | Gratis | Gratis | **Gratis (Rp 0)** |

## Consequences

### Positive
- Nol biaya lisensi; bootstrap hanya membutuhkan koneksi internet **sekali** di luar lokasi tambang.
- Seluruh dokumen SA (ERD, data dict, business rules, seeder) dapat diimplementasikan langsung tanpa adaptasi idiom.
- Pemeliharaan jangka panjang paling murah: talent PHP paling melimpah di pasar Indonesia.
- Runtime benar-benar offline — cocok untuk deploy LAN kantor site tanpa cloud.

### Negative
- Jendela dukungan framework lebih pendek dari Django/Rails/Spring (lihat TR-001).
- Eksposur permukaan: Apache + PHP-FPM + MariaDB dalam satu mesin — semua harus di-patch bersama.
- Klasifikasi footprint "sedang-ringan" bukan yang terkecil (Express lebih ringan) — tidak kritis di beban <10 user.

### Neutral
- Node/npm tetap dipakai, tetapi **hanya sebagai alat build saat pengembangan** — tidak ada di server site.
- MySQL tetap dipenuhi CON-002 lewat MariaDB (satu service, bukan dua).

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| TR-001: Laravel 12 hanya menerima security fix s/d 24 Feb 2027 | Tinggi | Sedang | Upgrade ke Laravel 13 (PHP 8.3 kompatibel, estimasi ≤1 hari) saat jalur update berikutnya dibuka; `APP_DEBUG=false`; deployment LAN-only |
| TR-002: PHP 8.3 security support berakhir 31 Des 2027 | Sedang | Sedang | Upgrade minor ke PHP 8.4/8.5 (diterima Laravel 12 & 13) |
| TR-003: MariaDB 10.11 Community EOL 16 Feb 2028 | Rendah | Rendah | Jendela terpanjang di stack; jalur upgrade 11.8 LTS (EOL Jun 2028) |
| TR-004: bootstrap awal gagal tanpa koneksi | Sedang | Tinggi | Selesaikan `composer install` + `npm run build` di luar lokasi; siapkan bundle offline |
| TR-005: kehilangan satu-satunya maintainer | Sedang | Sedang | Talent PHP melimpah + dokumentasi SA/arsitektur lengkap di repo |

## Compliance & Licensing

- **PHP** — PHP License v3.04: bebas penggunaan komersial, tanpa royalti.
- **Laravel, Blade, Tailwind, Vite, Node, npm, PHPUnit, Composer** — MIT: bebas didistribusikan & dimodifikasi.
- **MariaDB Server 10.11** — GPLv2: jaminan MariaDB Foundation "will remain Free and Open Source Software"; klausul share-alike aktif saat **distribusi biner MariaDB**, bukan saat menjalankan aplikasi web yang mengaksesnya → tidak memaksa pembukaan source aplikasi.
- **Total biaya lisensi: Rp 0.** Tanpa komponen komersial, tanpa tier berbayar, tanpa telemetri wajib.

## References

- https://www.php.net/supported-versions.php — PHP Supported Versions (cabang 8.2–8.5; 8.3: active s/d 31 Des 2025, security s/d 31 Des 2027)
- https://laravel.com/docs/12.x/releases — Laravel 12 Release Notes & Support Policy (rilis 24 Feb 2025; bug fix s/d 13 Agu 2026; security s/d 24 Feb 2027; PHP 8.2–8.5)
- https://laravel.com/docs/13.x/releases — Laravel 13 Release Notes (rilis Q1 2026; PHP 8.3–8.5; security s/d Q1 2028)
- https://laravel.com/docs/12.x/database — Database: Getting Started (dukungan MariaDB 10.3+, MySQL 5.7+, PostgreSQL, SQLite, SQL Server)
- https://mariadb.org/about/#maintenance-policy — MariaDB Server long-term release maintenance periods (10.11: GA 16 Feb 2023, Community/Enterprise/Extended EOL 16 Feb 2028; GPLv2 guarantee)
- https://www.djangoproject.com/download/ — Django Download (6.1.1; 6.1 mainstream s/d Apr 2027, extended s/d Des 2027; LTS 5.2 extended s/d Apr 2028)
- https://rubyonrails.org/ — Ruby on Rails (8.1.4 — released September 24, 2026)
- https://spring.io/projects/spring-boot — Spring Boot (versi terbaru 4.1.1)
- https://expressjs.com/ — Express (5.2.1; "The Express security team is pausing from September 17 to October 6, 2026")
- https://pypi.org/project/fastapi/ — FastAPI on PyPI (0.141.1, rilis 29 Jul 2026, MIT, Python ≥3.10)

## Review Schedule

- **Next Review Date:** 24 Februari 2027 (akhir security support Laravel 12) atau lebih awal bila salah satu trigger terpicu.
- **Trigger for Early Review:** (a) rencana penambahan user di luar profil <10 user; (b) kebutuhan internet stabil di lokasi; (c) kebutuhan modul baru di luar 3 modul SA; (d) CVE kritis pada Laravel 12 yang tak termitigasi.
