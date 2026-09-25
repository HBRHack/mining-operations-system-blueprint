# Data Dictionary — Master

Sumber kebenaran seluruh field 10 entitas. Excerpt per chapter auto-generate dari sini (jangan diedit manual di chapter).

## Entities

### users
Pengguna sistem dengan 1 dari 4 role tetap.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| name | string(100) | Yes | — | min 3 karakter | Nama pengguna |
| email | string | Yes | — | unique, format email | Login identifier |
| password | string | Yes | — | min 8, hashed | Password |
| role | enum | Yes | operator | nilai di Value List "Role" | Peran asli pengguna |
| act_as | enum | No | null | harus salah satu role; hanya Admin | Peran yang sedang ditiru |
| created_at | datetime | Yes | now() | — | Record creation |
| updated_at | datetime | Yes | now() | — | Last update |

**Business Rules:** BR-001 (login), BR-002 (hak akses per role), BR-003 (act-as)
**Relationships:** users --1:N-- p2h_checklist, shift_log, work_order, stock_transaction

---

### equipment
Unit alat berat dengan status state machine 5 nilai.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| unit_code | string(10) | Yes | — | unique, regex `^(EX\|DT\|DZ\|WT)-\d{2}$` | Kode unit (EX-01) |
| tipe | enum | Yes | — | Value List "Tipe Alat" | Kelompok alat |
| model | string(50) | No | — | — | PC200, D85, dst. |
| kapasitas | string(30) | No | — | — | "40 ton", "20 kL" |
| status | enum | Yes | idle | Value List "Status Alat" (5) | State machine saat ini |
| hour_meter | decimal(10,1) | Yes | 0 | ≥ 0 | Akumulasi jam operasi |
| created_at | datetime | Yes | now() | — | Record creation |
| updated_at | datetime | Yes | now() | — | Last update |

**Business Rules:** BR-021 (P2H gate), BR-036 (transisi status via WO), BR-040 (maks 1 WO terbuka)
**Relationships:** equipment --1:N-- p2h_checklist, shift_log, work_order

---

### p2h_checklist
Header checklist P2H harian per alat.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| equipment_id | bigint | Yes | — | FK equipment, exists | Alat yang diperiksa |
| user_id | bigint | Yes | — | FK users, role=operator | Pengisi checklist |
| tanggal | date | Yes | today() | unique per equipment | Hari P2H (WITA) |
| hasil | enum | Yes | — | pass / fail | Hasil akhir P2H |
| created_at | datetime | Yes | now() | — | Record creation |
| updated_at | datetime | Yes | now() | — | Last update |

**Business Rules:** BR-022 (1× per alat per hari), BR-023 (fail safety-critical → rejected)
**Relationships:** p2h_checklist --1:N-- p2h_checklist_item; equipment --1:N-- p2h_checklist

---

### p2h_checklist_item
Baris item hasil pemeriksaan (snapshot nama + flag saat diisi).

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| p2h_checklist_id | bigint | Yes | — | FK, cascade delete | Header checklist |
| nama_item | string(100) | Yes | — | — | Snapshot nama item (10 item tetap) |
| safety_critical | boolean | Yes | false | — | Snapshot flag: oli/rem/hidrolik |
| status_item | enum | Yes | ok | ok / fail | Hasil per item |
| catatan | string(255) | No | — | wajib bila status_item=fail | Keterangan kegagalan |

**Business Rules:** BR-023 (gagal safety-critical → alat ditolak), BR-024 (gagal non-kritis → warning)
**Relationships:** p2h_checklist --1:N-- p2h_checklist_item

---

### shift
Master 2 shift (diseed, bukan CRUD bebas).

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| kode | enum | Yes | — | day / night | Kode shift |
| nama | string(30) | Yes | — | — | "Day Shift"/"Night Shift" |
| jam_mulai | time | Yes | — | — | 06:00 / 18:00 |
| jam_selesai | time | Yes | — | ≠ jam_mulai | 18:00 / 06:00 (lintas hari) |
| created_at / updated_at | datetime | Yes | now() | — | Timestamps |

**Relationships:** shift --1:N-- shift_log

---

