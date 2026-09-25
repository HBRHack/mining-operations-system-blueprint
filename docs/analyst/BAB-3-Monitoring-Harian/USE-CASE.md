# Use Cases — BAB-3 Monitoring Harian (P2H & Log Shift)

## UC-007: Isi Checklist P2H Harian

Operator mengisi pengecekan sebelum alat dipakai; hasil menentukan alat bisa jalan atau ditolak.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Operator | operator | Memastikan alat layak jalan hari ini |

## Preconditions

- Login role `operator`
- Alat berstatus `idle` (atau `running` shift sebelumnya belum diisi — lihat Notes)
- P2H alat ini belum diisi hari ini (BR-022)

## Postconditions

- `p2h_checklist` + 10 `p2h_checklist_item` tersimpan
- Pass → alat memenuhi syarat ke running (UC-008) · Fail kritis → alat `maintenance` + WO auto (UC-009)

## Main Flow

1. **Operator** buka halaman P2H → pilih alat (DT-01)
2. **System** tampilkan 10 item: oli mesin*, coolant, tekanan ban, rem*, hidrolik*, lampu, klakson, APAR, sabuk pengaman, kaca spion (* = safety-critical)
3. **Operator** tandai ok/fail tiap item + catatan bila fail
4. **Operator** submit
5. **System** evaluasi: semua item ok → `hasil = pass`
6. **System** simpan checklist + item (snapshot nama & flag)

## Alternative Flows

### AF-1: Gagal item NON-kritis
- **Trigger:** Hanya item non-kritis yang fail
- **Step 5a:** System tetap `hasil = pass` + warning tampil di halaman (BR-024) → lanjut step 6

### AF-2: Gagal item safety-critical
- **Trigger:** ≥1 item * (oli/rem/hidrolik) berstatus fail
- **Step 5a:** **Trigger ke UC-009** (auto-WO, BR-023): 1 DB transaction: simpan checklist `hasil=fail` → `equipment=rejected` → insert WO `trigger_type=p2h_gagal` → `equipment=maintenance`
- **Step 5b:** Sistem tampilkan pesan "Alat ditolak jalan — WO-2026-XXXX dibuat otomatis"

## Exception Flows

### EF-1: P2H sudah diisi hari ini
- **Trigger:** (equipment_id, tanggal) sudah ada
- **Step X:** Tolak, arahkan ke checklist eksisting (BR-022)

### EF-2: Pembuatan WO auto gagal
- **Trigger:** DB error saat insert WO
- **Step X:** Rollback sampai `equipment = rejected` (fail-safe — **tidak** kembali idle), tampilkan error ke Foreman (BR-036 Else)

## Business Rules

- **BR-021** (gate ke running), **BR-022** (1×/hari), **BR-023** (fail kritis → rejected → WO), **BR-024** (non-kritis = warning), **BR-036** (WO auto)

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| p2h_checklist | Create | equipment_id, tanggal, hasil |
| p2h_checklist_item | Create | 10 baris snapshot |
| equipment | Update | status (rejected→maintenance) |
| work_order | Create | auto, trigger_type=p2h_gagal |

---

## UC-008: Mulai Operasi (Alat ke Running)

Operator menyalakan alat setelah P2H pass.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Operator | operator | Alat siap dijalankan |

## Preconditions

- P2H hari ini `hasil = pass` (BR-021)
- Alat berstatus `idle`

## Postconditions

- `equipment.status = running`

## Main Flow

1. **Operator** tekan "Mulai Operasi" di kartu alat
2. **System** cek P2H hari ini pass?
   - **Ya** → set `status = running`, tampilkan konfirmasi
   - **Tidak** → tolak: "Isi P2H dulu" (BR-021)

## Alternative Flows

### AF-1: P2H fail/sudah maintenance
- **Trigger:** Alat `maintenance` (WO terbuka) atau P2H gagal
- **Step 2a:** Tolak — tombol disabled + alasan

## Business Rules

- **BR-021**, **BR-040** (WO terbuka = alat tak bisa jalan)

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| equipment | Update | status → running |

---

## UC-010: Input Log Shift (Jam Operasional)

Operator mencatat jam operasi, material, tonase; hour meter bertambah otomatis.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Operator | operator | Mencatat aktivitas shift |

## Preconditions

- Login role `operator`
- Alat `running` (log selama operasi berjalan)
- Belum ada log (alat, shift, tanggal) → BR-028

## Postconditions

- Baris `shift_log` tersimpan dengan `jam_operasi` & `hour_meter_akhir` terhitung
- `equipment.hour_meter` bertambah

## Main Flow

1. **Operator** buka Log Shift → pilih alat + shift (Day/Night)
2. **Operator** isi jam_mulai, jam_selesai, material, tonase
3. **System** hitung `jam_operasi = jam_selesai − jam_mulai` (BR-025, lintas hari +24j)
4. **System** dalam 1 transaksi: insert log + `hour_meter += jam_operasi` + simpan `hour_meter_akhir` (BR-026)
5. **System** tampilkan log tersimpan + HM terbaru

## Alternative Flows

### AF-1: Idle penuh (jam_operasi = 0)
- **Trigger:** Operator catat alat tidak beroperasi
- **Step 3a:** System wajibkan `alasan_idle` (BR-027) — tanpa itu submit ditolak
- **Step 4a:** HM tidak bertambah (tambah 0)

### AF-2: Material ≠ none tapi tonase 0
- **Trigger:** Isian tidak konsisten
- **Step X:** Validasi tolak: tonase > 0 wajib bila material terisi

## Exception Flows

### EF-1: Duplikat (alat, shift, tanggal)
- **Trigger:** Log sama sudah ada
- **Step X:** Tolak + tawarkan edit log eksisting (BR-028)

## Business Rules

- **BR-025** (hitung jam), **BR-026** (HM otomatis), **BR-027** (alasan idle), **BR-028** (anti duplikat)

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| shift_log | Create | jam, material, tonase, HM akhir |
| equipment | Update | hour_meter |

---

## UC-012: Rekap Jam Operasional (Read)

Foreman/Admin melihat rekap log per alat/periode.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Foreman, Admin | foreman/admin | Memantau produktivitas |

## Preconditions

- Login foreman/admin

## Postconditions

- Tampil rekap (bukan perubahan data)

## Main Flow

1. **Foreman** buka Rekap → filter tanggal/alat
2. **System** query `shift_log` terfilter, tampilkan total jam_operasi & tonase per alat

## Business Rules

- **BR-080** (halaman laporan per role)

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| shift_log | Read | agregat per periode |
