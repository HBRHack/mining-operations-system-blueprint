# Assumption & Constraint Log: SIMTAS — Sistem Informasi Tambang (Monitoring, Inventori & Work Order)

Log asumsi yang belum terverifikasi penuh dan constraint yang membatasi ruang solusi. Sumber: SRS-MASTER §6–§7, question-framework Aspek E & G.

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 1.0 |
| **Last Updated** | 2026-09-25 |
| **Owner** | Pemilik proyek (SH-001) |

---

## ASSUMPTIONS

### ASM-001: PHP ≥ 8.2 terpasang di mesin dev

| Field | Value |
|-------|-------|
| **ID** | ASM-001 |
| **Date Made** | 2026-09-25 |
| **Made By** | Pemilik proyek |
| **Status** | Open |
| **Priority** | High |
| **Impact if Wrong** | Laravel 12 tidak bisa jalan → harus turun ke Laravel 11 (PHP 8.1) atau upgrade PHP |

**Assumption:** Mesin development sudah punya PHP ≥ 8.2 (target Laravel 12 butuh ≥ 8.2).

**Basis:** Keputusan Aspek G memilih Laravel 12 + PHP 8.3; mayoritas Laragon/XAMPP modern sudah 8.3.

**Verification Plan:** Jalankan `php -v` sebelum `composer create-project`.

**Verification Result:** Belum — belum masuk tahap coding.

**Related Requirements:** Semua FR (fondasi framework).

---

### ASM-002: P2H diisi maksimal 1× per alat per hari (shift pertama)

| Field | Value |
|-------|-------|
| **ID** | ASM-002 |
| **Date Made** | 2026-09-25 |
| **Made By** | Pemilik proyek |
| **Status** | Open |
| **Priority** | Medium |
| **Impact if Wrong** | Perlu unique constraint `(alat_id, tanggal)` diulang, atau skema jadi per-shift |

**Assumption:** Checklist P2H hanya diisi sekali per alat per hari, pada shift pertama (day shift).

**Basis:** Praktik P2H harian sebelum mulai operasi; question-framework Aspek A/B.

**Verification Plan:** Saat seeder & test — pastikan log 30 hari konsisten 1 baris P2H per alat per hari.

**Verification Result:** Belum.

**Related Requirements:** FR-013, FR-014 (P2H checklist & gate).

---

### ASM-003: Satu alat maksimal 1 Work Order terbuka

| Field | Value |
|-------|-------|
| **ID** | ASM-003 |
| **Date Made** | 2026-09-25 |
| **Made By** | Pemilik proyek |
| **Status** | Verified (dikonfirmasi koreksi state machine) |
| **Priority** | High |
| **Impact if Wrong** | Perubahan state machine — alat bisa butuh 2 jalur perbaikan paralel |

**Assumption:** Satu alat hanya punya 1 WO berstatus `open/menunggu_part/in_progress` pada satu waktu (BR-040).

**Basis:** Konfirmasi user saat koreksi state machine 2026-09-25.

**Verification Plan:** Test integrasi: auto-WO kedua untuk alat dengan WO terbuka harus ditolak.

**Verification Result:** Verified (keputusan user).

**Related Requirements:** FR-021, FR-022 (auto-WO & gate), BR-040.

---

### ASM-004: Shift log bersifat laporan, bukan clock-in

| Field | Value |
|-------|-------|
| **ID** | ASM-004 |
| **Date Made** | 2026-09-25 |
| **Made By** | Pemilik proyek |
| **Status** | Verified |
| **Priority** | Medium |
| **Impact if Wrong** | Skema berubah jadi sistem absensi (check-in/out terpisah) |

**Assumption:** Log shift diisi operator sebagai laporan jam operasional, bukan absensi kehadiran.

**Basis:** Question-framework Aspek A — scope latihan = operasional, bukan HRD.

**Verification Plan:** Review desain tabel shift_log saat migration.

**Verification Result:** Verified (keputusan user).

**Related Requirements:** FR-017, FR-018 (log shift & hour meter).

---

### ASM-005: Downtime = detected_at → closed_at

| Field | Value |
|-------|-------|
| **ID** | ASM-005 |
| **Date Made** | 2026-09-25 |
| **Made By** | Pemilik proyek |
| **Status** | Verified |
| **Priority** | Medium |
| **Impact if Wrong** | KPI downtime pada dashboard & laporan salah hitung |

**Assumption:** Downtime satu WO dihitung `closed_at − detected_at`; WO belum closed = downtime berjalan.

**Basis:** Kesepakatan koreksi state machine + BR-047.

**Verification Plan:** Test perhitungan downtime pada WO closed & WO masih open.

**Verification Result:** Verified (BR-047).

**Related Requirements:** FR-034, FR-042 (laporan downtime & dashboard).

---

## CONSTRAINTS

### CON-001: Backend Laravel 12, PHP 8.3

| Field | Value |
|-------|-------|
| **ID** | CON-001 |
| **Category** | Technology |
| **Severity** | Hard |
| **Source** | Keputusan Aspek G (question-framework) |

**Constraint:** Backend WAJIB Laravel 12 dengan PHP 8.3.

**Impact:** Menutup framework lain (CodeIgniter, Symfony, dsb).

**Workaround:** Tidak ada — ini pilihan sadar untuk nilai latihan.

**Related Requirements:** Semua FR.

---

### CON-002: Database MySQL/MariaDB

| Field | Value |
|-------|-------|
| **ID** | CON-002 |
| **Category** | Technology |
| **Severity** | Hard |
| **Source** | Keputusan Aspek G |

**Constraint:** Database MySQL atau MariaDB saja.

