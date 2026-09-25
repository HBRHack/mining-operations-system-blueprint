# Process Flows — BAB-6 Dashboard & Laporan

Chapter ini **read-only total** — tidak ada flow yang mengubah data. Satu proses utama: mengambil angka segar dari database lalu merendernya cepat (NFR-PERF-001: < 2 detik), dengan lapisan filter role.

## PF-013: Alur Render Dashboard & Laporan

Proses agregasi data untuk dashboard status alat dan halaman laporan.

**Trigger:** Pengguna membuka `/dashboard` atau halaman laporan (downtime, top part, safety).

**Goal:** Angka live, sesuai peran, dirender < 2 detik tanpa N+1 query.

**Actors:**
- Semua role: melihat kartu status (BR-080)
- Foreman/Admin: panel downtime, top part, KPI (BR-080)
- Storekeeper: badge stok menipis (via UC-020)

### Flow

```
START
  |
  v
[1] Pengguna buka /dashboard
  |
  v
[2] Middleware: sesi aktif? (BR-001)
  |
  +--- TIDAK ---> Redirect /login ----> END
  |
  +--- YA ------> [3] Tentukan cakupan data = role aktif
                   (bila act-as: pakai role target)
  |
                   v
                [4] QUERY AGREGAT (eager / GROUP BY,
                    tanpa N+1 — NFR-PERF-001)
                    |
                    +---> COUNT(equipment) GROUP BY status
                    +---> downtime aktif (WO belum closed,
                    |      pakai now() - detected_at, BR-047)
                    +---> top 5 part (sum qty OUT + dari WO)
                    +---> KPI safety: COUNT(WO) GROUP BY
                    |      trigger_type (BR-039)
                    +---> part WHERE stok <= min_stok (BR-060)
                    |
                    v
                [5] Filter role (BR-080)
                    |
                    +--- Operator ---> KARTU STATUS SAJA
                    |                     |
                    +--- Storekeeper -> KARTU + BADGE MENIPIS
                    |                     |
                    +--- Foreman ----> KARTU + DOWNTIME + TOP PART
                    |                     |
                    +--- Admin ------> SEMUA PANEL + KPI
                    |
                    v
                [6] Rendor Blade (format ID, WITA, d/m/Y)
                    |
                    v
                [7] Stok sudah lewat 2 detik?
                    |
                    +--- YA ---> catat: optimasi query dulu
                    |            sebelum cache (BR-076: data live,
                    |            boleh spot-check tapi tanpa cache)
                    |
                    +--- TIDAK -> tampilkan halaman ----> END
```

### Laporan (read-only, filter tanggal)

```
[1] Pengguna buka laporan (downtime / part / safety)
      |
      v
[2] Pilih rentang tanggal + filter (role & alat bila perlu)
      |
      v
[3] QUERY dengan WHERE created_at / detected_at dalam rentang
      |
      v
[4] FORMAT
      - downtime: closed_at - detected_at;
        belum closed = now() - detected_at (BR-047)
      - top part: SUM(qty) tipe out, urut turun, LIMIT 5
      - safety: COUNT per trigger_type (p2h_gagal vs
        breakdown_lapangan) — 2 KPI dari 1 tabel (BR-039)
      |
      v
[5] TAMPILKAN (bisa diekspor CSV ringan — belum dijanjikan
    sebagai FR, opsional) ----> END
```

### Exception Flows

| Kondisi | Respons |
|---------|---------|
| Rentang tanpa data | "Tidak ada data pada rentang ini" — bukan tabel kosong |
| WO belum closed | Downtime tetap dihitung berjalan (`now()`), ditandai "(berjalan)" |

### Catatan Implementasi

| Aspek | Nilai |
|-------|-------|
| Tanpa cache | BR-076 — angka segar tiap request; kuncinya query efisien, bukan cache |
| Filter role | BR-080 — payload berbeda per role, query dasar sama |
| BR terkait | BR-039 (KPI per trigger_type), BR-047 (rumus downtime), BR-060 (badge), BR-076, BR-080 |
| NFR terkait | NFR-PERF-001 (render < 2 detik), NFR-USAB-001 (label ID) |
| UC terkait | UC-017, UC-018, UC-019, UC-021 |
