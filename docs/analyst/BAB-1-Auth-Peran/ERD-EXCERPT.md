# ERD Excerpt — BAB-1 Auth & Peran

> Excerpt dari `00-Global/ERD-MASTER.md` — sumber kebenaran ada di master. Read-only view; jangan edit manual.

```
+------------------+
|      users       |
+------------------+
| PK | id          |
|    | name        |
|    | email       | unique
|    | password    |
|    | role        | admin/foreman/operator/storekeeper
|    | act_as      | nullable (fitur Admin)
|    | created_at  |
+------------------+
```

**Relasi keluar excerpt:**
- `users.id` → FK di `p2h_checklist.user_id`, `shift_log.user_id`, `work_order.reported_by/assigned_to`, `stock_transaction.user_id` (→ lihat `00-Global/ERD-MASTER.md` / BAB-3, BAB-4, BAB-5)