### shift_log
Catatan jam operasional alat per shift.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| equipment_id | bigint | Yes | — | FK equipment | Alat beroperasi |
| shift_id | bigint | Yes | — | FK shift | Shift terkait |
| user_id | bigint | Yes | — | FK users, role=operator | Pengisi log |
| tanggal | date | Yes | — | — | Hari operasi (WITA) |
| jam_mulai | time | Yes | — | — | Mulai operasi |
| jam_selesai | time | Yes | — | > jam_mulai (atau lintas hari) | Selesai operasi |
| material | enum | Yes | none | ob / coal / ore / none | Material yang diangkut |
| tonase | decimal(12,2) | No | 0 | ≥ 0; wajib > 0 bila material ≠ none | Ton terangkut |
| alasan_idle | string(255) | No | — | **wajib** bila jam_operasi = 0 | Alasan tidak beroperasi |
| jam_operasi | decimal(4,2) | Yes | hitung | 0 ≤ x ≤ 12 | Hasil selisih jam |
| hour_meter_akhir | decimal(10,1) | Yes | hitung | ≥ HM sebelumnya | HM setelah log ini |
| created_at / updated_at | datetime | Yes | now() | — | Timestamps |

**Business Rules:** BR-025 (jam_operasi = selisih jam), BR-026 (HM bertambah = jam_operasi), BR-027 (alasan_idle wajib bila 0)
**Relationships:** equipment --1:N-- shift_log; shift --1:N-- shift_log; users --1:N-- shift_log

---

### part
Master spare part dengan stok non-negatif.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| kode | string(30) | Yes | — | unique | Kode part (ENG-OIL-15W40) |
| nama | string(100) | Yes | — | — | Nama part |
| kategori | enum | Yes | — | Value List "Kategori Part" | engine/hidrolik/electrical/undercarriage/tire |
| satuan | enum | Yes | — | pcs / liter / set / unit | Satuan pengukuran |
| stok | int | Yes | 0 | **≥ 0 selalu** (BR-056) | Stok saat ini |
| min_stok | int | Yes | 0 | ≥ 0 | Ambang alert |
| created_at / updated_at | datetime | Yes | now() | — | Timestamps |

**Business Rules:** BR-056 (stok ≥ 0), BR-060 (alert min_stok)
**Relationships:** part --M:N-- work_order (via work_order_part); part --1:N-- stock_transaction

---

### work_order
Tiket perbaikan — jantung integrasi 3 modul.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| wo_code | string(15) | Yes | auto-gen | unique, `WO-YYYY-NNNN` | Nomor WO |
| equipment_id | bigint | Yes | — | FK equipment; maks 1 WO terbuka per alat (BR-040) | Alat yang rusak |
| status | enum | Yes | open | Value List "Status WO" | Fase perbaikan |
| trigger_type | enum | Yes | — | p2h_gagal / breakdown_lapangan | Asal temuan (KPI safety) |
| prioritas | enum | Yes | high | low / medium / high | Di-set Foreman (auto=high) |
| deskripsi | text | Yes | — | min 10 karakter | Uraian kerusakan |
| reported_by | bigint | Yes | — | FK users | Pelapor |
| assigned_to | bigint | No | null | FK users, role=foreman | Foreman penanggung jawab |
| detected_at | datetime | Yes | now() | — | Mulai hitung downtime |
| closed_at | datetime | No | null | > detected_at | Selesai — akhir downtime |
| created_at / updated_at | datetime | Yes | now() | — | Timestamps |

**Business Rules:** BR-036 (auto-buat WO), BR-037 (urutan transisi status), BR-040 (maks 1 terbuka), BR-044 (close wajib semua part taken), BR-045 (kembali running/idle otomatis)
**Relationships:** work_order --M:N-- part (bridge work_order_part); work_order --1:N-- stock_transaction; equipment --1:N-- work_order

---

### work_order_part
Bridge M:N: part yang dibutuhkan vs benar-benar terpakai.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| work_order_id | bigint | Yes | — | FK, cascade delete | WO terkait |
| part_id | bigint | Yes | — | FK part | Part dibutuhkan |
| qty | int | Yes | — | > 0 | Jumlah dibutuhkan |
| status | enum | Yes | planned | planned / taken | Rencana vs terpakai |
| stock_transaction_id | bigint | No | null | FK stock_transaction; wajib bila status=taken | Jejak transaksi |
| created_at / updated_at | datetime | Yes | now() | — | Timestamps |

**Business Rules:** BR-041 (rencana ≠ transaksi), BR-042 (ambil part atomik), BR-044 (close wajib taken)
**Relationships:** work_order --M:N-- part (bridge)

---