**Impact:** Tidak boleh pakai PostgreSQL/SQLite sebagai produksi (SQLite boleh untuk test unit).

**Workaround:** — 

**Related Requirements:** Semua FR + NFR-REL-001 (transaction DB).

---

### CON-003: Blade + minimal JS, tanpa SPA berat

| Field | Value |
|-------|-------|
| **ID** | CON-003 |
| **Category** | Technology |
| **Severity** | Hard |
| **Source** | Task description user |

**Constraint:** Frontend Blade + JS minimal; HTMX hanya bila perlu; dilarang SPA full client-side (React/Vue CSR).

**Impact:** Prioritas kecepatan render server-side; interaksi dinamis harus hemat.

**Workaround:** HTMX/Alpine ringan untuk fragment yang butuh update sebagian.

**Related Requirements:** FR-042 (NFR-PERF-001 — render < 2 detik).

---

### CON-004: Tanpa Livewire/Filament — CRUD manual

| Field | Value |
|-------|-------|
| **ID** | CON-004 |
| **Category** | Process |
| **Severity** | Hard |
| **Source** | Keputusan Aspek G |

**Constraint:** Dilarang pakai Livewire, Filament, atau package CRUD instan. Semua CRUD ditulis manual.

**Impact:** Effort lebih tinggi — tapi ini memang tujuan latihan.

**Workaround:** — 

**Related Requirements:** FR-006–FR-012, FR-049 (CRUD master).

---

### CON-005: Data 100% fiktif (aman UU PDP)

| Field | Value |
|-------|-------|
| **ID** | CON-005 |
| **Category** | Regulatory |
| **Severity** | Hard |
| **Source** | Keputusan Aspek E |

**Constraint:** Semua data (user, alat, part) fiktif — tanpa data pribadi asli.

**Impact:** Tidak boleh ada NIK, no. HP, email asli di seeder.

**Workaround:** — 

**Related Requirements:** FR-001 (user), seeder seluruhnya.

---

### CON-006: UI Bahasa Indonesia, WITA, tanggal d/m/Y

| Field | Value |
|-------|-------|
| **ID** | CON-006 |
| **Category** | Business |
| **Severity** | Hard |
| **Source** | Keputusan Aspek G |

**Constraint:** Label & pesan UI Bahasa Indonesia; zona waktu Asia/Makassar (WITA); format tanggal `d/m/Y`.

**Impact:** Localization harus konsisten; jangan ada label campur Inggris.

**Workaround:** `config/app.php` timezone + locale.

**Related Requirements:** Semua FR yang punya UI; NFR-USAB-001.

---

### CON-007: Development lokal; deploy opsional

| Field | Value |
|-------|-------|
| **ID** | CON-007 |
| **Category** | Infrastructure |
| **Severity** | Soft |
| **Source** | Keputusan Aspek E |

**Constraint:** Development di Laragon/XAMPP lokal; deploy ke hosting/VPS hanya opsional untuk demo.

**Impact:** Tidak ada dependency cloud/wajib-deploy.

**Workaround:** Bila ingin demo publik, deploy ke VPS murah — boleh dinegosiasikan (Soft).

**Related Requirements:** NFR-PORT-001 (portabilitas environment).

---

## Assumption vs Constraint Summary

| ID | Type | Statement | Status | Impact |
|----|------|-----------|--------|--------|
| ASM-001 | Assumption | PHP ≥ 8.2 di mesin dev | Open | High |
| ASM-002 | Assumption | P2H 1× per alat per hari | Open | Medium |
| ASM-003 | Assumption | Maks 1 WO terbuka per alat | Verified | High |
| ASM-004 | Assumption | Shift log = laporan, bukan absensi | Verified | Medium |
| ASM-005 | Assumption | Downtime = detected_at → closed_at | Verified | Medium |
| CON-001 | Constraint | Laravel 12 + PHP 8.3 | Hard | High |
| CON-002 | Constraint | MySQL/MariaDB | Hard | High |
| CON-003 | Constraint | Blade + minimal JS, tanpa SPA | Hard | Medium |
| CON-004 | Constraint | Tanpa Livewire/Filament, CRUD manual | Hard | Medium |
| CON-005 | Constraint | Data 100% fiktif | Hard | High |
| CON-006 | Constraint | UI ID, WITA, d/m/Y | Hard | Medium |
| CON-007 | Constraint | Dev lokal, deploy opsional | Soft | Low |

## Risk Register (derived from assumptions)

| Risk | Source | Likelihood | Impact | Mitigation |
|------|--------|------------|--------|------------|
| PHP di mesin < 8.2 → Laravel 12 gagal | ASM-001 | Low | High | Cek `php -v` di awal; fallback ke turunan versi bila perlu |
| P2H ternyata per-shift, bukan per-hari | ASM-002 | Low | Medium | Unique constraint `(alat_id, tanggal)` siap diulang jadi `(alat_id, tanggal, shift)` |
| Butuh 2 WO paralel per alat | ASM-003 | Low | High | Sudah verified — regresi dicek via test BR-040 |
| Interpretasi downtime dipersoalkan | ASM-005 | Low | Medium | BR-047 terdokumentasi & jadi test case |

## Review Schedule

| Review Date | Attendees | Purpose |
|-------------|-----------|---------|
| Awal tahap 2 (coding) | SH-001 | Verifikasi ASM-001 (`php -v`), ASM-002 (seeder) |
| Setiap selesai tahap | SH-001 | Review constraint & open assumption |

## Sign-off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Business Owner | Pemilik proyek | 2026-09-25 | (sesi review dokumen) |
| Technical Lead | Pemilik proyek | 2026-09-25 | (sesi review dokumen) |
