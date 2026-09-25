# Data Dict Excerpt — BAB-2 Master Data

> Excerpt dari `00-Global/DATA-DICT-MASTER.md` — sumber kebenaran ada di master.

## equipment

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| unit_code | string(10) | Yes | — | unique, regex `^(EX\|DT\|DZ\|WT)-\d{2}$` | Kode unit |
| tipe | enum | Yes | — | Tipe Alat list | Kelompok alat |
| model | string(50) | No | — | — | PC200, D85 |
| kapasitas | string(30) | No | — | — | "40 ton" |
| status | enum | Yes | idle | 5 State Alat | State machine |
| hour_meter | decimal(10,1) | Yes | 0 | ≥ 0 | Jam operasi kumulatif |

**Value List — Status Alat (5):** idle / running / rejected / breakdown / maintenance (`p2h_ok` TIDAK ada di sini)
**Value List — Tipe Alat:** excavator / dump_truck / bulldozer / water_truck

## part

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| kode | string(30) | Yes | — | unique | Kode part |
| nama | string(100) | Yes | — | — | Nama part |
| kategori | enum | Yes | — | Kategori Part list | engine/hidrolik/electrical/undercarriage/tire |
| satuan | enum | Yes | — | pcs/liter/set/unit | Satuan |
| stok | int | Yes | 0 | **≥ 0** (BR-056) | Stok saat ini |
| min_stok | int | Yes | 0 | ≥ 0 | Ambang alert |
