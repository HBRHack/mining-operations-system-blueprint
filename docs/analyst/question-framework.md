# Question Framework: Simulasi Sistem Operasional Tambang (Loading–Hauling)

> **Klien:** Proyek latihan belajar (aplikasi web simulasi operasional tambang batubara/mineral terbuka)
> **Konteks:** 3 modul terintegrasi — Equipment Monitoring, Inventory Spare Part, Maintenance Work Order — dengan state machine alat: `Idle → P2H OK → Running → Breakdown → WO Open → Ambil Part → WO Closed → Running/Idle`
> **Status:** TERJAWAB (asumsi solo-project, siap eksekusi ke tahap skema database) + **KOREKSI STATE MACHINE** — lihat bagian "Koreksi Terkunci" sebelum Keputusan Terkunci

---

## A. Bisnis

| Pertanyaan | Keputusan | Alasan |
|---|---|---|
| Tujuan utama | Latihan + portofolio (dua-duanya) | Nilai di proses desain, hasil akhir juga bisa ditunjukkan pas interview kerja tambang |
| Masalah yang mau dipahami | Desain relasi many-to-many + state machine di Laravel | Skill yang langsung relevan ke tugas beneran nanti |
| Alur laporan breakdown | Operator input langsung ke sistem (skip WA/manual) | Simulasi digital, alur manual di real world disederhanakan jadi 1 form |
| Siapa ubah status alat | Operator set Running (dari P2H), Teknisi/Foreman set Maintenance/Closed | Realistis + melatih role-based logic |
| P2H gagal → alat dikunci? | Ya, kalau item safety-critical (rem, ban, hidrolik) gagal → status otomatis "Ditolak Jalan" | Business rule enforcement, bagian menarik buat ditunjukkan |
| Jumlah shift | 2 shift: Day 06:00–18:00, Night 18:00–06:00 | Paling umum di tambang, lebih simpel dari 3 shift |
| Downtime "penting" | > 30 menit wajib WO, di bawah itu cukup dicatat sebagai idle biasa | Batas realistis tanpa logika ribet |
| PALM goal | Personal (lamaran kerja) + growth (belajar) | QA relevan: test otomatis logika stok sebagai bukti kualitas kode |

## B. Data

| Pertanyaan | Keputusan |
|---|---|
| Format unit_id | `EX-01`, `DT-01`, `DZ-01`, `WT-01` (Excavator/Dump Truck/Dozer/Water Truck) |
| Tipe alat master data | Excavator (PC200/PC2000), Dump Truck (40 ton), Bulldozer (D85), Water Truck (20 kL) |
| Hour meter | Ya, pakai HM — bertambah tiap shift sesuai jam running |
| Kolom log shift | jam_mulai, jam_selesai, material (OB/coal/ore), tonase, alasan_idle |
| Item P2H | 10 item: oli mesin, coolant, tekanan ban, rem, hidrolik, lampu, klakson, APAR, sabuk pengaman, kaca spion. Safety-critical: oli, rem, hidrolik |
| Kode part | `ENG-OIL-15W40`, `HDL-HOSE-001` dst. Kategori: engine/hidrolik/electrical/undercarriage/tire |
| Satuan | pcs/liter/set/unit. Stok tidak boleh minus (kurang → WO status "Menunggu Part") |
| Format WO | `WO-2026-0001`, prioritas low/medium/high di-set Foreman |
| Arsip vs permanen | Semua permanen dulu (data dummy kecil, tidak perlu arsip) |

## C. Pengguna

- **Peran**: Admin, Foreman, Operator, Storekeeper (4 role, Supervisor digabung ke Admin)
- **Beda hak**: Foreman buka/tutup WO; Storekeeper catat transaksi part
- **Satu orang banyak peran**: Ya, Admin bisa "act as" semua role buat demo cepat
- **Operator lihat data**: Semua alat (biar dashboard kelihatan ramai pas demo)
- **Login**: Perlu, tapi simpel (email+password, tanpa 2FA/email verification)
- **11 peran generik**: relevan hanya operator, teknisi/foreman, storekeeper, dan pemilik proyek (analis+implementer+tester). Customer, regulator, sponsor, supplier — tidak relevan (simulasi internal, tanpa procurement/transaksi eksternal)

