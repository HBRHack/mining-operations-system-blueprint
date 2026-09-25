# Change Management: SIMTAS — Sistem Informasi Tambang (Monitoring, Inventori & Work Order)

Versioning & change tracking requirement selama fase analisis. Mencatat evolusi requirement: apa yang berubah, kenapa, siapa menyetujui, kapan.

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 1.1 |
| **Last Updated** | 2026-09-25 |
| **Baseline** | v1.0 — dokumen SA Tahap 1, disusun 2026-09-25 |
| **Change Authority** | Pemilik proyek (SH-001) — proyek satu orang, semua level approval di tangan yang sama |

## Change Policy

| Threshold | Approval Required |
|-----------|-------------------|
| Minor (typo, klarifikasi) | SA boleh approve sendiri |
| Moderate (tambah/hapus requirement non-kritis) | SA + konfirmasi user (sesi grill) |
| Major (scope change, modul baru, perubahan arsitektur/state machine) | Diskusi eksplisit dengan user sebelum diterapkan |

---

## Change Log

### CHG-001: Koreksi state machine jadi 5 state

| Field | Value |
|-------|-------|
| **Change ID** | CHG-001 |
| **Date Proposed** | 2026-09-25 |
| **Date Approved** | 2026-09-25 |
| **Requested By** | User (koreksi eksplisit) |
| **Approved By** | User |
| **Status** | Implemented |
| **Impact Level** | Major |

**Affected Requirements:**
- FR-021, FR-022, FR-023 (state machine & transisi)
- UC-007, UC-009 (P2H gate & auto-WO)
- BR-023, BR-036, BR-039, BR-040
- PF-003 (flow state machine)

**Reason for Change:**
Draf awal memuat 6 state termasuk `p2h_ok`. User menegaskan state machine cuma 5 state: `idle | running | rejected | breakdown | maintenance` — `p2h_ok` bukan state, hanya nilai `p2h_checklist.hasil` (pass/fail).

**Before:** 6 state (`idle, p2h_ok, running, rejected, breakdown, maintenance`).

**After:** 5 state (`idle, running, rejected, breakdown, maintenance`); hasil P2H hanya di tabel checklist; asal kerusakan pindah ke `work_orders.trigger_type` (`p2h_gagal` / `breakdown_lapangan`) sehingga 2 KPI berasal dari 1 tabel.

**Impact Assessment:**

| Impact Area | Details |
|-------------|---------|
| Scope | Tidak berubah — perbaikan konseptual |
| Timeline | Tidak ada dampak (masih fase dokumen) |
| Budget | — |
| Affected Modules | BAB-3 (P2H), BAB-4 (WO) |
| Risk | State tambahan yang membingungkan dihilangkan; risiko tersisa = `rejected`/`breakdown` sebagai state singkat tetap dipertahankan (fail-safe bila WO gagal dibuat) |

**Decision:**
Diterima penuh. Turunan keputusan ikut dikunci: `rejected`/`breakdown` tetap di enum (fail-safe), maks 1 WO terbuka per alat (BR-040), WO auto default `open`/`high`/`assigned_to=null`, close WO → alat `running` (shift masih jalan) atau `idle` (shift habis) otomatis dari jam vs jadwal shift.

---

### CHG-002: Renumber FR-012b → FR-049, FR-027b → FR-050

| Field | Value |
|-------|-------|
| **Change ID** | CHG-002 |
| **Date Proposed** | 2026-09-25 |
| **Date Approved** | 2026-09-25 |
| **Requested By** | Audit internal (self-review dokumen) |
| **Approved By** | Pemilik proyek |
| **Status** | Implemented |
| **Impact Level** | Minor |

**Affected Requirements:**
- FR-012b → **FR-049** (Admin/storekeeper CRUD master part)
- FR-027b → **FR-050** (Foreman kelola prioritas & klaim WO)
- Rujukan di SRS-MASTER §2 (range per chapter)

**Reason for Change:**
Nomor tempelan `…b` tidak proper untuk dokumen yang jadi satu-satunya sumber traceability. Nomor FR-049/FR-050 sebelumnya kosong.

**Before:** `FR-012b`, `FR-027b` (suffix tempelan).

**After:** `FR-049`, `FR-050` (nomor unik penuh, total 50 FR: FR-001…FR-050).

**Impact Assessment:**

| Impact Area | Details |
|-------------|---------|
| Scope | Tidak berubah — penataan nomor |
| Timeline | Tidak ada dampak |
| Budget | — |
| Affected Modules | BAB-2, BAB-4 (dokumen saja) |
| Risk | Rujukan lama yang belum ter-update — mitigasi: grep `FR-012b|FR-027b` = 0 hasil |

