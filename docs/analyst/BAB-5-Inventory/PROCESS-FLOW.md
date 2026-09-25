# Process Flows — BAB-5 Inventory

Proses inti chapter ini: setiap pergerakan stok (masuk, koreksi, keluar untuk WO) tercatat sebagai transaksi atomik — saldo `part.stok` tidak pernah diubah langsung. Pengambilan part untuk WO ada di PF-004 (BAB-4, karena memicu status WO `menunggu_part`); chapter ini berisi sisi supply & pemantauan.

## PF-010: Alur Catat Stok Masuk

Proses penerimaan part baru dari supplier — menambah saldo disertai baris jejak.

**Trigger:** Storekeeper menerima part fisik di gudang.

**Goal:** `part.stok` bertambah dan `stock_transaction` punya bukti in.

**Actors:**
- Storekeeper: pelaku (Admin juga boleh — BR-059)

### Flow

```
START
  |
  v
[1] Storekeeper buka "Stok Masuk"
  |
  v
[2] Isi form: pilih part, qty, keterangan
  |
  v
[3] Validasi qty > 0? (BR-061)
  |
  +--- GAGAL ---> [4] "Qty harus lebih dari 0" ----> kembali [2]
  |
  +--- OK ------> [5] DB TRANSACTION MULAI
  |
                  v
                [6] INSERT stock_transaction
                    (tipe = in, qty, keterangan,
                     work_order_id = NULL, pelaku)
                    |
                    v
                [7] UPDATE part.stok += qty
                    |
                    +--- ERROR ---> [8] ROLLBACK -> kembali [2]
                    |
                    +--- OK -----> [9] COMMIT
                                     |
                                     v
                                  [10] Tampilkan kartu stok
                                       + badge "Stok Menipis"
                                       bila stok <= min_stok
                                       (BR-060) ----> END
```

### Catatan Implementasi

| Aspek | Nilai |
|-------|-------|
| atomisasi | INSERT + UPDATE dalam 1 transaksi DB (BR-057, NFR-REL-001) |
| Badge | Selalu cek `stok <= min_stok` setelah COMMIT — bukan cache |
| BR terkait | BR-057, BR-059, BR-060, BR-061 |
| UC terkait | UC-015 |

---

## PF-011: Alur Kartu Stok & Daftar Part Stok Menipis

Proses baca (read-only) untuk telusur riwayat pergerakan dan pantau part yang harus segera direstock.

**Trigger:** Pengguna membuka kartu stok sebuah part, atau filter "Stok Menipis".

**Goal:** Riwayat keluar-masuk + saldo terkini; daftar part ≤ min_stok sebagai acuan pengadaan.

**Actors:**
- Storekeeper / Foreman / Admin: penelusur riwayat
- Storekeeper / Admin: pemantau stok menipis

### Flow

```
START
  |
  +----------+-----------+
  |                    |
  v                    v
[A] KARTU STOK     [B] DAFTAR STOK MENIPIS
(Aktifkan           (UC-020)
 saat buka part)    
  |                    |
  v                    v
[1] Query             [5] Query part WHERE
 stock_transaction     stok <= min_stok
 per part,             ORDER BY stok ASC
 urut terbaru          
  |                    |
  v                    v
[2] Ambil saldo       [6] Hitung selisih
 dari part.stok        (min_stok - stok)
  |                    |
  v                    v
[3] Render tabel      [7] Render tabel:
 riwayat: tipe, qty,   kode, nama, stok,
 WO ref, keterangan,   min_stok, kurang N
 pelaku, tanggal       
  |                    |
  +----------+---------+
             |
             v
[4] Tampilkan badge "Stok Menipis" pada baris
    dengan stok <= min_stok (BR-060)
    |
    v
[8] Bila kosong: tampilkan "Semua stok aman"
    (bukan halaman kosong tanpa keterangan)
    |
    v
  END
```

### Exception Flows

| Kondisi | Respons |
|---------|---------|
| Part belum punya transaksi | Riwayat kosong + ajakan "Belum ada pergerakan stok" |
| Tidak ada part menipis | Pesan "Semua stok aman" (bukan error) |

### Catatan Implementasi

| Aspek | Nilai |
|-------|-------|
| Read-only | Tanpa tombol ubah/hapus — transaksi append-only (BR-063) |
| BR terkait | BR-060 (badge), BR-063 (append-only) |
| UC terkait | UC-016, UC-020 |

---

## PF-012: Alur Integritas Stok — Aturan Global

Ringkasan aturan yang berlaku di SEMUA alur stok (PF-009, PF-010, PF-004) — patokan saat implementasi & menulis test.

**Trigger:** Setiap kali ada niat mengubah `part.stok`.

**Goal:** Stok tidak pernah minus, tidak pernah berubah tanpa jejak.

### Flow

```
[ATURAN 1 — SATU JALUR"]
    part.stok  <--- HANYA boleh diubah oleh UPDATE
        ^            di dalam transaksi DB yang juga
        |            INSERT stock_transaction
        |
        +-- dilarang: form yang menulis stok langsung
        +-- dilarang: delete baris transaksi (append-only)

[ATURAN 2 — TIDAK BOLEH MINUS (BR-056)]
    operasi out/koreksi out:
        stok_sekarang - qty  >= 0  ?
              |                  |
             TIDAK              YA
              |                  |
              v                  v
        ROLLBACK,           lanjut COMMIT
        pesan "Stok
        tidak cukup"

[ATURAN 3 — ATOMIS (BR-057 / NFR-REL-001)]
    BEGIN;
      INSERT stock_transaction ...;
      UPDATE part.stok ...;
      -- gagal di salah satu? ROLLBACK; COMMIT;
    END;

[ATURAN 4 — REncana != TRANSAKSI (BR-041)]
    "rencana part" pada WO = baris rencana (belum ada
    efek apa pun ke stok)
    stok baru berkurang saat klik "Ambil Part"
    -> INSERT transaksi out (PF-004)
```

### Patokan Test

| # | Skenario | Ekspektasi |
|---|----------|------------|
| 1 | Ambil part melebihi stok | Gagal, stok tidak berubah, WO tetap `open` |
| 2 | Ambil part 2 item dalam 1 klik | 2 baris transaksi out ter-commit bersama, atau semua gagal |
| 3 | Koreksi out melebihi stok | Ditolak, tidak ada baris transaksi |
| 4 | Tutup WO dengan item masih `planned` | Ditolak (BR-044) |
| 5 | Direct UPDATE `part.stok` di luar transaksi | Tidak ada jalur kode demikian (audit) |
