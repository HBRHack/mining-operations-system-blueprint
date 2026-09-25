# Process Flows — BAB-2 Master Data

Proses inti chapter ini: menjaga dua sumber data dasar (alat & part) tetap hidup, serta mencatat selisih stok fisik lewat koreksi (stock opname). Koreksi stok adalah titik masuk transaksi stok pertama sebelum WO terlibat (alur stok masuk di PF-010, BAB-5).

## PF-008: Alur CRUD Master Alat & Master Part

Proses penambahan/pengubahan data alat dan spare part — dua alur dengan pola sama, berbeda entitas.

**Trigger:** Admin membuka menu "Master Alat" / "Master Part".

**Goal:** Katalog alat & part lengkap, kode unik, siap dipakai modul lain (P2H butuh alat, WO butuh part).

**Actors:**
- Admin: pelaku CRUD (storekeeper boleh untuk part — BR-014)

### Flow

```
START
  |
  v
[1] Admin buka daftar alat / part
  |
  v
[2] Aksi?
  |
  +--- TAMBAH ---> [3] Isi form
  |                 Alat: kode (EX-01), nama, kategori, unit_code,
  |                       kapasitas, status = idle
  |                 Part: kode (FLT-001), nama, satuan, stok = 0,
  |                       min_stok, lokasi rak
  |                    |
  |                    v
  |                 [4] Validasi
  |                    - kode unik?
  |                    - field wajib terisi?
  |                    - stok awal part = 0 (masuk lewat transaksi,
  |                      bukan angka manual — BR-057)
  |                    |
  |                    +--- GAGAL --> error per field ----> kembali [3]
  |                    |
  |                    +--- OK ----> [5] Insert
  |                                    |
  |                                    v
  |                                 [6] Tampilkan baris baru ----> END
  |
  +--- UBAH -----> [3b] Edit (nama, kategori, min_stok, rak, dll)
  |                    |
  |                    v
  |                 [7] Validasi kode masih unik bila diubah
  |                    |
  |                    +--- GAGAL --> error ----> kembali [3b]
  |                    |
  |                    +--- OK ----> [8] Update ----> END
  |
  +--- HAPUS ----> [9] Punya referensi?
  |                    (P2H log / WO / transaksi stok menunjuk id ini?)
  |                    |
  |                    +--- YA --> tolak: "Data sudah dipakai —
  |                    |            nonaktifkan saja" ----> END
  |                    |
  |                    +--- TIDAK --> [10] Hard delete ----> END
  |
  +--- LIHAT -----> [11] Detail + riwayat terkait ----> END
```

### Catatan Implementasi

| Aspek | Nilai |
|-------|-------|
| Anti-orphan | Hapus hanya bila 0 referensi; selain itu nonaktifkan |
| Part tanpa stok manual | Stok awal = 0; semua penambahan lewat `stock_transaction` (BR-057) |
| BR terkait | BR-011 (kode unik alat), BR-012 (alat baru idle), BR-013 (hapus = nonaktif bila ada riwayat), BR-014 (kode part unik), BR-015 (stok awal part = 0) |
| UC terkait | UC-004, UC-005 |

---

## PF-009: Alur Koreksi Stok Manual (Stock Opname)

Proses menyamakan stok sistem dengan stok fisik gudang — transaksi stok tanpa ikatan Work Order.

**Trigger:** Storekeeper menemukan selisih stok fisik vs sistem.

**Goal:** `part.stok` sesuai hitungan fisik, dengan jejak alasan di `stock_transaction`.

**Actors:**
- Storekeeper: pelaku koreksi (Admin juga boleh — BR-059)

### Flow

```
START
  |
  v
[1] Storekeeper buka part -> "Koreksi Stok"
  |
  v
[2] Isi form: arah (in/out), qty, keterangan (WAJIB)
  |
  v
[3] Validasi
    - qty > 0? (BR-061)
    - keterangan terisi? (BR-058)
    - bila out: stok - qty >= 0? (BR-056)
  |
  +--- GAGAL ---> [4] Pesan spesifik
  |                 - qty tidak valid
  |                 - "Keterangan wajib diisi"
  |                 - "Stok tidak cukup (sisa: N)"
  |                    |
  |                    +-----> kembali [2] (TANPA perubahan data)
  |
  +--- OK ------> [5] DB TRANSACTION MULAI
  |
                  v
                [6] INSERT stock_transaction
                    (tipe = in/out, qty, keterangan,
                     work_order_id = NULL, pelaku = aktor)
                    |
                    v
                [7] UPDATE part.stok (+/- qty)
                    |
                    +--- ERROR DI TENGAH ---> [8] ROLLBACK total
                    |                          (stok tak berubah,
                    |                           tidak ada baris
                    |                           yatim) -> ulang [2]
                    |
                    +--- OK ----------------> [9] COMMIT
                                                |
                                                v
                                             [10] Tampilkan kartu
                                                  stok terbaru +
                                                  badge menipis
                                                  bila stok <= min_stok
                                                  (BR-060) ----> END
```

### Catatan Implementasi

| Aspek | Nilai |
|-------|-------|
| Aturan atomisasi | Satu transaksi DB: INSERT log + UPDATE saldo — gagal = rollback total (BR-057, NFR-REL-001) |
| `work_order_id = NULL` | Pembeda koreksi vs pengambilan part untuk WO (PF-004) |
| BR terkait | BR-056 (stok ≥ 0), BR-057 (saldo hanya via transaksi), BR-058 (keterangan wajib), BR-059 (role), BR-061 (qty > 0) |
| UC terkait | UC-006 |
