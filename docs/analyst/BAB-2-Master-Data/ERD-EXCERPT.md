# ERD Excerpt — BAB-2 Master Data

> Excerpt dari `00-Global/ERD-MASTER.md` — sumber kebenaran ada di master. Read-only view; jangan edit manual.

```
+------------------+
|    equipment     |
+------------------+
| PK | id          |
|    | unit_code   | unique (EX-01, DT-01...)
|    | tipe        |
|    | model       |
|    | kapasitas   |
|    | status      | 5 state machine
|    | hour_meter  |
+--------+---------+
         | 1:N (ke BAB-3 P2H/log, BAB-4 WO)
         +--> FK to p2h_checklist / shift_log / work_order

+------------------+
|      part        |
+------------------+
| PK | id          |
|    | kode        | unique
|    | nama        |
|    | kategori    |
|    | satuan      |
|    | stok        | >= 0
|    | min_stok    |
+--------+---------+
         | 1:N / M:N (ke BAB-4 work_order_part, BAB-5 stock_transaction)
         +--> FK to work_order_part / stock_transaction
```