## D. Sambungan ke Sistem Lain

- **3 aturan integrasi inti** (dikonfirmasi semua):
  1. Saat WO pakai part → stok inventory berkurang otomatis
  2. Status alat mengikuti state machine WO (Breakdown saat WO dibuka, kembali Running/Idle saat WO ditutup)
  3. Saat WO closed → alat balik ke **Running** kalau masih ada sisa shift, atau **Idle** kalau shift sudah habis — ditentukan sistem otomatis dari jam saat ini vs jadwal shift, bukan dipilih manual
- **Sambungan eksternal**: Tidak ada (tanpa GPS, fuel system, HRD, notifikasi WA/SMS) — di luar scope simulasi, fokus tetap 3 modul inti
- **Stok part kurang saat WO minta lebih**: Sistem tolak ambil sebagian, WO masuk status "Menunggu Part" (backorder), tidak boleh stok minus
- **Interface 6W (WO ambil part)**:
  - Siapa: Storekeeper/Foreman trigger dari form WO
  - Apa/volume: 1 transaksi per pengambilan part, estimasi ~50–100 transaksi/bulan skala simulasi
  - Kapan: otomatis saat klik tombol "Ambil Part" di halaman WO
  - Di mana: halaman detail Work Order
  - Kenapa: mengurangi stok real-time dan mencatat histori pemakaian part per alat
  - Bagaimana: satu tombol "Ambil Part" → pilih part + qty → validasi stok → commit transaksi
- **WO ditutup tapi part belum tercatat keluar**: Sistem **wajib** validasi — tidak bisa close WO kalau ada part yang direncanakan tapi belum dicatat terpakai (mencegah data stok tidak akurat)

## E. Kebutuhan Teknis

| Pertanyaan | Keputusan |
|---|---|
| Pengguna bersamaan | 1–5 user (skala demo/lokal, bukan produksi) |
| Skala data dummy | 5 unit alat, 15 jenis part, 2 shift × 30 hari (~60 baris log shift), 10–15 WO histori |
| Offline vs online | Lokal (Laragon/XAMPP) untuk development, opsional deploy ke hosting murah/VPS untuk demo interview |
| Data pribadi | 100% fiktif, tidak ada nama/NIK asli — aman dari UU PDP |
| Keamanan minimal | Login + role-based access (tanpa fitur keamanan lanjutan seperti 2FA) |
| Data awal (biar dashboard terisi) | Minimal 30 hari log shift + 10 WO selesai + riwayat pemakaian part sejak seeding |
| Kalau logika stok salah | Reset database + seed ulang (`php artisan migrate:fresh --seed`), tidak perlu tombol reset di UI untuk v1 |

## F. Kelompok Fitur

- **Kelompok fitur**: Master Data (Alat, Part, User/Peran) · P2H Checklist · Log Shift/Jam Operasional · Work Order · Inventory Transaksi · Dashboard & Laporan
- **Wajib v1**: Semua di atas kecuali laporan ekspor Excel (boleh menyusul di v2)
- **Urutan pengerjaan**: Skema DB → Seeder → CRUD 3 modul → Logika integrasi (stok + state machine) → Dashboard
- **Relasi paling riskan**: Part terpakai tercatat dua kali kalau WO dibuka lalu diedit berulang. Aturan: transaksi stok dicatat sekali per klik "Ambil Part", bukan otomatis tiap kali form WO disimpan — perlu test khusus untuk ini

## G. Tech Stack

| Pertanyaan | Keputusan |
|---|---|
| PHP & Laravel | PHP 8.3 + Laravel 12 (cek versi terpasang dengan `php -v` sebelum mulai) |
| Database | MySQL (via Laragon/XAMPP, MariaDB juga kompatibel) |
| HTMX | Mulai murni Blade form + redirect dulu; tambah HTMX hanya untuk 2–3 interaksi dinamis (misal dropdown alat→part) kalau memang dibutuhkan, bukan dari awal |
| Autentikasi | Laravel Breeze (Blade + Alpine) — ringan, cukup, dan tetap terlihat profesional |
| Package tambahan | Tidak pakai Livewire/Filament — CRUD Blade manual justru nilai latihan lebih besar dan sejalan dengan prinsip "ringan" |
| Deploy/environment | Lokal Laragon untuk development; VPS murah/shared hosting kalau perlu demo online ke HRD |
| Testing | Ya, PHPUnit feature test untuk logika "pakai part → stok berkurang" dan "state machine alat" — bagian paling gampang rusak, juga paling bagus ditunjukkan di portofolio |
| Git/GitHub | Ya, `git init` di awal, commit per tahap (skema, seeder, CRUD, integrasi, dashboard) |
| Konvensi | UI Bahasa Indonesia, zona waktu WITA, format tanggal `d/m/Y` |

