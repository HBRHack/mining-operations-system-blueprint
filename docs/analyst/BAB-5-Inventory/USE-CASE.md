# Use Cases — BAB-5 Inventory

## UC-015: Catat Stok Masuk

Storekeeper mencatat penerimaan part baru dari supplier.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Storekeeper | storekeeper | Menambah stok fisik ke sistem |

## Preconditions

- Role `storekeeper`/`admin` (BR-059)
- Part sudah ada di master

## Postconditions

- `stock_transaction(tipe=in)` tersimpan, `part.stok` bertambah

## Main Flow

1. **Storekeeper** buka "Stok Masuk" → pilih part, isi qty, keterangan
2. **System** validasi qty > 0 (BR-061)
3. **System** dalam 1 transaksi: insert `stock_transaction(in, work_order_id=null)` + `part.stok += qty` (BR-057)
4. **System** tampilkan kartu stok terbaru + badge bila menipis (BR-060)

## Exception Flows

### EF-1: Qty ≤ 0
- **Trigger:** Input tidak valid
- **Step X:** Tolak, tidak ada perubahan

## Business Rules

- **BR-057** (lewat transaksi), **BR-059** (form terpisah), **BR-061** (qty > 0)

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| stock_transaction | Create | in |
| part | Update | stok bertambah |

---

## UC-016: Lihat Kartu Stok & Riwayat

Semua role yang berkepentingan melihat riwayat keluar-masuk per part.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Storekeeper, Foreman, Admin | storekeeper/foreman/admin | Menelusuri pergerakan stok |

## Preconditions

- Part punya minimal 1 transaksi

## Postconditions

- Tampil riwayat + saldo (read-only)

## Main Flow

1. **Pengguna** buka part → tab "Riwayat Stok"
2. **System** query `stock_transaction` per part (urut terbaru), tampilkan tipe, qty, WO ref (bila ada), keterangan, pelaku
3. **System** tampilkan badge "Stok Menipis" bila `stok ≤ min_stok` (BR-060)

## Business Rules

- **BR-060** (alert), **BR-063** (append-only — tanpa tombol edit/hapus)

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| stock_transaction | Read | kartu stok |
| part | Read | saldo & badge |

---

## UC-020: Lihat Daftar Part Stok Menipis

Storekeeper memantau part yang perlu segera di-restock untuk perencanaan pengadaan.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Storekeeper, Admin | storekeeper/admin | Tahu part mana yang harus segera dipesan |

## Preconditions

- Master part sudah terisi (FR-049)

## Postconditions

- Daftar part stok ≤ min_stok terpampang, siap jadi dasar pengadaan (read-only)

## Main Flow

1. **Storekeeper** buka daftar part → filter "Stok Menipis" (atau lihat badge di katalog)
2. **System** query `part` dengan kondisi `stok ≤ min_stok` (BR-060), urutkan stok naik
3. **System** tampilkan kode, nama, stok saat ini, min_stok, selisih kekurangan
4. **Storekeeper** pakai daftar ini sebagai acuan input stok masuk (UC-015)

## Exception Flows

- **Tidak ada part menipis:** tampilkan "Semua stok aman" — bukan halaman kosong tanpa keterangan

## Business Rules

- **BR-060** (badge/alert stok menipis)

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| part | Read | filter stok ≤ min_stok |
