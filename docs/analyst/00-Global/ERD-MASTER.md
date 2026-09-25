# ERD: Simulasi Sistem Operasional Tambang (Loading–Hauling)

Data model 3 modul terintegrasi: Equipment Monitoring, Inventory Spare Part, Maintenance Work Order — dengan state machine alat 5 status sebagai jantung integrasinya.

## Entity Relationship Diagram

```
+------------------+        +------------------+
|      users       |        |    equipment     |
+------------------+        +------------------+
| PK | id          |        | PK | id          |
|    | name        |        |    | unit_code   | (unique: EX-01, DT-01...)
|    | email       |unique  |    | tipe        |
|    | password    |        |    | model       |
|    | role        |        |    | kapasitas   |
|    | act_as      |        |    | status      | (5 state machine)
|    | created_at  |        |    | hour_meter  |
+--------+---------+        |    | created_at  |
         |                  +---+----+---------+
         |                       /    |    \
        1:N  ┌───────────────────/     |     \──────────────┐
         |   | 1:N                    1:N                    | 1:N
+--------v---------+        +---------v--------+   +--------v---------+
|   shift_log      |        |  p2h_checklist   |   |   work_order     |
+------------------+        +------------------+   +------------------+
| PK | id          |        | PK | id          |   | PK | id          |
| FK | equipment_id|        | FK | equipment_id|   | FK | equipment_id|
| FK | shift_id    |        | FK | user_id     |   |    | wo_code     | (unique WO-2026-0001)
| FK | user_id     |        |    | tanggal     |   |    | status      |
|    | tanggal     |        |    | hasil       | (pass/fail) | trigger_type |
|    | jam_mulai   |        +--------+---------+   |    | prioritas   |
|    | jam_selesai |                 | 1:N        | FK | reported_by |
|    | material    |        +---------v--------+   | FK | assigned_to |
|    | tonase      |        |p2h_checklist_item|   |    | detected_at |
|    | alasan_idle |        +------------------+   |    | closed_at   |
|    | hour_meter_akhir                         +--+---+----+--------+
+------------------+                            /       |    \
                                               /        |     \ 1:N
                             +-----------------/-+  1:N  |      |
                             | work_order_part |<--------      |
+------------------+         +------------------+              | 1:N
|      part        |         | PK | id          |    +---------v--------+
+------------------+         | FK | work_order_id     | stock_transaction|
| PK | id          |    1:N  | FK | part_id     |    +------------------+
|    | kode        |<--------|    | qty         |    | PK | id          |
|    | nama        |         |    | status      |    | FK | part_id     |
|    | kategori    |         +------------------+    | FK | work_order_id| (nullable)
|    | satuan      |                                 | FK | user_id     |
|    | stok        |<------------------------------- |    | tipe        | (in/out)
|    | min_stok    |              1:N                |    | qty         |
|    | created_at  |                                 +------------------+
+------------------+

Legend:
  PK = Primary Key        FK = Foreign Key        1:N = One-to-Many
  work_order_part = bridge table M:N (work_order ↔ part)
  stock_transaction.work_order_id nullable — transaksi 'in' & koreksi manual tanpa WO
```

## Entity Details

### users

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | bigint | Yes | auto | Primary key |
| name | string | Yes | — | Nama pengguna |
| email | string | Yes | — | Unique, login identifier |
| password | string | Yes | — | Hashed |
| role | enum | Yes | operator | admin / foreman / operator / storekeeper |
| act_as | enum | Yes | null | Peran yang sedang ditiru Admin (null = peran asli) |
| created_at / updated_at | datetime | Yes | now() | Timestamps |

**Constraints:** unique(email); role ∈ {admin, foreman, operator, storekeeper}
**Indexes:** role — filter halaman per peran

