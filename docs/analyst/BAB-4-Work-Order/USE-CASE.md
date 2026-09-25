# Use Cases — BAB-4 Work Order

## UC-009: Auto-Pembuatan Work Order (System)

WO dibuat otomatis oleh sistem — tanpa approval manual — saat P2H gagal kritis atau laporan breakdown masuk.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Sistem | actor non-manusia | Mengubah temuan jadi tindakan perbaikan |
| Operator (pemicu) | operator | Melaporkan kerusakan lapangan |

## Preconditions

- P2H gagal safety-critical (dari UC-007) ATAU operator submit laporan breakdown (alat `running`)
- Alat tidak punya WO terbuka lain (BR-040)

## Postconditions

- Baris `work_order` tersimpan: `status=open`, `prioritas=high`, `assigned_to=null`, `detected_at=now()`, `trigger_type` terisi
- `equipment.status = maintenance` (transisi rejected/breakdown hanya hidup dalam transaksi yang sama)

## Main Flow

1. **System** menerima pemicu (submit P2H fail / submit laporan breakdown)
2. **System** cek alat punya WO terbuka? → **ya: tolak + pesan "WO-XXX masih terbuka"** (BR-040)
3. **System** generate `wo_code` (WO-YYYY-NNNN), isi `trigger_type` sesuai pemicu (BR-039)
4. **System** dalam SATU DB transaction: insert WO → set `equipment.status = maintenance` (BR-036, BR-038)
5. **System** tampilkan notifikasi ke Foreman: "WO-2026-0001 dibuat otomatis (prioritas High)"

## Alternative Flows

### AF-1: Breakdown < 30 menit
- **Trigger:** Operator memilih "catat sebagai idle" untuk gangguan singkat
- **Step X:** Tidak ada WO — operator cukup isi log shift dengan `alasan_idle` (FR-041, BR-047)

## Exception Flows

### EF-1: Insert WO gagal di tengah transaksi
- **Trigger:** DB error
- **Step X:** Rollback → alat tetap `rejected`/`breakdown` (**fail-safe**, tidak pernah kembali idle/running) + error tampil ke Foreman; Foreman/Admin bisa retry atau buka WO manual (BR-036 Else)

## Business Rules

- **BR-036** (auto tanpa approval), **BR-039** (trigger_type), **BR-040** (maks 1 terbuka), **BR-047** (aturan 30 menit), **BR-023**

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| work_order | Create | auto default open/high |
| equipment | Update | status → maintenance |

---

## UC-011: Klaim & Kelola WO (Board Foreman)

Foreman melihat papan WO aktif, mengklaim, mengatur prioritas & rencana part.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Foreman | foreman | Menugaskan & memprioritaskan perbaikan |

## Preconditions

- Login `foreman`/`admin` (BR-046)
- WO sudah terbuka (UC-009)

## Postconditions

- `assigned_to` terisi / `prioritas` berubah / baris `work_order_part` (planned) bertambah

## Main Flow

1. **Foreman** buka Board WO → daftar WO ≠ closed (filter status/prioritas)
2. **Foreman** buka WO → "Klaim" → `assigned_to = dirinya`
3. **Foreman** set prioritas (low/medium/high) bila perlu (default high dari auto)
4. **Foreman** tambah rencana part: pilih part + qty → baris `work_order_part(status=planned)` tersimpan
5. **System** evaluasi status: ada planned dengan qty > stok → `status = menunggu_part` (BR-043); semua tersedia → `in_progress` (BR-037); tanpa part → `in_progress`
6. **System** `part.stok` **tidak berubah** (BR-041 — rencana ≠ transaksi)

## Alternative Flows

### AF-1: Stok berubah setelah rencana
- **Trigger:** Part terpakai WO lain
- **Step X:** Evaluasi ulang → WO bisa kembali `menunggu_part`

## Exception Flows

### EF-1: Non-foreman mencoba klaim
- **Trigger:** Operator buka aksi klaim
- **Step X:** 403 (BR-046, BR-002)

## Business Rules

- **BR-037** (transisi tertutup), **BR-041** (rencana ≠ transaksi), **BR-043** (backorder), **BR-046** (hak foreman)

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| work_order | Update | assigned_to, prioritas, status |
| work_order_part | Create | planned rows |
| part | Read | cek stok (tanpa perubahan) |