**Decision:**
Diterima — tanpa dampak kode, hanya kerapian dokumen.

---

### CHG-003: Perbaikan rujukan UC-020 & penambahan UC-020

| Field | Value |
|-------|-------|
| **Change ID** | CHG-003 |
| **Date Proposed** | 2026-09-25 |
| **Date Approved** | 2026-09-25 |
| **Requested By** | Audit internal (traceability check) |
| **Approved By** | Pemilik proyek |
| **Status** | Implemented |
| **Impact Level** | Moderate |

**Affected Requirements:**
- UC-020 (baru — didefinisikan di BAB-5)
- BR-047 (rusak: merujuk UC-020 yang tidak ada)
- BR-060 (rusak: merujuk UC-020 yang belum didefinisikan + FR-039 yang tidak relevan)

**Reason for Change:**
Validation check menemukan `UC-020` dirujuk BR-047 & BR-060 tapi tidak pernah didefinisikan (rujukan mati), plus FR-012b/FR-027b nomor tempelan.

**Before:**
- BR-047 → `Related: UC-012, UC-020 · FR-034, FR-041` (UC-020 yatim)
- BR-060 → `Related: UC-020, UC-016 · FR-039` (UC-020 yatim, FR-039 salah sasaran)

**After:**
- BR-047 → `Related: UC-012, UC-018 · FR-034, FR-041` (UC-018 = Laporan Downtime, rujukan benar)
- BR-060 → `Related: UC-016, UC-020 · FR-037, FR-039` (FR-037 badge stok menipis = sumber BR-060)
- UC-020 "Lihat Daftar Part Stok Menipis" didefinisikan di BAB-5-Inventory/USE-CASE.md

**Impact Assessment:**

| Impact Area | Details |
|-------------|---------|
| Scope | Bertambah 1 UC (read-only, sudah jadi kebutuhan lewat FR-037) |
| Timeline | Tidak ada dampak |
| Budget | — |
| Affected Modules | BAB-5 (1 UC baru), business-rules (2 baris Related) |
| Risk | Rendah — UC baru hanya membaca filter `stok ≤ min_stok` |

**Decision:**
Definisi, bukan penghapusan — fitur daftar stok menipis memang nyata (FR-037) dan sayang dibuang.

---

## Version History

| Version | Date | Author | Changes | Status |
|---------|------|--------|---------|--------|
| 1.0 | 2026-09-25 | Pemilik proyek | Initial baseline — 50 FR, 43 BR, 21 UC, 8 NFR | Approved (Validation Gate §5c lolos) |
| 1.1 | 2026-09-25 | Pemilik proyek | CHG-001 (state machine 5 state), CHG-002 (renumber FR), CHG-003 (UC-020 + file wajib) | Under Review (menunggu review user) |

## Change Statistics

| Metric | Count |
|--------|-------|
| Total Changes | 3 |
| Approved | 3 |
| Rejected | 0 |
| Pending | 0 |
| By Impact: Minor | 1 |
| By Impact: Moderate | 1 |
| By Impact: Major | 1 |

## Requirement Stability Index

| Requirement | Versions Changed | Last Change | Stability |
|-------------|------------------|-------------|-----------|
| FR-021–FR-023 (state machine) | 1× (CHG-001) | 2026-09-25 | Stable (baru 1 perubahan, dampak besar) |
| FR-049 (CRUD part) | 1× (CHG-002, penomoran saja) | 2026-09-25 | Stable |
| FR-050 (kelola WO) | 1× (CHG-002, penomoran saja) | 2026-09-25 | Stable |
| FR-037 (badge stok menipis) | 1× (CHG-003, rujukan UC) | 2026-09-25 | Stable |

Tidak ada requirement yang berubah >3× — belum ada yang perlu di-flag volatile.

## Traceability to Approval

| Requirement | Version | Approved By | Date | Sign-off Reference |
|-------------|---------|-------------|------|-------------------|
| Seluruh FR (baseline) | 1.0 | Pemilik proyek | 2026-09-25 | Validation Gate §5c |
| FR-021–FR-023 | 1.1 | User | 2026-09-25 | Koreksi eksplisit state machine (CHG-001) |
| FR-049, FR-050 | 1.1 | Pemilik proyek | 2026-09-25 | CHG-002 |
| FR-037 + UC-020 | 1.1 | Pemilik proyek | 2026-09-25 | CHG-003 |
