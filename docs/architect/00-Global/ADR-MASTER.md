# ADR Master — Simulasi Operasional Tambang (Loading–Hauling)

Index seluruh Architecture Decision Record. Tiap keputusan teknis wajib punya satu baris di sini + satu file di `ADR/`. Gate **AG-01** (tiap ADR wajib punya blok `## References` berisi sumber riset) diverifikasi oleh script, bukan klaim.

> Riset & format: `architect-grill-with-docs` · Konteks stack: [`../TECH-STACK-MASTER.md`](../TECH-STACK-MASTER.md) · Kontrak keputusan awal: [`../../analyst/assumptions-constraints.md`](../../analyst/assumptions-constraints.md) (CON-001…CON-007)

## Indeks ADR

| ADR | Judul | Status | Date | Scope | Chapter Terdampak | Related Requirements |
|-----|-------|--------|------|-------|-------------------|----------------------|
| [ADR-001](ADR/ADR-001-php-laravel-blade-stack.md) | PHP 8.3 + Laravel 12 + Blade (Server-Rendered) sebagai Stack Inti | Accepted | 25 Sep 2026 | Global | Global (BAB-1…BAB-6) | CON-001…CON-004, NFR-PORT-001, NFR-PERF-001, NFR-SEC-001 |

## Antrian (belum dijadikan ADR)

Keputusan berikut sudah terwacana di TECH-STACK-MASTER tetapi belum punya ADR sendiri — diangkat saat nilainya cukup besar:

| Topik | Sementara di | Trigger menjadi ADR |
|-------|--------------|---------------------|
| Session store file-based (tanpa Redis/queue) | TECH-STACK-MASTER §TL;DR | Saat ada kebutuhan cache/queue nyata |
| Deployment LAN kantor site (tanpa cloud) | TECH-STACK-MASTER §Konteks Tambang | Saat tahap implementasi menentukan rincian deploy |
| Backup `mysqldump` via scheduler lokal | ADR-001 (wacana awal) | Saat tahap implementasi backup |

## Gate

| Gate | Aturan | Verifikasi |
|------|--------|------------|
| AG-01 | Setiap file `ADR/ADR-*.md` wajib punya section `## References` berisi ≥1 URL sumber riset | Script grep — 0 toleransi |
| AG-04 | Setiap ADR yang disebut di dokumen manapun harus punya file sendiri (tidak ada ADR yatim) | Script cross-check nama file |
