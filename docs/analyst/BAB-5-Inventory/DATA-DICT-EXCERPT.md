# Data Dict Excerpt — BAB-5 Inventory

> Excerpt dari `00-Global/DATA-DICT-MASTER.md` — sumber kebenaran ada di master.

## part (field inventory-relevan)

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| kode | string(30) | Yes | — | unique | Kode part |
| kategori | enum | Yes | — | Kategori list | Kelompok part |
| satuan | enum | Yes | — | pcs/liter/set/unit | Satuan |
| stok | int | Yes | 0 | **≥ 0 selalu** (BR-056) | Hanya lewat stock_transaction (BR-057) |
| min_stok | int | Yes | 0 | ≥ 0 | Alert ≤ ini (BR-060) |

**Value List — Kategori Part:** engine / hidrolik / electrical / undercarriage / tire

## stock_transaction

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| part_id | bigint | Yes | — | FK part | Part terkait |
| work_order_id | bigint | No | null | null = in/koreksi (BR-058) | Rujukan WO |
| user_id | bigint | Yes | — | FK users | Pencatat |
| tipe | enum | Yes | — | in / out | Arah |
| qty | int | Yes | — | **> 0** (BR-061) | Jumlah |
| keterangan | string(255) | No | — | wajib bila koreksi | Catatan |
| created_at | datetime | Yes | now() | — | Append-only (BR-063) |