---

## UC-013: Ambil Part dari Inventory (untuk WO)

Storekeeper mengambil part yang direncanakan — satu-satunya jalur stok keluar ber-Referensi WO.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Storekeeper | storekeeper | Mengeluarkan part fisik & mencatatnya |

## Preconditions

- WO punya `work_order_part(status=planned)`
- Role `storekeeper`/`foreman`/`admin` (BR-062)

## Postconditions

- `stock_transaction(out)` tersimpan → `part.stok` berkurang tepat 1× → `work_order_part.status=taken` + `stock_transaction_id` terisi
- Status WO dievaluasi (BR-037): semua taken → `in_progress`; sisa yang tak tersedia → `menunggu_part`

## Main Flow

1. **Storekeeper** buka detail WO → daftar part rencana + stok tersedia
2. **Storekeeper** tekan "Ambil Part" pada baris part (qty dari rencana)
3. **System** dalam SATU DB transaction (BR-042):
   a. Validasi `stok ≥ qty` (BR-056)
   b. Insert `stock_transaction(tipe=out, work_order_id)`
   c. `part.stok -= qty`
   d. Tandai `work_order_part` → `taken` + isi `stock_transaction_id`
   e. Evaluasi status WO (BR-037)
4. **System** commit → tampilkan stok baru & sisa rencana

## Alternative Flows

### AF-1: Ambil semua sisa sekaligus
- **Trigger:** Beberapa baris planned
- **Step X:** Boleh per baris; atomisitas per baris (tiap klik = 1 transaksi)

## Exception Flows

### EF-1: Stok kurang
- **Trigger:** `qty > stok`
- **Step X:** Seluruh transaksi **rollback**; pesan "stok tidak cukup"; WO `menunggu_part` (BR-043)

### EF-2: Kegagalan di tengah (DB error)
- **Trigger:** Error pada langkah c–e
- **Step X:** Rollback total — stok tak berkurang, baris tetap `planned` (NFR-REL-001)

## Business Rules

- **BR-042** (atomik sekali jalan), **BR-056** (stok ≥ 0), **BR-062** (jalur resmi), **BR-043**, **BR-037**

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| stock_transaction | Create | out + WO ref |
| part | Update | stok berkurang |
| work_order_part | Update | planned → taken |
| work_order | Update | status (evaluasi) |

---

## UC-014: Tutup Work Order

Foreman menutup WO setelah perbaikan — memicu alat kembali running/idle.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Foreman | foreman | Mengakhiri perbaikan & mengembalikan alat |

## Preconditions

- WO `in_progress` (semua part rencana `taken`, BR-044)
- Role `foreman`/`admin` (BR-046)

## Postconditions

- `work_order.status = closed`, `closed_at` terisi
- `equipment.status` = `running` (masih shift) ATAU `idle` (di luar shift) — otomatis (BR-045)

## Main Flow

1. **Foreman** buka WO `in_progress` → "Tutup WO"
2. **System** cek semua `work_order_part.status = taken`? (BR-044)
   - **Ya** → lanjut
   - **Tidak** → tolak + daftar part belum diambil
3. **System** isi `closed_at = now()`
4. **System** hitung waktu: masih dalam jadwal shift hari ini (WITA)?
   - **Ya** → `equipment.status = running`
   - **Tidak** → `equipment.status = idle`
5. **System** commit (status WO + alat dalam 1 transaksi) → tampilkan durasi downtime (`closed_at − detected_at`)

## Alternative Flows

### AF-1: WO tanpa part
- **Trigger:** Breakdown selesai tanpa ganti part
- **Step 2a:** Tidak ada baris planned → langsung boleh close

## Exception Flows

### EF-1: Ada part planned tersisa
- **Trigger:** Foreman buru-buru close
- **Step X:** Tolak, tunjukkan daftar → arahkan ke UC-013

## Business Rules

- **BR-044** (close wajib taken), **BR-045** (kembali otomatis), **BR-046** (hak), **BR-038**, **BR-047** (downtime)

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| work_order | Update | closed + closed_at |
| equipment | Update | status → running/idle |
