# RACI Matrix: SIMTAS — Sistem Informasi Tambang (Monitoring, Inventori & Work Order)

Proyek latihan satu orang, sehingga sebagian besar baris diakhiri `R/A` pada Dev Lead — tetapi matriks tetap ditulis utuh agar pembagian peran jelas dan bisa di-scale bila ada orang kedua (mis. penguji/reviewer).

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 1.0 |
| **Last Updated** | 2026-09-25 |
| **Owner** | Pemilik proyek (SH-001) |

## RACI Legend

| Letter | Role | Definition |
|--------|------|------------|
| **R** | Responsible | Does the work. The person who completes the task. |
| **A** | Accountable | Owns the work. Signs off. Only ONE per task. |
| **C** | Consulted | Provides input. Two-way communication. |
| **I** | Informed | Kept updated. One-way communication. |

## Stakeholder Directory

| ID | Name | Role/Title | Department | Contact |
|----|------|------------|------------|---------|
| SH-001 | Pemilik proyek | Sponsor + SA + Dev Lead + QA Lead | Proyek latihan | (lokal) |
| SH-002 | Admin | Administrator / Supervisor | Operasional (simulasi) | (in-app) |
| SH-003 | Foreman | Mandor / penyelia lapangan | Operasional (simulasi) | (in-app) |
| SH-004 | Operator | Pengoperasi alat | Lapangan (simulasi) | (in-app) |
| SH-005 | Storekeeper | Penjaga gudang spare part | Logistik (simulasi) | (in-app) |

---

## RACI by Requirement Area

### BAB-1 — Auth & Peran

| Requirement | SA (SH-001) | Admin (SH-002) | Foreman (SH-003) | Operator (SH-004) | Storekeeper (SH-005) |
|-------------|-------------|----------------|------------------|-------------------|----------------------|
| FR-001–FR-005: login, 4 role, act-as | R/A | C | C | C | C |

### BAB-2 — Master Data

| Requirement | SA (SH-001) | Admin (SH-002) | Foreman (SH-003) | Operator (SH-004) | Storekeeper (SH-005) |
|-------------|-------------|----------------|------------------|-------------------|----------------------|
| FR-006–FR-012, FR-049: CRUD alat/part, koreksi stok | R | C | I | I | C |

### BAB-3 — Monitoring Harian (P2H & Log Shift)

| Requirement | SA (SH-001) | Admin (SH-002) | Foreman (SH-003) | Operator (SH-004) | Storekeeper (SH-005) |
|-------------|-------------|----------------|------------------|-------------------|----------------------|
| FR-013–FR-020, FR-048: P2H, gate safety, log shift | R | I | C (gate) | R (isi) | I |

### BAB-4 — Work Order

| Requirement | SA (SH-001) | Admin (SH-002) | Foreman (SH-003) | Operator (SH-004) | Storekeeper (SH-005) |
|-------------|-------------|----------------|------------------|-------------------|----------------------|
| FR-021–FR-034, FR-041, FR-047, FR-050: auto-WO, state machine, klaim, ambil part, close | R | I | C | I | C |

### BAB-5 — Inventory

| Requirement | SA (SH-001) | Admin (SH-002) | Foreman (SH-003) | Operator (SH-004) | Storekeeper (SH-005) |
|-------------|-------------|----------------|------------------|-------------------|----------------------|
| FR-035–FR-040, FR-045, FR-046: stok masuk, ambil part, kartu stok, alert | R | C | I | I | C |

### BAB-6 — Dashboard & Laporan

| Requirement | SA (SH-001) | Admin (SH-002) | Foreman (SH-003) | Operator (SH-004) | Storekeeper (SH-005) |
|-------------|-------------|----------------|------------------|-------------------|----------------------|
| FR-042–FR-044: dashboard read-only & laporan | R | I | I | I | I |

---

## RACI by Activity

| Activity | SA (SH-001) | Dev Lead (SH-001) | QA Lead (SH-001) | User UAT (SH-002…005) |
|----------|-------------|-------------------|------------------|------------------------|
| Eliciting requirement (grill session) | R/A | I | I | C |
| Dokumentasi SA (SRS/ERD/BR/matrix) | R/A | I | C | I |
| Validation Gate dokumen | R/A | I | C | I |
| Setup project (Laravel, migration) | C | R/A | I | I |
| Seeder realistis | C | R/A | C | C (validasi data masuk akal) |
| CRUD manual | C | R/A | C | I |
| Logika integrasi + state machine | R/A | R | C | C |
| Unit/integration test (≥10) | C | C | R/A | I |
| Dashboard & render performance | C | R/A | C | I |
| UAT peran | C | C | C | R, **A** = SH-001 |
| Go/no-go tiap tahap coding | R | R | C | C, **A** = SH-001 |

---

## RACI by Decision Type

| Decision | SA (SH-001) | Dev Lead (SH-001) | User (SH-002…005) | Escalation Path |
|----------|-------------|-------------------|-------------------|-----------------|
| Functional requirement (FR/BR) | R/A | C | C | SA → (kembali ke user bila bentrok) |
| State machine & aturan transisi | R/A | C | C (Foreman/Operator validasi alur) | SA → user lapangan |
| Skema database | R | R/A | I | SA → Dev Lead |
| UI label & alur form (Bahasa Indonesia) | C | R/A | C | Dev Lead → SA |
| Aturan stok & atomisasi transaksi | R/A | C | C (Storekeeper) | SA → user gudang |
| Scope change (tambah/kurang fitur) | R | C | C | SA → pemilik proyek (A) |

---

## Conflict Resolution

| Scenario | Resolution |
|----------|------------|
| Multiple people claim "A" | Only ONE person can be A. Pemilik proyek decides. |
| R and A are the same person | Acceptable — proyek satu orang. Document it. |
| Missing R (nobody to do it) | Resource gap — pemilik proyek menambah effort / menunda scope. |
| Missing A (nobody accountable) | Critical gap — must be filled before sign-off. |
| Stakeholder not listed | Add to `stakeholder-register.md` and assign RACI. |

## Sign-off Authority Matrix

| Requirement Level | Minimum Approver | Sign-off Format |
|-------------------|------------------|-----------------|
| Master doc (SRS/ERD/BR) | SH-001 | Komentar "approved" pada sesi review |
| FR baru / scope change | SH-001 + catatan di `change-management.md` | Entri CHG-XXX |
| Tahap coding selesai | SH-001 + test hijau | Commit + laporan test |
| UAT per role | Role terkait + SH-001 | Checklist UAT |

## Review Schedule

| Review Date | Attendees | Purpose |
|-------------|-----------|---------|
| 2026-09-25 | SH-001 | Initial RACI assignment (awal dokumen SA) |
| Saat review dokumen SA | SH-001 + user (role peran) | Review & koreksi requirement |
| Sebelum tiap tahap coding ditutup | SH-001 | Pre-sign-off review + test |

## Sign-off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Sponsor / SA | Pemilik proyek | 2026-09-25 | (sesi review dokumen) |
