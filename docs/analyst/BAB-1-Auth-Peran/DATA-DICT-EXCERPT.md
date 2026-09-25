# Data Dict Excerpt — BAB-1 Auth & Peran

> Excerpt dari `00-Global/DATA-DICT-MASTER.md` — sumber kebenaran ada di master.

## users

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| name | string(100) | Yes | — | min 3 | Nama pengguna |
| email | string | Yes | — | unique, format email | Login identifier |
| password | string | Yes | — | min 8, hashed | Password |
| role | enum | Yes | operator | Role: admin/foreman/operator/storekeeper | Peran asli |
| act_as | enum | No | null | harus role valid; hanya Admin | Peran ditiru |
| created_at / updated_at | datetime | Yes | now() | — | Timestamps |

**Value List — Role:**

| Code | Label | Hak inti |
|------|-------|----------|
| admin | Admin | Kelola user & master; act-as |
| foreman | Foreman | Klaim, prioritas, tutup WO |
| operator | Operator | P2H, log shift, lapor breakdown |
| storekeeper | Storekeeper | Transaksi stok, ambil part |