---

## Koreksi Terkunci — State Machine 5 State (menggantikan bagian A & D)

**Koreksi dari user (2026-09-25):** `p2h_ok` BUKAN state tersimpan — hanya hasil validasi sesaat (pass/fail). `rejected` dan `breakdown` adalah state transisi singkat: WO auto-dibuat tanpa approval manual, status langsung pindah ke `maintenance`. Asal kerusakan dicatat di `work_orders.trigger_type`, bukan di status alat.

**Enum final `equipments.status` (5):** `idle | running | rejected | breakdown | maintenance`

```
idle ──(P2H)──┬─ lolos ──────► running ──(breakdown lapangan)──► breakdown ─┐
              │                                                             │
              └─ gagal safety-critical ──► rejected ─────────────────────────┤
                                                                             ▼
                                                              WO auto-dibuat (same transaction)
                                                              trigger_type = p2h_gagal | breakdown_lapangan
                                                                             ▼
                                                                      maintenance
                                                                             │
                                                              (part terpakai, WO closed)
                                                                             ▼
                                                        running (sisa shift) / idle (shift habis)
```

| Keputusan | Jawaban |
|---|---|
| `rejected`/`breakdown` tetap di enum? | **Ya** — fail-safe: kalau auto-pembuatan WO gagal, alat tetap di state transisi, tidak mungkin balik `idle`/`running` tanpa tindakan |
| Mapping WO → status alat | `maintenance` membawahi seluruh fase WO (`open` → `menunggu_part` → `in_progress` → `closed`) |
| Jumlah WO terbuka per alat | **Maks 1** — selama `maintenance` alat tidak bisa running → tidak bisa breakdown kedua |
| Kalau auto-WO gagal dibuat | Alat stuck `rejected`/`breakdown` + error ke Foreman; hanya Foreman/Admin yang bisa retry/buka WO manual |
| WO auto default | prioritas `high`, status `open`, assignee kosong (Foreman klaim sendiri) |
| KPI safety | `COUNT(WO WHERE trigger_type='p2h_gagal')` vs `COUNT(WO WHERE trigger_type='breakdown_lapangan')` — satu tabel, dua KPI |

## Keputusan Terkunci

| Aspek | Keputusan | Tanggal |
|-------|-----------|---------|
| A. Tujuan & shift | Latihan + portofolio; 2 shift (Day 06:00–18:00 / Night 18:00–06:00) | 2026-09-25 |
| A+. State machine | **5 state** (`p2h_ok` bukan state); `rejected`/`breakdown` transisi singkat → WO auto; asal di `trigger_type` | 2026-09-25 |
| B. Format ID & kolom log | `EX-01`/`DT-01`/`DZ-01`/`WT-01`; log shift dengan HM, tonase, material | 2026-09-25 |
| C. Peran & autentikasi | Admin, Foreman, Operator, Storekeeper; login sederhana (email+password) | 2026-09-25 |
| D. Aturan integrasi (stok + state machine) | Stok berkurang otomatis saat "Ambil Part"; status alat ikut state machine WO; validasi close WO | 2026-09-25 |
| E. Skala data & environment | 5 alat, 15 part, 30 hari log dummy; lokal Laragon | 2026-09-25 |
| F. Urutan pengerjaan | Skema DB → Seeder → CRUD → Integrasi → Dashboard | 2026-09-25 |
| G. Tech stack final | Laravel 12 + PHP 8.3 + MySQL + Breeze, tanpa Livewire/Filament | 2026-09-25 |

---

> **Langkah selanjutnya:** Skema database (migration Laravel) dieksekusi berdasarkan keputusan Aspek B (format ID + kolom log) dan Aspek G (Laravel 12/PHP 8.3/MySQL).