### equipment

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | bigint | Yes | auto | Primary key |
| unit_code | string | Yes | — | Unique: EX-01, DT-01, DZ-01, WT-01 |
| tipe | enum | Yes | — | excavator / dump_truck / bulldozer / water_truck |
| model | string | No | — | PC200, PC2000, D85, dst. |
| kapasitas | string | No | — | "40 ton", "20 kL", "1,8 m³" |
| status | enum | Yes | idle | **5 state:** idle / running / rejected / breakdown / maintenance |
| hour_meter | decimal(10,1) | Yes | 0 | Akumulasi jam operasi |
| created_at / updated_at | datetime | Yes | now() | Timestamps |

**Constraints:** unique(unit_code); status ∈ 5 nilai state machine
**Indexes:** status — dashboard agregat per status

### p2h_checklist

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | bigint | Yes | auto | Primary key |
| equipment_id | bigint | Yes | — | FK → equipment |
| user_id | bigint | Yes | — | FK → users (pengisi) |
| tanggal | date | Yes | today() | Hari P2H |
| hasil | enum | Yes | — | pass / fail |
| created_at / updated_at | datetime | Yes | now() | Timestamps |

**Constraints:** unique(equipment_id, tanggal) — maksimal 1 P2H per alat per hari
**Indexes:** (equipment_id, tanggal)

### p2h_checklist_item

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | bigint | Yes | auto | Primary key |
| p2h_checklist_id | bigint | Yes | — | FK → p2h_checklist |
| nama_item | string | Yes | — | Snapshot nama item saat diisi |
| safety_critical | boolean | Yes | false | Snapshot: apakah item ini safety-critical |
| status_item | enum | Yes | ok | ok / fail |
| catatan | string | No | — | Keterangan bila fail |

**Constraints:** FK cascade delete dari p2h_checklist
**Indexes:** p2h_checklist_id

### shift

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | bigint | Yes | auto | Primary key |
| kode | enum | Yes | — | day / night |
| nama | string | Yes | — | "Day Shift" / "Night Shift" |
| jam_mulai | time | Yes | — | 06:00 / 18:00 |
| jam_selesai | time | Yes | — | 18:00 / 06:00 (lintas hari) |
| created_at / updated_at | datetime | Yes | now() | Timestamps |

**Constraints:** unique(kode); 2 baris (diseed)

### shift_log

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | bigint | Yes | auto | Primary key |
| equipment_id | bigint | Yes | — | FK → equipment |
| shift_id | bigint | Yes | — | FK → shift |
| user_id | bigint | Yes | — | FK → users (operator pengisi) |
| tanggal | date | Yes | — | Hari operasi |
| jam_mulai | time | Yes | — | Mulai operasi |
| jam_selesai | time | Yes | — | Selesai operasi |
| material | enum | Yes | — | ob / coal / ore / none |
| tonase | decimal(12,2) | No | 0 | Ton terangkut |
| alasan_idle | string | No | — | Wajib diisi bila jam_operasi = 0 |
| jam_operasi | decimal(4,2) | Yes | — | hasil jam_selesai − jam_mulai |
| hour_meter_akhir | decimal(10,1) | Yes | — | HM equipment setelah log ini |

**Constraints:** check(jam_operasi ≥ 0); alasan_idle wajib bila jam_operasi = 0
**Indexes:** (equipment_id, tanggal) — rekap harian

### part

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | bigint | Yes | auto | Primary key |
| kode | string | Yes | — | Unique: ENG-OIL-15W40, HDL-HOSE-001 |
| nama | string | Yes | — | Nama part |
| kategori | enum | Yes | — | engine / hidrolik / electrical / undercarriage / tire |
| satuan | enum | Yes | — | pcs / liter / set / unit |
| stok | int | Yes | 0 | **Tidak boleh minus** |
| min_stok | int | Yes | 0 | Ambang alert dashboard |
| created_at / updated_at | datetime | Yes | now() | Timestamps |

**Constraints:** unique(kode); check(stok ≥ 0)
**Indexes:** kategori — filter katalog; stok ≤ min_stok — alert

