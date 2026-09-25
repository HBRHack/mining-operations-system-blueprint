# SRS-MASTER: Simulasi Sistem Operasional Tambang (Loading–Hauling)

> Master lintas modul — detail per fitur ada di BAB-1..BAB-6/USE-CASE.md

Aplikasi web latihan yang mensimulasikan alur operasional tambang batubara/mineral terbuka: monitoring alat berat (P2H + log shift), inventory spare part, dan maintenance Work Order yang saling terintegrasi lewat state machine alat.

## 1. System Overview

### 1.1 Purpose
- Membangun pemahaman praktis desain relasi many-to-many + state machine di Laravel (tujuan belajar).
- Menghasilkan aplikasi portofolio yang cukup realistis untuk ditunjukkan saat interview kerja bidang tambang/software (tujuan karier).
- Mensimulasikan integrasi 3 modul: saat Work Order memakai part → stok inventory berkurang otomatis; status alat berubah mengikuti state machine Work Order.

### 1.2 Scope

**In scope:**
- 4 peran pengguna (Admin, Foreman, Operator, Storekeeper) dengan login email+password.
- Master data alat berat & spare part (CRUD).
- Checklist P2H harian dengan enforcement item safety-critical.
- Log jam operasional per shift + hour meter.
- Work Order yang **auto-dibuat** saat breakdown/P2H gagal (tanpa approval manual), dengan alur status `open → menunggu_part → in_progress → closed`.
- Transaksi stok keluar-masuk dengan referensi Work Order; validasi stok tidak boleh minus.
- Dashboard: jumlah alat per status, total downtime, part paling sering dipakai, KPI safety (`trigger_type`).

**Explicitly out of scope:**
- Sambungan eksternal: GPS alat berat, fuel system, HRD/absensi, notifikasi WA/SMS/email, API pihak ketiga.
- Ekspor laporan ke Excel/PDF (v2).
- Procurement/pembelian part, faktur, accounting.
- Multi-cabang/multi-tambang, multi-tenant.
- Fitur keamanan lanjutan: 2FA, email verification, audit log formal.
- Pelaporan breakdown manual via WhatsApp/kertas (disederhanakan jadi 1 form digital).

### 1.3 Actors

| Actor | Description | Primary Interface |
|-------|-------------|-------------------|
| Admin | Kelola user, master data, "act as" semua role untuk demo; laporan penuh | Web (Blade) |
| Foreman | Kelola Work Order: klaim, ubah status, tutup WO; set prioritas | Web (Blade) |
| Operator | Isi P2H, mulai/akhiri shift (log operasional), lapor breakdown alat | Web (Blade) |
| Storekeeper | Catat transaksi stok masuk/keluar, ambil part untuk WO | Web (Blade) |
| Sistem (actor non-manusia) | Auto-buat WO saat P2H gagal/breakdown; validasi stok & state machine | Internal process |

### 1.4 Business Objectives & Success Metrics

**Objectives:**

| ID | Objective | Owner |
|----|-----------|-------|
| OBJ-001 | Menyelesaikan aplikasi 3 modul terintegrasi yang berjalan end-to-end (P2H → breakdown → WO → part → closed) | Pemilik proyek |
| OBJ-002 | Menguasai desain relasi many-to-many + state machine di Laravel melalui pembangunan nyata | Pemilik proyek |
| OBJ-003 | Menghasilkan portofolio yang bisa ditunjukkan & didemokan saat interview kerja | Pemilik proyek |

**Success Metrics:**

| Metric | Baseline (sekarang) | Target | Cara Ukur |
|--------|--------------------|--------|-----------|
| Cakupan alur inti yang berfungsi | 0% (belum ada kode) | 100% alur P2H→WO→part→closed jalan di demo | Jalankan script demo end-to-end |
| Test otomatis logika kritis | 0 test | ≥ 10 PHPUnit feature test (stok berkurang, state machine, validasi close) | `php artisan test` |
| Waktu demo penuh 3 modul | — | ≤ 10 menit tanpa error | Rekaman sesi interview simulasi |
| Jumlah commit terstruktur | 0 | ≥ 5 commit (skema, seeder, CRUD, integrasi, dashboard) | `git log` |

> Metrik ini untuk proyek latihan — bukan SLA operasional. Tidak ada baseline produksi karena sistem tidak dipakai operasi nyata.

## 2. Functional Requirements

