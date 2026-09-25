# Use Cases — BAB-2 Master Data

## UC-004: Kelola Master Alat

Admin menambah/mengubah/menonaktifkan unit alat berat.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | admin | Menjaga katalog alat akurat |

## Preconditions

- Login sebagai `admin` (atau act-as admin)

## Postconditions

- Alat baru tersimpan `status=idle`, `hour_meter=0` / perubahan tersimpan / alat dinonaktifkan

## Main Flow

1. **Admin** buka Master Alat → "Tambah Alat"
2. **Admin** isi unit_code (DT-01), tipe, model, kapasitas
3. **System** validasi unique unit_code + pola regex (BR-011)
4. **System** simpan dengan `status='idle'`, `hour_meter=0` (BR-012)
5. **System** tampilkan daftar alat ter-update

## Alternative Flows

### AF-1: Unit code duplikat / format salah
- **Trigger:** `DT-01` sudah ada atau format bukan (EX|DT|DZ|WT)-NN
- **Step 3a:** System tolak dengan pesan format

### AF-2: Nonaktifkan alat ber-riwayat
- **Trigger:** Alat punya P2H/log/WO dan Admin menekan Hapus
- **Step X:** System tolak hapus keras → tawarkan "Nonaktifkan" (BR-013)

## Exception Flows

### EF-1: Role bukan admin
- **Trigger:** Operator membuka route master alat
- **Step X:** 403 (BR-002)

## Business Rules

- **BR-011:** unit_code unik & pola
- **BR-012:** alat baru = idle
- **BR-013:** alat ber-riwayat tidak dihapus keras

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| equipment | Create/Update | unit_code, tipe, model, kapasitas |

---

## UC-005: Kelola Master Part

Admin/Storekeeper menambah & mengubah data spare part.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin, Storekeeper | admin/storekeeper | Katalog part lengkap & akurat |

## Preconditions

- Login role `admin` atau `storekeeper`

## Postconditions

- Part baru tersimpan dengan `stok = 0`

## Main Flow

1. **Storekeeper** buka Master Part → "Tambah Part"
2. **Storekeeper** isi kode, nama, kategori, satuan, min_stok
3. **System** validasi kode unik + enum kategori/satuan (BR-014)
4. **System** simpan `stok = 0` (BR-015) — field stok tidak tersedia di form
5. **System** tampilkan daftar part

## Alternative Flows

### AF-1: Ubah part
- **Trigger:** Admin membuka part → Ubah
- **Step X:** Kategori/satuan boleh diubah; `stok` tetap tidak bisa diedit langsung (BR-057)

## Business Rules

- **BR-014:** kode unik, enum valid
- **BR-015:** stok awal 0
- **BR-057:** stok hanya lewat stock_transaction

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| part | Create/Update | kode, nama, kategori, satuan, min_stok |

---

## UC-006: Koreksi Stok Manual (Stock Opname)

Storekeeper mencatat selisih stok fisik sebagai transaksi koreksi.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Storekeeper | storekeeper | Menyamakan stok sistem dengan fisik |

## Preconditions

- Login role `storekeeper`/`admin`
- Part sudah ada

## Postconditions

- Baris `stock_transaction` (in/out, tanpa WO, berketerangan) tersimpan
- `part.stok` berubah sesuai koreksi (tetap ≥ 0)

## Main Flow

1. **Storekeeper** buka part → "Koreksi Stok"
2. **Storekeeper** pilih arah (in/out), qty, keterangan (wajib)
3. **System** validasi qty > 0 (BR-061) dan bila out: stok hasil ≥ 0 (BR-056)
4. **System** dalam 1 transaksi DB: insert `stock_transaction` (work_order_id=null) + update `part.stok` (BR-057)
5. **System** tampilkan kartu stok terbaru

## Exception Flows

### EF-1: Koreksi out melebihi stok
- **Trigger:** stok − qty < 0
- **Step X:** Rollback, pesan "stok tidak cukup"

### EF-2: Keterangan kosong
- **Trigger:** Koreksi tanpa alasan
- **Step X:** Tolak (BR-058 wajib keterangan)

## Business Rules

- **BR-056, BR-057, BR-058, BR-061**

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| stock_transaction | Create | tipe, qty, keterangan |
| part | Update | stok (dalam transaksi) |
