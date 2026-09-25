# ERD Excerpt — BAB-6 Dashboard

> Excerpt dari `00-Global/ERD-MASTER.md` — sumber kebenaran ada di master. Read-only view; jangan edit manual. BAB-6 hanya membaca (read-only agregat).

```
Yang dibaca dashboard:

+------------------+    +------------------+    +----------------------+
|    equipment     |    |    work_order    |    |  stock_transaction   |
+------------------+    +------------------+    +----------------------+
|    | status       |   |    | trigger_type |   |    | tipe = out       |
+--------+----------+   |    | detected_at  |   | FK | work_order_id <>NULL
         |              |    | closed_at    |   | FK | part_id          |
    COUNT GROUP BY      +--------+----------+   +----------+-----------+
    status (BR-076)             |                          |
                        downtime + KPI per           GROUP BY part_id
                        trigger_type (BR-077,       ORDER BY COUNT DESC
                        BR-079)                     LIMIT 5 (BR-078)

   +------------------+
   |      part        |  --> badge "Stok Menipis" bila stok <= min_stok (BR-060)
   +------------------+
```
