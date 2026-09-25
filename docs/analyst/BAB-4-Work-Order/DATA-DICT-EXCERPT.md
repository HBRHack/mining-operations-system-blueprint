# Data Dict Excerpt — BAB-4 Work Order

> Excerpt dari `00-Global/DATA-DICT-MASTER.md` — sumber kebenaran ada di master.

## work_order

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| wo_code | string(15) | Yes | auto-gen | unique, `WO-YYYY-NNNN` | Nomor WO |
| equipment_id | bigint | Yes | — | FK; **maks 1 WO ≠ closed per alat** (BR-040) | Alat rusak |
| status | enum | Yes | open | Status WO list | Fase perbaikan |
| trigger_type | enum | Yes | — | p2h_gagal / breakdown_lapangan (BR-039) | Asal temuan |
| prioritas | enum | Yes | high | low/medium/high | Auto=high (BR-036) |
| deskripsi | text | Yes | — | min 10 karakter | Uraian kerusakan |
| reported_by | bigint | Yes | — | FK users | Pelapor |
| assigned_to | bigint | No | null | FK users (foreman) BR-046 | Foreman PJ |
| detected_at | datetime | Yes | now() | — | Mulai downtime |
| closed_at | datetime | No | null | > detected_at | Akhir downtime (BR-047) |

**Value List — Status WO:** open / menunggu_part / in_progress / closed (transisi BR-037)
**Value List — Trigger Type:** p2h_gagal / breakdown_lapangan

## work_order_part

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| work_order_id | bigint | Yes | — | FK, cascade delete | WO terkait |
| part_id | bigint | Yes | — | FK part | Part dibutuhkan |
| qty | int | Yes | — | > 0 | Jumlah |
| status | enum | Yes | planned | planned / taken | Rencana vs terpakai |
| stock_transaction_id | bigint | No | null | wajib bila status=taken | Jejak transaksi |
