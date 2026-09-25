# ERD Excerpt — BAB-4 Work Order

> Excerpt dari `00-Global/ERD-MASTER.md` — sumber kebenaran ada di master. Read-only view; jangan edit manual.

```
+------------------+                    +------------------+
|    equipment     |                    |      part        |
+------------------+                    +------------------+
| PK | id          |                    | PK | id          |
|    | status      | 5 state            |    | stok        |
+---+--------------+                    +--------+---------+
        | 1:N                                   | M:N via bridge
+-------v--------------------+        +---------v------------+
|       work_order           |        |  work_order_part     |
+----------------------------+        +----------------------+
| PK | id                    |   1:N  | PK | id              |
|    | wo_code    | unique   |<-------| FK | work_order_id    |
| FK | equipment_id          |        | FK | part_id          |
|    | status     | open/... |        |    | qty              |
|    | trigger_type | p2h_   |        |    | status | planned/|
|    |            | gagal /  |        |    |        | taken   |
|    |            | breakd.. |        | FK | stock_transaction_id
|    | prioritas  | auto=high|        +----------------------+
| FK | reported_by           |
| FK | assigned_to | nullable|        +----------------------+
|    | detected_at           |   1:N  |  stock_transaction    |
|    | closed_at   | nullable|<-------+----------------------+
+----------------------------+        | FK | part_id          |
                                      | FK | work_order_id    | null=in/koreksi
                                      |    | tipe | in/out    |
                                      |    | qty              |
                                      +----------------------+
```

**Catatan kunci:** `trigger_type` = pembeda asal WO (BR-039). Maks 1 WO ≠ closed per equipment (BR-040).
