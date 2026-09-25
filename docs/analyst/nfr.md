# Non-Functional Requirements (NFR)

Skala: proyek latihan lokal 1–5 user. Setiap NFR ditulis 6-bagian QA scenario (Source → Stimulus → Environment → Artifact → Response → Response Measure) + tag runtime/development-time.

## NFR-PERF-001 — Kecepatan render halaman *(runtime)*

| Bagian | Isi |
|--------|-----|
| Stimulus Source | Pengguna (operator/foreman) menekan link halaman |
| Stimulus | Request GET halaman CRUD/dashboard |
| Environment | Runtime normal, skala 1–5 user, data seed 30 hari log + 15 WO |
| Artifact | Seluruh aplikasi web (Blade server-rendered) |
| Response | Halaman ter-render penuh |
| **Response Measure** | < 2 detik per halaman (p95) untuk daftar ≤ 100 baris; daftar lebih panjang pakai paginasi 25/baris |

**Refinement:** FR-042 (dashboard). Deviasi → cek query N+1 (eager loading relasi).

---

## NFR-SEC-001 — Autentikasi & otorisasi *(runtime)*

| Bagian | Isi |
|--------|-----|
| Stimulus Source | Pengguna tanpa sesi / salah role |
| Stimulus | Akses halaman atau aksi tulis di luar haknya |
| Environment | Runtime (dev lokal maupun demo online) |
| Artifact | Seluruh route web + middleware |
| Response | Ditolak dan dialihkan |
| **Response Measure** | 100% route tulis terproteksi middleware auth; akses salah role → HTTP 403; tidak ada 1 route tertulis yang bisa diakses tanpa login (diverifikasi test: hit semua route tanpa sesi → tidak ada 200 untuk halaman utama) |

**Refinement:** BR-001, BR-002, BR-005.

---

## NFR-SEC-002 — Kerahasiaan data *(runtime)*

| Bagian | Isi |
|--------|-----|
| Stimulus Source | Penyusup akses langsung ke database/dump |
| Stimulus | Membaca isi tabel users |
| Environment | Runtime |
| Artifact | Kolom `users.password` |
| Response | Password tidak terbaca |
| **Response Measure** | 0 password tersimpan plain text — semua ter-hash bcrypt/argon2 (dicek: tidak ada string password asli di DB dump) |

**Catatan:** data 100% fiktif (CON-005) jadi UU PDP tidak jadi beban compliance, tapi hash tetap wajib sebagai praktik baik + bahan cerita di portofolio.

---

## NFR-REL-001 — Integritas stok & state atomik *(runtime)*

| Bagian | Isi |
|--------|-----|
| Stimulus Source | Klik "Ambil Part" / submit P2H gagal / tutup WO |
| Stimulus | Operasi yang menyentuh ≥2 tabel (stok + WO + status alat) |
| Environment | Runtime, termasuk kondisi kegagalan (DB error di tengah operasi) |
| Artifact | Database (transaksi MySQL) |
| Response | Seluruh langkah berhasil **atau** seluruhnya batal |
| **Response Measure** | Tidak pernah ada kondisi parsial: stok berkurang tanpa baris `taken`, atau alat `maintenance` tanpa WO. Diverifikasi feature test yang memaksa kegagalan di tengah → assert tidak ada perubahan. |

**Refinement:** BR-042, BR-056, BR-023.

---

## NFR-TEST-001 — Testability *(development-time)*

| Bagian | Isi |
|--------|-----|
| Stimulus Source | Developer menjalankan `php artisan test` |
| Stimulus | Menjalankan suite test |
| Environment | Development, sebelum commit tiap tahap |
| Artifact | Logika stok, state machine, validasi close |
| Response | Test selesai memberi hijau/merah |
| **Response Measure** | ≥ 10 feature test mencakup: (1) ambil part kurangi stok tepat 1×, (2) stok kurang → menunggu_part, (3) close WO dengan part planned → ditolak, (4) P2H gagal kritis → rejected → WO auto → maintenance, (5) close WO → running/idle sesuai shift, (6) stok tidak bisa minus. Runtime test suite < 30 detik. |

---

## NFR-USAB-001 — Kemudahan pemakaian *(runtime)*

| Bagian | Isi |
|--------|-----|
| Stimulus Source | Pengguna baru (simulasi interview/demo) |
| Stimulus | Mengisi P2H dan melapor breakdown tanpa pelatihan |
| Environment | Runtime, pertama kali |
| Artifact | UI Blade (form-based, Bahasa Indonesia) |
| Response | Menyelesaikan tugas dengan benar |
| **Response Measure** | Tanpa training: isi P2H lengkap ≤ 3 menit, lapor breakdown ≤ 1 menit; label & pesan error 100% Bahasa Indonesia; tidak ada istilah jargon SA di UI (pakai kosakata GLOSSARY.md) |

---

## NFR-PORT-001 — Portabilitas environment *(development-time)*

| Bagian | Isi |
|--------|-----|
| Stimulus Source | Developer di mesin berbeda / proses deploy demo |
| Stimulus | Menjalankan aplikasi di environment baru |
| Environment | Laragon/XAMPP (PHP 8.3 + MySQL/MariaDB) lokal; opsional shared hosting/VPS |
| Artifact | Seluruh aplikasi |
| Response | Aplikasi jalan dengan konfigurasi standar |
| **Response Measure** | `composer install && php artisan migrate:fresh --seed` membuat environment kerja dalam < 10 menit; tidak ada hardcode path absolut; kredensial DB lewat `.env` |

---

## NFR-MAINT-001 — Kemudahan diri sendiri memodifikasi *(development-time)*)

| Bagian | Isi |
|--------|-----|
| Stimulus Source | Developer (user) menambah fitur baru (v2: ekspor Excel) |
| Stimulus | Menambah 1 modul/layar baru |
| Environment | Development time |
| Artifact | Kode Blade + Controller + Service |
| Response | Fitur baru ditambahkan tanpa merombak yang lama |
| **Response Measure** | Menambah 1 halaman CRUD baru ≤ 1 jam dengan mengikuti pola CRUD yang sudah ada; logika bisnis terpusat di Service class (bukan controller) agar bisa ditest |

---

## Coverage 16 kategori (checklist)

| Kategori | Status | Catatan |
|----------|--------|---------|
| performance | ✅ NFR-PERF-001 | |
| security | ✅ NFR-SEC-001/002 | |
| reliability | ✅ NFR-REL-001 | atomisitas |
| usability | ✅ NFR-USAB-001 | |
| testability | ✅ NFR-TEST-001 | development-time |
| portability | ✅ NFR-PORT-001 | |
| maintainability | ✅ NFR-MAINT-001 | |
| availability | ➖ N/A | aplikasi lokal latihan, tanpa SLA uptime |
| scalability | ➖ N/A (dibatasi CON) | skala 1–5 user sudah jadi constraint |
| compatibility | ➖ N/A | browser modern saja |
| functionality | ➖ | dicakup FR |
| certification / compliance / localization | ➖ N/A | data fiktif, tanpa regulasi (kecuali bahasa = ID) |
| SLA / extensibility | ➖ N/A | bukan sistem produksi |