Ringkasan per chapter — detail di REQUIREMENTS-MATRIX.md:

- BAB-1-Auth-Peran: FR-001–FR-005 (login, role, act-as, kelola user)
- BAB-2-Master-Data: FR-006–FR-012, FR-049 (CRUD alat, CRUD part, koreksi stok)
- BAB-3-Monitoring-Harian: FR-013–FR-020, FR-048 (P2H, gate safety, log shift, hour meter, daftar alat)
- BAB-4-Work-Order: FR-021–FR-034, FR-041, FR-047, FR-050 (auto-WO, state machine, prioritas, klaim, ambil part, close)
- BAB-5-Inventory: FR-035–FR-040 (transaksi stok, stok masuk, validasi minus, kartu stok, alert minimum)
- BAB-6-Dashboard: FR-042–FR-046 (agregat status, downtime, top part, KPI safety, hak laporan)

## 3. Non-Functional Requirements

Ringkasan — detail 6-bagian di `nfr.md`:
- **Performance:** render halaman < 2 dtk untuk skala 1–5 user lokal (NFR-PERF-001)
- **Security:** semua tulis wajib login + role check; stok hanya boleh diubah lewat jalur terkontrol (NFR-SEC-001)
- **Reliability:** transaksi stok + status WO + status alat atomik (satu DB transaction) (NFR-REL-001)
- **Usability:** UI form-based Bahasa Indonesia, tanpa training khusus (NFR-USAB-001)
- **Testability:** logika stok & state machine teruji PHPUnit (NFR-TEST-001)
- **Portability:** jalan lokal (Laragon/XAMPP) PHP 8.3 + MySQL/MariaDB (NFR-PORT-001)

## 4. Data Requirements

10 entitas inti: `users`, `equipment`, `p2h_checklist`, `p2h_checklist_item`, `shift`, `shift_log`, `part`, `stock_transaction`, `work_order`, `work_order_part`. Detail di `DATA-DICT-MASTER.md`, ERD di `ERD-MASTER.md`.

## 5. Integration Points

| System | Protocol | Purpose | Direction |
|--------|----------|---------|-----------|
| *(tidak ada)* | — | Sistem eksternal di luar scope: GPS, fuel, HRD, WA/SMS, API pihak ketiga | — |

Integrasi antar-modul sepenuhnya **internal** dalam satu database Laravel (stok ↔ WO ↔ status alat lewat DB transaction).

## 6. Constraints

| ID | Constraint | Tipe | Sumber |
|----|-----------|------|--------|
| CON-001 | Backend Laravel 12, PHP 8.3 | Teknis | Keputusan Aspek G |
| CON-002 | Database MySQL/MariaDB | Teknis | Keputusan Aspek G |
| CON-003 | Frontend Blade + minimal JS; HTMX hanya bila perlu; tanpa SPA berat (React/Vue full CSR) | Teknis | Task description |
| CON-004 | Tanpa Livewire/Filament/package CRUD instan — CRUD manual untuk nilai latihan | Proses | Keputusan Aspek G |
| CON-005 | Data 100% fiktif (aman UU PDP) | Regulasi | Keputusan Aspek E |
| CON-006 | UI Bahasa Indonesia, zona waktu WITA, format tanggal d/m/Y | Bisnis | Keputusan Aspek G |
| CON-007 | Development lokal Laragon/XAMPP; deploy opsional ke hosting/VPS untuk demo | Teknis | Keputusan Aspek E |

## 7. Assumptions

| ID | Assumption | Risk bila salah |
|----|-----------|------------------|
| ASM-001 | PHP ≥ 8.2 terpasang di mesin dev (diverifikasi `php -v` saat mulai coding) | Perlu turun versi Laravel |
| ASM-002 | P2H diisi maksimal 1× per alat per hari (shift pertama) | Perlu unique constraint ulang |
| ASM-003 | Satu alat maksimal 1 Work Order terbuka pada satu waktu | Perubahan state machine |
| ASM-004 | Shift log bersifat laporan (diisi operator), bukan clock-in sistem | Skema berubah jadi absensi |
| ASM-005 | "Downtime" dihitung dari `detected_at` WO sampai `closed_at` | KPI downtime salah |

## 8. Open Questions

Tidak ada — seluruh aspek A–G dari question-framework sudah terjawab dan terkunci (2026-09-25), termasuk koreksi state machine 5 state.