### stock_transaction
Jurnal keluar-masuk stok — satu-satunya jalur mengubah `part.stok`.

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| part_id | bigint | Yes | — | FK part | Part terkait |
| work_order_id | bigint | No | null | FK work_order | Rujukan WO (null = in/koreksi) |
| user_id | bigint | Yes | — | FK users | Pencatat transaksi |
| tipe | enum | Yes | — | in / out | Arah transaksi |
| qty | int | Yes | — | > 0 | Jumlah (arah dari tipe) |
| keterangan | string(255) | No | — | wajib bila work_order_id null (koreksi/masuk manual, BR-058/BR-059) | Catatan |
| created_at / updated_at | datetime | Yes | now() | — | Timestamps |

**Business Rules:** BR-042 (transaksi sekali per klik), BR-056 (validasi stok ≥ 0 dalam transaksi DB), BR-057 (stok hanya berubah lewat tabel ini)
**Relationships:** part --1:N-- stock_transaction; work_order --1:N-- stock_transaction; users --1:N-- stock_transaction

## Value Lists

### Status Alat (equipments.status) — 5 state
| Code | Label | Description |
|------|-------|-------------|
| idle | Idle | Siap/tidak beroperasi, menunggu shift |
| running | Running | Sedang beroperasi (setelah P2H lolos) |
| rejected | Rejected | **Transisi singkat** — P2H gagal safety-critical; langsung dispar ke WO |
| breakdown | Breakdown | **Transisi singkat** — rusak saat operasi; langsung dispar ke WO |
| maintenance | Maintenance | Dalam perbaikan (seluruh fase WO open→closed) |

> `p2h_ok` TIDAK ada di sini — itu hasil validasi sesaat di `p2h_checklist.hasil`, bukan state.

### Status WO (work_orders.status)
| Code | Label | Description |
|------|-------|-------------|
| open | Open | Baru dibuat, belum ada rencana part / belum diklaim |
| menunggu_part | Menunggu Part | Ada part planned yang stoknya kurang (backorder) |
| in_progress | In Progress | Semua part sudah diambil, perbaikan berjalan |
| closed | Closed | Selesai; alat kembali running/idle |

### Trigger Type (work_orders.trigger_type)
| Code | Label | Description |
|------|-------|-------------|
| p2h_gagal | P2H Gagal | Temuan sebelum alat jalan (KPI safety) |
| breakdown_lapangan | Breakdown Lapangan | Kerusakan saat operasi (KPI keandalan) |

### Tipe Alat (equipment.tipe)
| Code | Label | Contoh model |
|------|-------|--------------|
| excavator | Excavator | PC200, PC2000 |
| dump_truck | Dump Truck | 40 ton |
| bulldozer | Bulldozer | D85 |
| water_truck | Water Truck | 20 kL |

### Kategori Part (part.kategori)
| Code | Label |
|------|-------|
| engine | Engine |
| hidrolik | Hidrolik |
| electrical | Electrical |
| undercarriage | Undercarriage |
| tire | Tire/Ban |

### Material (shift_log.material)
| Code | Label |
|------|-------|
| ob | Overburden (OB) |
| coal | Batubara |
| ore | Ore/Mineral |
| none | Tidak ada (idle) |

### Role (users.role)
| Code | Label | Hak inti |
|------|-------|----------|
| admin | Admin | Kelola user & master; act-as semua role |
| foreman | Foreman | Klaim, prioritas, tutup WO |
| operator | Operator | P2H, log shift, lapor breakdown |
| storekeeper | Storekeeper | Transaksi stok, ambil part |

### Prioritas WO (work_orders.prioritas)
| Code | Label | Catatan |
|------|-------|---------|
| high | High | Default WO auto-created |
| medium | Medium | Di-set Foreman |
| low | Low | Di-set Foreman |

## Data Flow Summary

| Source | Target | Data | Frequency | Method |
|--------|--------|------|-----------|--------|
| Form P2H (Operator) | p2h_checklist + item → equipment | hasil pass/fail → status alat | 1×/alat/hari | Form Blade submit |
| Gagal safety-critical | work_order (auto) | WO trigger_type=p2h_gagal | Saat terjadi | DB transaction internal |
| Form lapor breakdown (Operator) | work_order (auto) | WO trigger_type=breakdown_lapangan | Saat terjadi | DB transaction internal |
| Form log shift (Operator) | shift_log → equipment.hour_meter | jam operasi, tonase, HM | 2×/alat/hari | Form Blade submit |
| Tombol Ambil Part (Storekeeper) | stock_transaction (out) → part.stok | qty keluar, atomik | Per pengambilan | DB transaction internal |
| Form stok masuk (Storekeeper) | stock_transaction (in) → part.stok | qty masuk | Per penerimaan | Form Blade submit |
| work_order + stock_transaction + equipment | Dashboard (read-only) | agregat status, downtime, top part | On-load | Query agregat |
