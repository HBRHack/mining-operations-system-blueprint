# ERD Excerpt — BAB-5 Inventory

> Excerpt dari `00-Global/ERD-MASTER.md` — sumber kebenaran ada di master. Read-only view; jangan edit manual.

```
+------------------+         +----------------------+
|      part        |         |  stock_transaction   |
+------------------+         +----------------------+
| PK | id          |    1:N  | PK | id              |
|    | kode        |<--------| FK | part_id          |
|    | stok >= 0   |         | FK | work_order_id    | nullable
|    | min_stok    |         | FK | user_id          |
+------------------+         |    | tipe | in/out    |
                             |    | qty > 0          |
                             |    | keterangan       |
                             +----------+-----------+
                                        | 1:N
                             FK ke work_order (BAB-4)
                             bila tipe=out untuk WO
```

**Aturan kunci:** `part.stok` hanya berubah lewat tabel ini dalam transaksi atomik (BR-057, BR-042). Append-only (BR-063).
