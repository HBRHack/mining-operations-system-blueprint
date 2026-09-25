# ERD Excerpt — BAB-3 Monitoring Harian

> Excerpt dari `00-Global/ERD-MASTER.md` — sumber kebenaran ada di master. Read-only view; jangan edit manual.

```
+------------------+       +------------------+
|    equipment     |       |      shift       |
+------------------+       +------------------+
| PK | id          |       | PK | id          |
|    | status      |       |    | kode        | day/night
|    | hour_meter  |       |    | jam_mulai   |
+---+----+---------+       |    | jam_selesai |
        |                  +--------+---------+
        | 1:N                       | 1:N
        |                  +--------v---------+
+-------v---------+        |    shift_log     |
| p2h_checklist   |        +------------------+
+------------------+        | PK | id          |
| PK | id          |        | FK | equipment_id|
| FK | equipment_id|        | FK | shift_id    |
| FK | user_id     |        | FK | user_id     |
|    | tanggal     |unique  |    | jam_mulai   |
|    | hasil       |pass/   |    | jam_selesai |
|                  | fail   |    | material    |
+--------+---------+        |    | tonase      |
         | 1:N              |    | alasan_idle |
+--------v---------+        |    | jam_operasi |
|p2h_checklist_item|        |    | hour_meter_akhir
+------------------+        +------------------+
| PK | id          |
| FK | p2h_checklist_id
|    | nama_item   | snapshot
|    | safety_critical
|    | status_item | ok/fail
+------------------+
```

**Relasi keluar excerpt:** `p2h_checklist.user_id` / `shift_log.user_id` → FK to `users` (BAB-1). Gagal safety-critical memicu `work_order` (BAB-4 — trigger_type=p2h_gagal).
