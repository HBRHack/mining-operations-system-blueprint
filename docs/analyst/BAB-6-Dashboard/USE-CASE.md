# Use Cases — BAB-6 Dashboard & Laporan

## UC-017: Dashboard Status Alat

Semua role melihat jumlah alat per status (ringkasan live).

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Semua role | all | Gambaran cepat kondisi fleet |

## Preconditions

- Login (BR-001)

## Postconditions

- Tampil 5 kartu agregat (read-only)

## Main Flow

1. **Pengguna** buka `/dashboard`
2. **System** query `COUNT(equipment) GROUP BY status` (BR-076, live tanpa cache)
3. **System** tampilkan kartu: Idle, Running, Rejected, Breakdown, Maintenance + daftar alat ringkas
4. **System** (role foreman/admin) tampilkan panel downtime, top part, KPI — operator/storekeeper hanya kartu status (BR-080)

## Business Rules

- **BR-076** (real-time), **BR-080** (filter role)

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| equipment | Read | agregat per status |

---

## UC-018: Laporan Downtime

Foreman/Admin melihat total downtime per rentang tanggal.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Foreman, Admin | foreman/admin | Mengukur ketersediaan alat |

## Preconditions

- Role `foreman`/`admin` (BR-080)

## Postconditions

- Tampil total & rincian downtime per WO (read-only)

## Main Flow

1. **Foreman** buka Laporan → pilih rentang tanggal
2. **System** query WO pada rentang → durasi = `closed_at − detected_at` (belum closed: `now()`, BR-047)
3. **System** tampilkan total downtime, per alat, per trigger_type

## Business Rules

- **BR-047** (definisi downtime), **BR-077** (agregasi), **BR-080**

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| work_order | Read | agregat durasi |

---

## UC-019: Part Paling Sering Dipakai

Melihat 5 part teratas pemakaiannya untuk perencanaan stok.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Foreman, Storekeeper, Admin | foreman/storekeeper/admin | Antisipasi pengadaan |

## Preconditions

- Ada transaksi out yang ber-Referensi WO

## Postconditions

- Tampil top 5 part (read-only)

## Main Flow

1. **Pengguna** buka panel "Part Terpakai" di dashboard
2. **System** query `stock_transaction` tipe=out + `work_order_id NOT NULL` → GROUP BY part → COUNT + SUM(qty) → ORDER DESC LIMIT 5 (BR-078)

## Business Rules

- **BR-078** (definisi top part), **BR-080**

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| stock_transaction | Read | agregat out+WO |

---

## UC-021: Laporan Safety (KPI per trigger_type)

Foreman/Admin membandingkan dua KPI: penolakan P2H vs breakdown lapangan.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Foreman, Admin | foreman/admin | Mengevaluasi keselamatan kerja |

## Preconditions

- Role `foreman`/`admin`
- Ada minimal 1 WO

## Postconditions

- Dua angka KPI terpisah tampil (read-only)

## Main Flow

1. **Foreman** buka "Laporan Safety" → pilih periode
2. **System** query `COUNT(*) FROM work_orders GROUP BY trigger_type` (BR-079)
3. **System** tampilkan: "Ditolak jalan karena P2H: N kali" vs "Rusak saat operasi: M kali" + daftar WO per kategori
4. **System** sertakan rincian item P2H mana yang paling sering gagal (dari `p2h_checklist_item` fail)

## Business Rules

- **BR-079** (KPI dari 1 tabel via trigger_type), **BR-039**, **BR-080**

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| work_order | Read | GROUP BY trigger_type |
| p2h_checklist_item | Read | item fail terbanyak |