### work_order

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | bigint | Yes | auto | Primary key |
| wo_code | string | Yes | auto | Unique: WO-2026-0001 |
| equipment_id | bigint | Yes | — | FK → equipment |
| status | enum | Yes | open | open / menunggu_part / in_progress / closed |
| trigger_type | enum | Yes | — | p2h_gagal / breakdown_lapangan |
| prioritas | enum | Yes | high | low / medium / high (auto = high) |
| deskripsi | text | Yes | — | Uraian kerusakan (temuan P2H / laporan lapangan) |
| reported_by | bigint | Yes | — | FK → users |
| assigned_to | bigint | No | null | FK → users (Foreman; null = belum diklaim) |
| detected_at | datetime | Yes | now() | Waktu temuan → mulai hitung downtime |
| closed_at | datetime | No | null | Waktu tutup → akhir downtime |
| created_at / updated_at | datetime | Yes | now() | Timestamps |

**Constraints:** unique(wo_code); **maks 1 WO berstatus ≠ closed per equipment** (partial unique index); trigger_type ∈ 2 nilai
**Indexes:** status — board WO; equipment_id — cek WO terbuka; (detected_at, closed_at) — laporan downtime

### work_order_part (bridge M:N)

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | bigint | Yes | auto | Primary key |
| work_order_id | bigint | Yes | — | FK → work_order |
| part_id | bigint | Yes | — | FK → part |
| qty | int | Yes | — | Jumlah yang dibutuhkan |
| status | enum | Yes | planned | planned / taken |
| stock_transaction_id | bigint | No | null | FK → stock_transaction setelah diambil |

**Constraints:** unique(work_order_id, part_id); check(qty > 0)
**Indexes:** work_order_id — validasi close; status — cari yang belum diambil

### stock_transaction

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| id | bigint | Yes | auto | Primary key |
| part_id | bigint | Yes | — | FK → part |
| work_order_id | bigint | No | null | FK → work_order (null = masuk/koreksi manual) |
| user_id | bigint | Yes | — | FK → users (pencatat) |
| tipe | enum | Yes | — | in / out |
| qty | int | Yes | — | Selalu positif; arah ditentukan tipe |
| keterangan | string | No | — | Catatan transaksi |
| created_at / updated_at | datetime | Yes | now() | Timestamps |

**Constraints:** check(qty > 0); setelah insert `out`, stok part ≥ 0 (divalidasi dalam transaksi DB)
**Indexes:** part_id — kartu stok; work_order_id — audit pemakaian per WO

## Relationship Summary

| From | Cardinality | To | FK Location | Description |
|------|-------------|-----|-------------|-------------|
| equipment | 1:N | p2h_checklist | p2h_checklist.equipment_id | P2H harian per alat |
| equipment | 1:N | shift_log | shift_log.equipment_id | Log operasional per alat |
| equipment | 1:N | work_order | work_order.equipment_id | WO per alat (maks 1 terbuka) |
| users | 1:N | p2h_checklist | p2h_checklist.user_id | Operator pengisi P2H |
| users | 1:N | shift_log | shift_log.user_id | Operator pengisi log |
| users | 1:N | work_order | work_order.reported_by / assigned_to | Pelapor & Foreman penanggung jawab |
| users | 1:N | stock_transaction | stock_transaction.user_id | Storekeeper pencatat |
| shift | 1:N | shift_log | shift_log.shift_id | Log per shift |
| p2h_checklist | 1:N | p2h_checklist_item | p2h_checklist_item.p2h_checklist_id | Item checklist |
| work_order | M:N | part | work_order_part (bridge) | Part yang dibutuhkan/terpakai |
| work_order | 1:N | stock_transaction | stock_transaction.work_order_id | Transaksi keluar per WO |
| part | 1:N | stock_transaction | stock_transaction.part_id | Kartu stok |
