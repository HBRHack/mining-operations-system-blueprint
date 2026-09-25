# Data Dict Excerpt — BAB-3 Monitoring Harian

> Excerpt dari `00-Global/DATA-DICT-MASTER.md` — sumber kebenaran ada di master.

## p2h_checklist

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| equipment_id | bigint | Yes | — | FK equipment | Alat diperiksa |
| user_id | bigint | Yes | — | FK users (operator) | Pengisi |
| tanggal | date | Yes | today() | **unique(equipment_id, tanggal)** (BR-022) | Hari P2H (WITA) |
| hasil | enum | Yes | — | pass / fail | Hasil akhir |

## p2h_checklist_item

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| p2h_checklist_id | bigint | Yes | — | FK, cascade delete | Header |
| nama_item | string(100) | Yes | — | — | Snapshot nama (10 item tetap) |
| safety_critical | boolean | Yes | false | — | Snapshot flag (oli/rem/hidrolik) |
| status_item | enum | Yes | ok | ok / fail | Hasil per item |
| catatan | string(255) | No | — | wajib bila fail | Keterangan |

## shift

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| kode | enum | Yes | — | day / night | Kode shift |
| nama | string(30) | Yes | — | — | Day/Night Shift |
| jam_mulai | time | Yes | — | — | 06:00 / 18:00 |
| jam_selesai | time | Yes | — | ≠ jam_mulai | 18:00 / 06:00 |

## shift_log

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| equipment_id | bigint | Yes | — | FK equipment | Alat beroperasi |
| shift_id | bigint | Yes | — | FK shift | Shift terkait |
| user_id | bigint | Yes | — | FK users | Pengisi |
| tanggal | date | Yes | — | unique(equipment, shift, tanggal) BR-028 | Hari operasi |
| jam_mulai / jam_selesai | time | Yes | — | selesai > mulai / lintas hari | Jam |
| material | enum | Yes | none | ob/coal/ore/none | Material |
| tonase | decimal(12,2) | No | 0 | ≥ 0; > 0 bila material ≠ none | Ton |
| alasan_idle | string(255) | No | — | **wajib bila jam_operasi = 0** (BR-027) | Alasan idle |
| jam_operasi | decimal(4,2) | Yes | hitung | 0 ≤ x ≤ 12 | Hasil selisih (BR-025) |
| hour_meter_akhir | decimal(10,1) | Yes | hitung | ≥ HM sebelumnya | HM setelah log (BR-026) |

**Value List — Material:** ob (Overburden) / coal (Batubara) / ore / none
