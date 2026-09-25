# Data Dict Excerpt — BAB-6 Dashboard

> Excerpt dari `00-Global/DATA-DICT-MASTER.md` — sumber kebenaran ada di master. BAB-6 read-only — tidak ada field tulis.

Field yang dibaca untuk agregasi:

| Entitas | Field | Dipakai untuk | BR |
|---------|-------|---------------|-----|
| equipment | status | Kartu jumlah alat per status (`COUNT GROUP BY status`) | BR-076 |
| work_order | trigger_type | KPI safety: `p2h_gagal` vs `breakdown_lapangan` | BR-079, BR-039 |
| work_order | detected_at, closed_at | Downtime = closed_at − detected_at | BR-047, BR-077 |
| work_order | status | Filter WO aktif | BR-037 |
| stock_transaction | tipe, work_order_id, part_id, qty | Part terpakai: tipe=out + WO ref → `GROUP BY part LIMIT 5` | BR-078 |
| part | stok, min_stok | Badge "Stok Menipis" (stok ≤ min_stok) | BR-060 |
| p2h_checklist_item | status_item | Item P2H paling sering gagal | BR-079 |

**Value List referensi (konsistensi Check #6a):**
- `equipment.status` ∈ {idle, running, rejected, breakdown, maintenance} — **bukan** `p2h_ok`
- `work_orders.trigger_type` ∈ {p2h_gagal, breakdown_lapangan} — pastikan query pakai string persis ini
- `work_orders.status` ∈ {open, menunggu_part, in_progress, closed}
- `stock_transaction.tipe` ∈ {in, out}
